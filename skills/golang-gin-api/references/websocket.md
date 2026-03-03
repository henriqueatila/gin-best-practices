# WebSocket Reference

This file covers WebSocket integration with Gin using `gorilla/websocket`. Gin handles the HTTP upgrade handshake through its normal routing and middleware chain; after upgrade, all communication is raw WebSocket — Gin is no longer involved. Topics: upgrader setup, echo handler, hub pattern for broadcasting, authentication before upgrade, ping/pong keepalive, graceful shutdown, JSON messages, and testing.

## Table of Contents

1. [Upgrader Setup](#upgrader-setup)
2. [Basic Echo Handler](#basic-echo-handler)
3. [Hub Pattern](#hub-pattern)
4. [Client Struct with readPump / writePump](#client-struct-with-readpump--writepump)
5. [Auth Before Upgrade](#auth-before-upgrade)
6. [Ping/Pong Keepalive](#pingpong-keepalive)
7. [Graceful Shutdown](#graceful-shutdown)
8. [JSON Messages](#json-messages)
9. [Testing](#testing)

---

## Upgrader Setup

`websocket.Upgrader` converts an HTTP connection to a WebSocket connection. Configure it once at the package level (or inject it) — it is safe to use concurrently.

```go
// internal/ws/upgrader.go
package ws

import (
    "net/http"

    "github.com/gorilla/websocket"
)

// upgrader is the shared upgrader for all WebSocket endpoints.
// ReadBufferSize / WriteBufferSize tune the internal I/O buffers — not message
// size limits. 1024 bytes is a safe default for typical JSON payloads.
var upgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,

    // CheckOrigin gates which origins may open a WebSocket connection.
    // Never return true unconditionally in production — that allows
    // cross-site WebSocket hijacking (CSWSH).
    CheckOrigin: func(r *http.Request) bool {
        origin := r.Header.Get("Origin")
        // Allow requests with no origin header (e.g., native clients, tests).
        if origin == "" {
            return true
        }
        allowed := map[string]bool{
            "https://example.com":     true,
            "https://app.example.com": true,
        }
        return allowed[origin]
    },
}
```

**Why `CheckOrigin` matters:** Browsers automatically send the `Origin` header on WebSocket requests. Without validation, any website can open a WebSocket to your server using the visitor's credentials (cookies/sessions).

**`SetReadLimit`** — set a per-connection message size limit to prevent memory exhaustion from malicious clients:

```go
// After upgrading, before the read loop:
conn.SetReadLimit(512 * 1024) // 512 KB max per message
```

---

## Basic Echo Handler

The simplest handler: upgrade the connection, read messages in a loop, write each one back. This pattern demonstrates the full lifecycle.

```go
// internal/ws/echo.go
package ws

import (
    "errors"
    "log/slog"
    "net/http"

    "github.com/gin-gonic/gin"
    "github.com/gorilla/websocket"
)

// EchoHandler upgrades the HTTP connection and echoes every message back.
func EchoHandler(c *gin.Context) {
    conn, err := upgrader.Upgrade(c.Writer, c.Request, nil)
    if err != nil {
        // Upgrade already wrote a 400 response on failure; just log.
        slog.Error("websocket upgrade failed", "err", err)
        return
    }
    defer conn.Close()

    conn.SetReadLimit(512 * 1024)

    for {
        msgType, msg, err := conn.ReadMessage()
        if err != nil {
            if websocket.IsUnexpectedCloseError(err,
                websocket.CloseGoingAway,
                websocket.CloseNormalClosure,
            ) {
                slog.Warn("websocket read error", "err", err)
            }
            return // client disconnected or error — exit loop
        }

        if err := conn.WriteMessage(msgType, msg); err != nil {
            slog.Warn("websocket write error", "err", err)
            return
        }
    }
}
```

Register the handler like any Gin route — middleware runs during the HTTP upgrade request:

```go
r := gin.New()
r.Use(middleware.Logger(), middleware.Recovery())

r.GET("/ws/echo", ws.EchoHandler)
```

**Why `IsUnexpectedCloseError`:** When a client disconnects cleanly (browser tab closed), gorilla/websocket returns `CloseGoingAway`. Logging that as an error is noise. `IsUnexpectedCloseError` filters out expected close codes so you only log genuine errors.

---

## Hub Pattern

For broadcasting to multiple clients (chat rooms, live feeds), a central hub serializes all register/unregister/broadcast operations through a single goroutine — avoiding mutex-protected maps.

```go
// internal/ws/hub.go
package ws

// Hub maintains active client connections and routes messages.
type Hub struct {
    clients    map[*Client]bool
    broadcast  chan []byte
    register   chan *Client
    unregister chan *Client
}

func NewHub() *Hub {
    return &Hub{
        clients:    make(map[*Client]bool),
        broadcast:  make(chan []byte, 256),
        register:   make(chan *Client),
        unregister: make(chan *Client),
    }
}

// Run processes hub events. Call this in a dedicated goroutine.
func (h *Hub) Run() {
    for {
        select {
        case client := <-h.register:
            h.clients[client] = true

        case client := <-h.unregister:
            if _, ok := h.clients[client]; ok {
                delete(h.clients, client)
                close(client.send)
            }

        case message := <-h.broadcast:
            for client := range h.clients {
                select {
                case client.send <- message:
                default:
                    // send buffer full — client is too slow; drop and disconnect.
                    close(client.send)
                    delete(h.clients, client)
                }
            }
        }
    }
}
```

**Why channel-based (not mutex):** The hub goroutine owns the `clients` map exclusively. No locking needed. The `default` branch in broadcast prevents a slow client from blocking the entire broadcast loop.

Wire the hub into your Gin server:

```go
// cmd/server/main.go
hub := ws.NewHub()
go hub.Run()

r.GET("/ws", ws.ChatHandler(hub))
```

---

## Client Struct with readPump / writePump

Each connected client gets two goroutines: `readPump` reads from the WebSocket, `writePump` writes to it. Separating read and write avoids concurrent writes to `*websocket.Conn` (which gorilla/websocket does not allow).

```go
// internal/ws/client.go
package ws

import (
    "log/slog"
    "time"

    "github.com/gin-gonic/gin"
    "github.com/gorilla/websocket"
)

const (
    writeWait      = 10 * time.Second
    pongWait       = 60 * time.Second
    pingPeriod     = (pongWait * 9) / 10 // must be less than pongWait
    maxMessageSize = 512 * 1024           // 512 KB
)

// Client represents a single WebSocket connection.
type Client struct {
    hub  *Hub
    conn *websocket.Conn
    send chan []byte // buffered channel of outbound messages
}

// readPump reads messages from the WebSocket and forwards them to the hub.
// One goroutine per connection.
func (cl *Client) readPump() {
    defer func() {
        cl.hub.unregister <- cl
        cl.conn.Close()
    }()

    cl.conn.SetReadLimit(maxMessageSize)
    cl.conn.SetReadDeadline(time.Now().Add(pongWait))
    cl.conn.SetPongHandler(func(string) error {
        cl.conn.SetReadDeadline(time.Now().Add(pongWait))
        return nil
    })

    for {
        _, msg, err := cl.conn.ReadMessage()
        if err != nil {
            if websocket.IsUnexpectedCloseError(err,
                websocket.CloseGoingAway,
                websocket.CloseNormalClosure,
            ) {
                slog.Warn("ws read error", "err", err)
            }
            return
        }
        cl.hub.broadcast <- msg
    }
}

// writePump writes messages from the send channel to the WebSocket.
// One goroutine per connection — gorilla/websocket requires that writes
// happen from a single goroutine.
func (cl *Client) writePump() {
    ticker := time.NewTicker(pingPeriod)
    defer func() {
        ticker.Stop()
        cl.conn.Close()
    }()

    for {
        select {
        case msg, ok := <-cl.send:
            cl.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if !ok {
                // Hub closed the channel — send a close frame.
                cl.conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }
            if err := cl.conn.WriteMessage(websocket.TextMessage, msg); err != nil {
                slog.Warn("ws write error", "err", err)
                return
            }

        case <-ticker.C:
            cl.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if err := cl.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}

// ChatHandler upgrades the connection and registers the client with the hub.
func ChatHandler(hub *Hub) gin.HandlerFunc {
    return func(c *gin.Context) {
        conn, err := upgrader.Upgrade(c.Writer, c.Request, nil)
        if err != nil {
            slog.Error("ws upgrade failed", "err", err)
            return
        }

        // c.Copy() is required when passing *gin.Context to goroutines.
        // The original context is recycled by Gin after the handler returns.
        // However, after upgrade we don't use the Gin context anymore —
        // only the raw *websocket.Conn. Still, capture request-scoped values
        // (e.g., user ID set by auth middleware) before the handler returns.
        cpy := c.Copy()
        _ = cpy // use cpy.GetString("userID") etc. if needed

        client := &Client{
            hub:  hub,
            conn: conn,
            send: make(chan []byte, 256),
        }
        hub.register <- client

        go client.writePump()
        go client.readPump()
        // Handler returns immediately. readPump/writePump own the connection.
    }
}
```

**Why `c.Copy()` before goroutines:** Gin recycles `*gin.Context` objects via `sync.Pool` after the handler returns. If a goroutine holds the original context, it may read corrupted data from a different request. `c.Copy()` creates a snapshot safe to use after the handler returns.

---

## Auth Before Upgrade

WebSocket has no standard mechanism for sending auth headers after the handshake. Authenticate during the HTTP upgrade request — before calling `upgrader.Upgrade()`. If auth fails, return a normal HTTP error response.

```go
// internal/ws/auth_handler.go
package ws

import (
    "log/slog"
    "net/http"

    "github.com/gin-gonic/gin"
    "github.com/gorilla/websocket"
)

// AuthChatHandler validates a JWT from the query param before upgrading.
// Browsers cannot set custom headers on WebSocket connections — query params
// are the standard workaround. Keep tokens short-lived (< 60 s) when using
// them in URLs to reduce exposure in server logs.
func AuthChatHandler(hub *Hub, validateToken func(string) (string, error)) gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.Query("token")
        if token == "" {
            c.JSON(http.StatusUnauthorized, gin.H{"error": "missing token"})
            return
        }

        userID, err := validateToken(token)
        if err != nil {
            slog.Warn("ws auth failed", "err", err)
            c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid token"})
            return
        }

        conn, err := upgrader.Upgrade(c.Writer, c.Request, nil)
        if err != nil {
            slog.Error("ws upgrade failed", "err", err)
            return
        }

        // Auth middleware may also set userID via c.Set — capture it here
        // before the handler returns and Gin recycles the context.
        _ = userID // attach to Client struct as needed

        client := &Client{
            hub:  hub,
            conn: conn,
            send: make(chan []byte, 256),
        }
        hub.register <- client

        go client.writePump()
        go client.readPump()
    }
}
```

**Route registration — auth middleware still runs:**

```go
wsGroup := r.Group("/ws")
wsGroup.Use(middleware.RateLimit()) // rate limiting applies to upgrade request
{
    // Token in query param — no JWT middleware needed on this route
    wsGroup.GET("/chat", ws.AuthChatHandler(hub, tokenSvc.ValidateToken))
}
```

**Alternative — `Sec-WebSocket-Protocol` header:** Some clients send the token as a subprotocol name. This avoids URL exposure but requires echoing the subprotocol in the upgrade response header, which is more complex and less portable. Query param is simpler for most use cases.

---

## Ping/Pong Keepalive

Without keepalive, idle connections time out silently at the load balancer or OS level. gorilla/websocket supports the WebSocket ping/pong control frames natively.

The `writePump` ticker above already sends pings. The `readPump` deadline is reset on every pong. Here is the pattern isolated for clarity:

```go
// In readPump — reset read deadline when a pong arrives
conn.SetReadDeadline(time.Now().Add(pongWait)) // initial deadline
conn.SetPongHandler(func(appData string) error {
    // Extend deadline: client is alive
    conn.SetReadDeadline(time.Now().Add(pongWait))
    return nil
})

// In writePump ticker case — send a ping
conn.SetWriteDeadline(time.Now().Add(writeWait))
if err := conn.WriteMessage(websocket.PingMessage, nil); err != nil {
    return // connection broken; writePump exits, triggering cleanup
}
```

**Deadline chain:** writePump sends ping every `pingPeriod` (54 s). Client must reply with pong before `pongWait` (60 s) expires. If pong never arrives, `ReadMessage` returns a deadline error and `readPump` exits, which triggers `hub.unregister` and `conn.Close()`.

**Why `pingPeriod < pongWait`:** The ping must be sent before the read deadline fires. Using `(pongWait * 9) / 10` gives a 10% safety margin.

---

## Graceful Shutdown

On server shutdown, close all WebSocket connections cleanly so clients can reconnect rather than hang.

```go
// internal/ws/hub.go — add shutdown support
type Hub struct {
    clients    map[*Client]bool
    broadcast  chan []byte
    register   chan *Client
    unregister chan *Client
    shutdown   chan struct{}
}

func NewHub() *Hub {
    return &Hub{
        clients:    make(map[*Client]bool),
        broadcast:  make(chan []byte, 256),
        register:   make(chan *Client),
        unregister: make(chan *Client),
        shutdown:   make(chan struct{}),
    }
}

func (h *Hub) Run() {
    for {
        select {
        case <-h.shutdown:
            for client := range h.clients {
                // Send a close frame so the client knows the server is shutting down.
                client.conn.WriteMessage(
                    websocket.CloseMessage,
                    websocket.FormatCloseMessage(websocket.CloseServiceRestart, "server shutdown"),
                )
                close(client.send)
                delete(h.clients, client)
            }
            return

        case client := <-h.register:
            h.clients[client] = true

        case client := <-h.unregister:
            if _, ok := h.clients[client]; ok {
                delete(h.clients, client)
                close(client.send)
            }

        case message := <-h.broadcast:
            for client := range h.clients {
                select {
                case client.send <- message:
                default:
                    close(client.send)
                    delete(h.clients, client)
                }
            }
        }
    }
}

// Shutdown signals the hub to close all connections and stop.
func (h *Hub) Shutdown() {
    close(h.shutdown)
}
```

Wire into server shutdown:

```go
// cmd/server/main.go
hub := ws.NewHub()
go hub.Run()

srv := &http.Server{Addr: ":8080", Handler: r}

quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit

hub.Shutdown() // close WebSocket connections first

ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
if err := srv.Shutdown(ctx); err != nil {
    slog.Error("server shutdown error", "err", err)
}
```

**Why close WebSocket before HTTP shutdown:** `srv.Shutdown` stops accepting new requests and waits for in-flight HTTP handlers to finish. WebSocket handlers return immediately after spawning goroutines — those goroutines run outside Gin's request lifecycle. Shutting the hub down first ensures client goroutines exit before the process terminates.

---

## JSON Messages

Use typed message envelopes with `ReadJSON`/`WriteJSON` to make the wire protocol explicit.

```go
// internal/ws/message.go
package ws

import (
    "log/slog"
    "time"

    "github.com/gorilla/websocket"
)

// MessageType identifies the kind of payload.
type MessageType string

const (
    MsgChat   MessageType = "chat"
    MsgSystem MessageType = "system"
    MsgError  MessageType = "error"
)

// Envelope is the wire format for all WebSocket messages.
type Envelope struct {
    Type    MessageType `json:"type"`
    Payload any `json:"payload"`
    SentAt  time.Time   `json:"sent_at"`
}

// ChatPayload is the payload for MsgChat messages.
type ChatPayload struct {
    UserID  string `json:"user_id"`
    Content string `json:"content"`
}

// readJSON reads one message from the connection into dst.
func readJSON(conn *websocket.Conn, dst any) error {
    conn.SetReadDeadline(time.Now().Add(pongWait))
    return conn.ReadJSON(dst)
}

// writeJSON writes src to the connection as JSON.
func writeJSON(conn *websocket.Conn, src any) error {
    conn.SetWriteDeadline(time.Now().Add(writeWait))
    return conn.WriteJSON(src)
}

// Example: handler that processes typed messages
func handleTypedMessage(conn *websocket.Conn) {
    for {
        var env Envelope
        if err := readJSON(conn, &env); err != nil {
            if websocket.IsUnexpectedCloseError(err,
                websocket.CloseGoingAway,
                websocket.CloseNormalClosure,
            ) {
                slog.Warn("ws read error", "err", err)
            }
            return
        }

        switch env.Type {
        case MsgChat:
            // Re-encode payload to typed struct for validation
            // (ReadJSON into any gives map[string]any)
            slog.Info("chat message received", "payload", env.Payload)
        default:
            resp := Envelope{
                Type:    MsgError,
                Payload: "unknown message type",
                SentAt:  time.Now(),
            }
            if err := writeJSON(conn, resp); err != nil {
                slog.Warn("ws write error", "err", err)
                return
            }
        }
    }
}
```

**Why `any` in `Envelope.Payload`:** The envelope type is fixed, but the payload varies per message type. Unmarshal into `Envelope` first, then use `json.Unmarshal` on the re-encoded payload for strict typing per type:

```go
import "encoding/json"

raw, err := json.Marshal(env.Payload)
if err != nil {
    return
}
var chat ChatPayload
if err := json.Unmarshal(raw, &chat); err != nil {
    slog.Warn("invalid chat payload", "err", err)
    return
}
```

---

## Testing

WebSocket tests require a real TCP server — `httptest.NewServer` + `websocket.DefaultDialer.Dial`. You cannot use `httptest.NewRecorder` for WebSocket because the upgrade requires a hijackable connection.

```go
// internal/ws/echo_test.go
package ws_test

import (
    "net/http"
    "net/http/httptest"
    "strings"
    "testing"
    "time"

    "github.com/gin-gonic/gin"
    "github.com/gorilla/websocket"

    ws "github.com/yourorg/yourapp/internal/ws"
)

func newTestServer(t *testing.T) *httptest.Server {
    t.Helper()
    gin.SetMode(gin.TestMode)
    r := gin.New()
    r.GET("/ws/echo", ws.EchoHandler)
    return httptest.NewServer(r)
}

func wsURL(srv *httptest.Server, path string) string {
    return "ws" + strings.TrimPrefix(srv.URL, "http") + path
}

func TestEchoHandler_RoundTrip(t *testing.T) {
    srv := newTestServer(t)
    defer srv.Close()

    dialer := websocket.Dialer{HandshakeTimeout: 3 * time.Second}
    conn, resp, err := dialer.Dial(wsURL(srv, "/ws/echo"), nil)
    if err != nil {
        t.Fatalf("dial failed: %v (status %d)", err, resp.StatusCode)
    }
    defer conn.Close()

    want := "hello"
    if err := conn.WriteMessage(websocket.TextMessage, []byte(want)); err != nil {
        t.Fatalf("write failed: %v", err)
    }

    conn.SetReadDeadline(time.Now().Add(3 * time.Second))
    _, got, err := conn.ReadMessage()
    if err != nil {
        t.Fatalf("read failed: %v", err)
    }

    if string(got) != want {
        t.Errorf("got %q, want %q", got, want)
    }
}

func TestEchoHandler_ClientClose(t *testing.T) {
    srv := newTestServer(t)
    defer srv.Close()

    conn, _, err := websocket.DefaultDialer.Dial(wsURL(srv, "/ws/echo"), nil)
    if err != nil {
        t.Fatalf("dial failed: %v", err)
    }

    // Send a normal close frame; server should exit its read loop cleanly.
    err = conn.WriteMessage(
        websocket.CloseMessage,
        websocket.FormatCloseMessage(websocket.CloseNormalClosure, "done"),
    )
    if err != nil {
        t.Fatalf("close write failed: %v", err)
    }

    conn.SetReadDeadline(time.Now().Add(2 * time.Second))
    _, _, err = conn.ReadMessage()
    if err == nil {
        t.Error("expected error after close, got nil")
    }
    // A CloseError or io.EOF is expected — both indicate server closed the connection.
    if !websocket.IsCloseError(err, websocket.CloseNormalClosure) {
        if err.Error() != "EOF" && !strings.Contains(err.Error(), "use of closed") {
            t.Logf("close read returned: %v (acceptable)", err)
        }
    }
}

func TestChatHandler_Broadcast(t *testing.T) {
    gin.SetMode(gin.TestMode)
    hub := ws.NewHub()
    go hub.Run()
    defer hub.Shutdown()

    r := gin.New()
    r.GET("/ws/chat", ws.ChatHandler(hub))
    srv := httptest.NewServer(r)
    defer srv.Close()

    dial := func() *websocket.Conn {
        conn, _, err := websocket.DefaultDialer.Dial(wsURL(srv, "/ws/chat"), nil)
        if err != nil {
            t.Fatalf("dial failed: %v", err)
        }
        return conn
    }

    c1 := dial()
    c2 := dial()
    defer c1.Close()
    defer c2.Close()

    // Small sleep to let register events process
    time.Sleep(50 * time.Millisecond)

    want := "broadcast-test"
    if err := c1.WriteMessage(websocket.TextMessage, []byte(want)); err != nil {
        t.Fatalf("c1 write failed: %v", err)
    }

    // Both c1 and c2 should receive the broadcast
    for _, conn := range []*websocket.Conn{c1, c2} {
        conn.SetReadDeadline(time.Now().Add(2 * time.Second))
        _, msg, err := conn.ReadMessage()
        if err != nil {
            t.Fatalf("read failed: %v", err)
        }
        if string(msg) != want {
            t.Errorf("got %q, want %q", msg, want)
        }
    }
}
```

**Key testing patterns:**
- Use `httptest.NewServer` (not `httptest.NewRecorder`) — WebSocket requires a hijackable TCP connection.
- Convert `http://` to `ws://` with `strings.TrimPrefix`.
- Set `SetReadDeadline` in tests to prevent hangs on failure.
- Test both happy path and clean disconnect to verify server-side cleanup.
- `hub.Shutdown()` in defer ensures test goroutines exit cleanly.
