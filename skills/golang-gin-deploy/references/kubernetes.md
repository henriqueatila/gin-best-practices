# Kubernetes Reference

This file covers Kubernetes deployment for Go Gin APIs: Deployment manifest, Service, ConfigMap, Secret, liveness/readiness probes, Horizontal Pod Autoscaler, Ingress, PVC for PostgreSQL, complete manifests, brief Helm chart structure, and a GitHub Actions CI/CD workflow. Use when deploying a Gin API to a Kubernetes cluster.

> **Architectural recommendation:** Kubernetes patterns are not part of the Gin framework. These are mainstream cloud-native Go community patterns.

## Table of Contents

1. [Deployment Manifest](#deployment-manifest)
2. [Service Manifest](#service-manifest)
3. [ConfigMap and Secret](#configmap-and-secret)
4. [Liveness and Readiness Probes](#liveness-and-readiness-probes)
5. [Horizontal Pod Autoscaler](#horizontal-pod-autoscaler)
6. [Ingress](#ingress)
7. [PVC for PostgreSQL](#pvc-for-postgresql)
8. [PodDisruptionBudget](#poddisruptionbudget)
9. [NetworkPolicy](#networkpolicy)
10. [Complete Manifests](#complete-manifests)
11. [Helm Chart Structure](#helm-chart-structure)
12. [GitHub Actions CI/CD Workflow](#github-actions-cicd-workflow)

---

## Deployment Manifest

The Deployment declares the desired state: which image to run, how many replicas, resource limits, and probe configuration.

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp
  labels:
    app: myapp
    version: "1.0.0"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0    # never reduce below desired replicas during update
      maxSurge: 1          # allow one extra pod during rollout
  template:
    metadata:
      labels:
        app: myapp
    spec:
      # Graceful termination: allow 30s for in-flight requests to complete
      terminationGracePeriodSeconds: 30

      # Prevent all pods landing on same node
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: myapp
                topologyKey: kubernetes.io/hostname

      containers:
        - name: myapp
          image: ghcr.io/myorg/myapp:latest
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP

          # Load non-sensitive config from ConfigMap
          envFrom:
            - configMapRef:
                name: myapp-config

          # Load sensitive values from Secret
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: myapp-secret
                  key: database-url
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: myapp-secret
                  key: jwt-secret
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: myapp-secret
                  key: redis-url

          # Resource requests/limits — tune for your workload
          resources:
            requests:
              cpu: 100m       # 0.1 CPU core
              memory: 64Mi
            limits:
              cpu: 500m       # 0.5 CPU core
              memory: 256Mi

          # Probes — see dedicated section below
          livenessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 15
            periodSeconds: 20
            timeoutSeconds: 5
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3

          # Security context — run as non-root
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 65532      # matches distroless:nonroot UID
            capabilities:
              drop: ["ALL"]

      # Pod-level security context
      securityContext:
        seccompProfile:
          type: RuntimeDefault
```

---

## Service Manifest

The Service exposes the Deployment as a stable network endpoint inside the cluster.

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: myapp
  labels:
    app: myapp
spec:
  type: ClusterIP        # internal only — Ingress handles external traffic
  selector:
    app: myapp           # routes to pods with this label
  ports:
    - name: http
      port: 80           # service port (cluster-internal)
      targetPort: http   # named port from Deployment container
      protocol: TCP
```

**Service types:**

| Type | Use case |
|------|----------|
| `ClusterIP` | Default — internal access only (use with Ingress) |
| `NodePort` | Expose on each node's IP — dev/testing only |
| `LoadBalancer` | Cloud provider creates an external load balancer |

---

## ConfigMap and Secret

**ConfigMap — non-sensitive configuration:**

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: myapp
data:
  PORT: "8080"
  GIN_MODE: "release"
  MIGRATIONS_PATH: "db/migrations"
  READ_TIMEOUT: "10s"
  WRITE_TIMEOUT: "10s"
  SHUTDOWN_TIMEOUT: "30s"
```

**Secret — sensitive values:**

```yaml
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secret
  namespace: myapp
type: Opaque
# Values must be base64-encoded
# echo -n 'postgres://user:pass@host:5432/db?sslmode=require' | base64
data:
  database-url: cG9zdGdyZXM6Ly91c2VyOnBhc3NAaG9zdDo1NDMyL2RiP3NzbG1vZGU9cmVxdWlyZQ==
  jwt-secret: eW91ci1zZWNyZXQtaGVyZQ==
  redis-url: cmVkaXM6Ly9yZWRpczo2Mzc5
```

**Critical:** Never commit real Secret values to git. Use a secrets manager (Vault, AWS Secrets Manager, GCP Secret Manager) and inject at deploy time via CI/CD, or use sealed-secrets / External Secrets Operator.

**Generate base64 values:**

```bash
echo -n 'your-value' | base64
# Decode to verify:
echo 'eW91ci12YWx1ZQ==' | base64 --decode
```

**Create secrets from CLI without manifest (avoids base64 in files):**

```bash
kubectl create secret generic myapp-secret \
  --from-literal=database-url="postgres://user:pass@host:5432/db?sslmode=require" \
  --from-literal=jwt-secret="your-jwt-secret" \
  --from-literal=redis-url="redis://redis:6379" \
  --namespace=myapp
```

---

## Liveness and Readiness Probes

Both probes hit the `/health` endpoint from the **golang-gin-api** skill's `HealthHandler`.

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: http
  initialDelaySeconds: 15   # wait 15s before first check (app startup time)
  periodSeconds: 20          # check every 20s
  timeoutSeconds: 5          # fail if no response in 5s
  failureThreshold: 3        # restart after 3 consecutive failures
  successThreshold: 1

readinessProbe:
  httpGet:
    path: /health
    port: http
  initialDelaySeconds: 5    # readiness can start sooner than liveness
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 3        # remove from load balancer after 3 failures
  successThreshold: 1
```

**Probe behavior:**

| Probe | On failure | On success |
|-------|-----------|------------|
| `livenessProbe` | Restart the container | Nothing |
| `readinessProbe` | Remove from Service endpoints (stop receiving traffic) | Add to Service endpoints |

**When to split into separate endpoints:**

```go
// /health/live — process health only (no external deps)
r.GET("/health/live", func(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{"status": "ok"})
})

// /health/ready — can the app serve traffic? (checks DB)
r.GET("/health/ready", healthHandler.Check)
```

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: http
  initialDelaySeconds: 5

readinessProbe:
  httpGet:
    path: /health/ready
    port: http
  initialDelaySeconds: 10
```

Use separate endpoints when: app startup takes longer than DB connection (avoids liveness killing a healthy-but-slow-starting pod).

**Startup probe (for slow-starting apps):**

```yaml
startupProbe:
  httpGet:
    path: /health/live
    port: http
  failureThreshold: 30  # 30 * 10s = 5 minutes max startup time
  periodSeconds: 10
```

Startup probe runs first. Only after it succeeds do liveness and readiness probes activate. Prevents liveness from killing a legitimately slow-starting pod.

---

## Horizontal Pod Autoscaler

Scale replicas based on CPU or memory utilization.

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp
  namespace: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70   # scale up when avg CPU > 70%
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # wait 5 min before scaling down
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60            # remove at most 1 pod per minute
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60            # add at most 2 pods per minute
```

**Critical:** `resources.requests` must be set in the Deployment for HPA to calculate utilization percentages. HPA uses `actual / request` to compute utilization.

---

## Ingress

Ingress routes external HTTP/HTTPS traffic to the Service. Requires an Ingress controller (nginx-ingress, Traefik, etc.) installed in the cluster.

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: myapp
  annotations:
    # nginx-ingress specific — adjust for your controller
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "30"
    # TLS cert managed by cert-manager
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: myapp-tls   # cert-manager creates this Secret
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  name: http
```

---

## PVC for PostgreSQL

In production, use a managed database (RDS, Cloud SQL, Neon). For self-hosted PostgreSQL in Kubernetes, use a PersistentVolumeClaim.

```yaml
# k8s/postgres.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: myapp
spec:
  accessModes:
    - ReadWriteOnce      # single node access — suitable for stateful DB
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard   # use your cluster's StorageClass

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: myapp
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:17-alpine
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: db-name
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: db-user
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: db-password
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              memory: 256Mi
              cpu: 100m
            limits:
              memory: 512Mi
              cpu: 500m
  volumeClaimTemplates:
    - metadata:
        name: postgres-storage
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi

---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: myapp
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

---

## PodDisruptionBudget

Ensures at least one pod remains available during voluntary disruptions (node drains, cluster upgrades). Without a PDB, a rolling upgrade or node drain can take all replicas offline simultaneously.

```yaml
# k8s/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: myapp
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: myapp
```

**When to use `minAvailable` vs `maxUnavailable`:**

| Setting | Behavior |
|---------|----------|
| `minAvailable: 1` | At least 1 pod stays up — safe for 2+ replicas |
| `maxUnavailable: 1` | At most 1 pod goes down at a time |

For 2 replicas, `minAvailable: 1` and `maxUnavailable: 1` are equivalent. With 3+ replicas, `minAvailable: 2` is more conservative.

**Critical:** PDB only protects against *voluntary* disruptions (kubectl drain, cluster upgrades). It does not prevent crashes from OOMKill or node failures.

---

## NetworkPolicy

Restricts which pods can send traffic to the application. Without a NetworkPolicy, any pod in the cluster can reach the app on port 8080.

```yaml
# k8s/netpol.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: myapp-netpol
  namespace: myapp
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
      ports:
        - port: 8080
```

This policy allows inbound traffic only from the `ingress-nginx` namespace (the Ingress controller). All other ingress is denied by default once any NetworkPolicy selects the pod.

**Label the ingress-nginx namespace if not already done:**

```bash
kubectl label namespace ingress-nginx name=ingress-nginx
```

**Extend to allow inter-service communication:**

```yaml
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
      ports:
        - port: 8080
    - from:
        - podSelector:
            matchLabels:
              app: internal-service   # allow calls from a specific internal service
      ports:
        - port: 8080
```

**Note:** NetworkPolicy requires a CNI plugin that supports it (Calico, Cilium, Weave). Default GKE, EKS, and AKS clusters support NetworkPolicy when enabled at cluster creation.

---

## Complete Manifests

Apply all manifests at once:

```bash
# Create namespace
kubectl create namespace myapp

# Apply all manifests
kubectl apply -f k8s/ --namespace=myapp

# Verify rollout
kubectl rollout status deployment/myapp --namespace=myapp

# Check pods
kubectl get pods --namespace=myapp

# View logs
kubectl logs -l app=myapp --namespace=myapp --tail=100 -f

# Describe a pod (events, probes, status)
kubectl describe pod -l app=myapp --namespace=myapp
```

**Recommended k8s directory layout:**

```
k8s/
├── namespace.yaml
├── configmap.yaml
├── secret.yaml          # gitignored — created via CI/CD
├── deployment.yaml
├── service.yaml
├── ingress.yaml
├── hpa.yaml
├── pdb.yaml             # PodDisruptionBudget
├── netpol.yaml          # NetworkPolicy
├── migrate-job.yaml     # run before deploying new version
└── postgres/            # only if self-hosting postgres
    ├── statefulset.yaml
    ├── service.yaml
    └── secret.yaml
```

**Migration Job (run before deploying new app version):**

```yaml
# k8s/migrate-job.yaml
# NOTE: For Helm deployments, replace the static name with:
#   name: db-migrate-{{ .Values.image.tag }}
# For raw kubectl apply, use a timestamp suffix or generateName instead:
#   generateName: db-migrate-
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate-v1-0-0   # replace with your version tag (e.g. db-migrate-v1-2-3)
  namespace: myapp
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: migrate/migrate:v4
          args:
            - -path=/migrations
            - -database=$(DATABASE_URL)
            - up
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: myapp-secret
                  key: database-url
          volumeMounts:
            - name: migrations
              mountPath: /migrations
      volumes:
        - name: migrations
          configMap:
            name: db-migrations
```

See the **golang-gin-database** skill (`references/migrations.md`) for the full migration strategy and zero-downtime patterns.

---

## Helm Chart Structure

Helm templates manifests with values, enabling environment-specific configuration without duplicating YAML.

```
charts/myapp/
├── Chart.yaml           # chart metadata
├── values.yaml          # default values
├── values.prod.yaml     # production overrides
├── templates/
│   ├── _helpers.tpl     # named templates (labels, selectors)
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── hpa.yaml
│   └── migrate-job.yaml
└── .helmignore
```

`Chart.yaml`:

```yaml
apiVersion: v2
name: myapp
description: Gin REST API
type: application
version: 0.1.0
appVersion: "1.0.0"
```

`values.yaml` (excerpt):

```yaml
image:
  repository: ghcr.io/myorg/myapp
  tag: latest
  pullPolicy: Always

replicaCount: 2

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: nginx
  host: api.example.com
  tls: true

resources:
  requests:
    cpu: 100m
    memory: 64Mi
  limits:
    cpu: 500m
    memory: 256Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilization: 70

config:
  ginMode: release
  port: "8080"
  migrationsPath: db/migrations
```

Install / upgrade:

```bash
# Install
helm install myapp ./charts/myapp \
  --namespace=myapp \
  --create-namespace \
  --values=charts/myapp/values.prod.yaml \
  --set image.tag=v1.2.3

# Upgrade (rolling deploy)
helm upgrade myapp ./charts/myapp \
  --namespace=myapp \
  --values=charts/myapp/values.prod.yaml \
  --set image.tag=v1.2.4

# Rollback to previous release
helm rollback myapp 1 --namespace=myapp
```

---

## GitHub Actions CI/CD Workflow

Build image, push to registry, run migrations, and deploy to Kubernetes.

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: "1.24"
          cache: true

      - name: Run unit tests
        run: go test -v -race -cover ./...

  build:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}

    steps:
      - uses: actions/checkout@v4

      - name: Log in to container registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build and push image
        id: build
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          build-args: |
            VERSION=${{ github.sha }}
            BUILD_TIME=${{ github.event.head_commit.timestamp }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Scan image for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
          severity: CRITICAL,HIGH
          exit-code: 1

  migrate:
    needs: build
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Install migrate CLI
        run: |
          go install -tags 'postgres' \
            github.com/golang-migrate/migrate/v4/cmd/migrate@latest

      - name: Run migrations
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
        run: |
          migrate -path db/migrations -database "$DATABASE_URL" up

  deploy:
    needs: [build, migrate]
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v4

      - name: Configure kubeconfig
        run: |
          mkdir -p ~/.kube
          echo "${{ secrets.KUBECONFIG }}" | base64 --decode > ~/.kube/config
          chmod 600 ~/.kube/config

      - name: Deploy to Kubernetes
        env:
          IMAGE_TAG: sha-${{ github.sha }}
        run: |
          kubectl set image deployment/myapp \
            myapp=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${IMAGE_TAG} \
            --namespace=myapp

          kubectl rollout status deployment/myapp \
            --namespace=myapp \
            --timeout=5m

      - name: Verify deployment
        run: |
          kubectl get pods -l app=myapp --namespace=myapp
          kubectl get deployment myapp --namespace=myapp
```

**Required GitHub secrets:** `DATABASE_URL` (PostgreSQL connection string for migrations), `KUBECONFIG` (base64-encoded kubeconfig for cluster access).

**Deployment flow:** `push → test → build (image push) → migrate → deploy (kubectl set image) → verify (rollout status)`.
