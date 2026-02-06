# Infrastructure Data Model: Cloud Native Todo Chatbot Kubernetes Deployment

**Phase**: 1 (Design)
**Date**: 2026-02-03
**Feature**: 002-k8s-deployment
**Status**: Complete

---

## 1. Docker Image Model

### Entity: Docker Image (Frontend)

**Name**: `todo-frontend` or `todo-chatbot-frontend`

**Purpose**: Containerized Next.js/React application accessible via port 3000

**Attributes**:
- `name`: Image name (e.g., `todo-frontend`)
- `tag`: Version identifier (e.g., `latest`, `v1.0.0`, Git commit hash)
- `registry`: Registry location (localhost:5000, docker.io/username, ghcr.io/org)
- `baseImage`: `node:18-alpine` (minimal Node.js runtime)
- `size`: Target <500MB (optimized with multi-stage build)
- `healthCheckPath`: `/health` (GET request returning 200 OK)
- `port`: 3000 (standard React/Next.js port)

**Build Artifacts**:
- **Build Stage**: Full Node.js + npm + build tools
- **Runtime Stage**: Node.js runtime + built Next.js bundle (`.next/` directory)
- **Excluded**: Source code, development dependencies, npm cache

**Configuration (Environment Variables)**:
- `BACKEND_URL`: API endpoint for backend service (e.g., `http://backend:8000`)
- `NEXT_PUBLIC_API_URL`: Public API URL for browser requests
- `NODE_ENV`: Set to `production` in runtime

**Health Check**:
- **Type**: HTTP GET
- **Path**: `/health`
- **Expected Response**: 200 OK with body "OK" or minimal JSON
- **Timeout**: 3 seconds
- **Period**: 10 seconds
- **Failure Threshold**: 3 consecutive failures = pod restart

### Entity: Docker Image (Backend)

**Name**: `todo-backend` or `todo-chatbot-backend`

**Purpose**: Containerized FastAPI application serving chat endpoint on port 8000

**Attributes**:
- `name`: Image name (e.g., `todo-backend`)
- `tag`: Version identifier (e.g., `latest`, `v1.0.0`)
- `registry`: Registry location
- `baseImage`: `python:3.11-slim` (minimal Python runtime)
- `size`: Target <300MB (optimized with multi-stage build)
- `healthCheckPath`: `/health` (verifies database connectivity)
- `port`: 8000 (FastAPI default)

**Build Artifacts**:
- **Build Stage**: Python 3.11 + pip + build tools (for C extension compilation if needed)
- **Runtime Stage**: Python runtime + installed pip packages from requirements.txt
- **Excluded**: Source code, pip cache, development dependencies

**Configuration (Environment Variables)**:
- `DATABASE_URL`: PostgreSQL connection string (e.g., `postgresql://user:pass@neon.tech/dbname`)
- `OPENAI_API_KEY`: API key for OpenAI Agents SDK
- `MCP_SERVER_ENDPOINT`: Endpoint for MCP server (if running separately)
- `ENV`: Set to `production`

**Health Check**:
- **Type**: HTTP GET
- **Path**: `/health`
- **Expected Response**: 200 OK with database connectivity status
- **Timeout**: 3 seconds
- **Period**: 10 seconds
- **Failure Threshold**: 3 consecutive failures = pod restart

### Relationships

```
Docker Image (Frontend) ──> Docker Registry
Docker Image (Backend) ──-> Docker Registry
      ↓
   (used by)
      ↓
Kubernetes Deployment
```

---

## 2. Kubernetes Pod Model

### Entity: Pod (Frontend)

**Purpose**: Running instance of frontend Docker image in Kubernetes cluster

**Attributes**:
- `name`: Pod name (e.g., `todo-chatbot-frontend-6f7d9c8b`)
- `namespace`: `default` (MVP scope)
- `image`: `todo-frontend:latest`
- `imagePolicy`: `IfNotPresent` (use local image if exists; pull only if missing)
- `port`: 3000
- `restartPolicy`: `Always` (restart pod if it crashes)

**Environment Variables** (from ConfigMap/Secret):
- Injected via `env` field in pod spec
- Frontend pod sees variables set in Deployment template

**Health Checks**:
- **Liveness Probe**: Restarts pod if health check fails
- **Readiness Probe**: Removes pod from service endpoint if failing (but doesn't restart)

**Resource Constraints**:
- **Requests** (guaranteed allocation):
  - CPU: 100m (0.1 core)
  - Memory: 256Mi
- **Limits** (maximum allowed):
  - CPU: 500m (0.5 core)
  - Memory: 512Mi

**Volume Mounts**: None in MVP (stateless application)

### Entity: Pod (Backend)

**Purpose**: Running instance of backend Docker image in Kubernetes cluster

**Attributes**:
- `name`: Pod name (e.g., `todo-chatbot-backend-4d2f1a9e`)
- `namespace`: `default`
- `image`: `todo-backend:latest`
- `imagePolicy`: `IfNotPresent`
- `port`: 8000
- `restartPolicy`: `Always`

**Environment Variables** (from ConfigMap/Secret):
- `DATABASE_URL`: Injected from Secret
- `OPENAI_API_KEY`: Injected from Secret
- `BACKEND_URL`: Set to localhost for internal operations

**Health Checks**:
- **Liveness Probe**: Verifies pod is alive; restarts if failing
- **Readiness Probe**: Checks database connectivity; removes from service if failing

**Resource Constraints**:
- **Requests**:
  - CPU: 200m
  - Memory: 512Mi
- **Limits**:
  - CPU: 1000m (1 core)
  - Memory: 1Gi

**Volume Mounts**: None in MVP (database external; no persistent storage)

---

## 3. Kubernetes Deployment Model

### Entity: Deployment (Frontend)

**Purpose**: Manages pods for frontend application with configured replicas, update strategy, and health checks

**Attributes**:
- `name`: `todo-chatbot-frontend`
- `namespace`: `default`
- `replicas`: 1 (MVP; configurable via values.yaml)
- `strategy`: `RollingUpdate` (Kubernetes default; no disruption during updates)
  - `maxSurge`: 1 (max 2 pods during update)
  - `maxUnavailable`: 0 (no pod should be unavailable)

**Selector Labels**:
- `app: todo-chatbot-frontend`
- `tier: frontend`

**Pod Template** (defines pods created by Deployment):
- Image: `{{ .Values.frontend.image.repository }}:{{ .Values.frontend.image.tag }}`
- Ports: 3000
- Environment variables from ConfigMap
- Resource requests/limits as per Pod specification above

**Update Behavior**:
1. Create new pod with new image
2. Wait for readiness probe to pass
3. Remove old pod
4. Repeat until all pods updated (rolling update)

### Entity: Deployment (Backend)

**Purpose**: Manages pods for backend application with stateless design

**Attributes**:
- `name`: `todo-chatbot-backend`
- `namespace`: `default`
- `replicas`: 1 (MVP; manually scaled via kubectl-ai or helm values)
- `strategy`: `RollingUpdate`
  - `maxSurge`: 1
  - `maxUnavailable`: 0

**Selector Labels**:
- `app: todo-chatbot-backend`
- `tier: backend`

**Pod Template**:
- Image: `{{ .Values.backend.image.repository }}:{{ .Values.backend.image.tag }}`
- Ports: 8000
- Environment variables from ConfigMap + Secret
- Resource constraints
- Database connection via Secret injection

---

## 4. Kubernetes Service Model

### Entity: Service (Frontend)

**Purpose**: Network abstraction exposing frontend pods to external traffic

**Attributes**:
- `name`: `todo-chatbot-frontend`
- `namespace`: `default`
- `type`: `LoadBalancer` (in Minikube, maps to localhost; in cloud, creates external IP)
- `port`: 3000 (port exposed on service)
- `targetPort`: 3000 (port on pod)
- `selector`: `app: todo-chatbot-frontend` (routes traffic to matching pods)

**Behavior**:
- Traffic to `<service-ip>:3000` → routed to pod:3000
- In Minikube: `kubectl port-forward svc/frontend 3000:3000` exposes service on localhost:3000
- Service is ephemeral; external IP changes on service recreation

### Entity: Service (Backend)

**Purpose**: Internal service for backend pods; stable DNS for inter-service communication

**Attributes**:
- `name`: `todo-chatbot-backend`
- `namespace`: `default`
- `type`: `ClusterIP` (internal only; no external exposure)
- `port`: 8000
- `targetPort`: 8000
- `selector`: `app: todo-chatbot-backend`

**DNS Behavior**:
- Stable DNS name: `todo-chatbot-backend` within cluster (short form)
- Full DNS: `todo-chatbot-backend.default.svc.cluster.local`
- Frontend pod can reach backend at `http://todo-chatbot-backend:8000`

**Load Balancing**:
- If 3 backend pods running, requests round-robin across all 3
- Failed pod automatically removed from service endpoints (via readiness probe)

---

## 5. Kubernetes ConfigMap Model

### Entity: ConfigMap (Application Configuration)

**Purpose**: Non-sensitive configuration data injected into pods as environment variables

**Attributes**:
- `name`: `todo-chatbot-config`
- `namespace`: `default`
- `data`: Key-value pairs for environment variables

**Configuration Keys**:
```yaml
data:
  BACKEND_URL: "http://todo-chatbot-backend:8000"
  DATABASE_HOST: "neon-staging.us-east-1.postgres.vercel-storage.com"
  DATABASE_PORT: "5432"
  DATABASE_NAME: "todo_chatbot"
  ENVIRONMENT: "development"
  LOG_LEVEL: "DEBUG"
```

**Pod Reference**:
```yaml
env:
- name: BACKEND_URL
  valueFrom:
    configMapKeyRef:
      name: todo-chatbot-config
      key: BACKEND_URL
```

**Immutability**: ConfigMap is mutable; changes don't automatically roll out to pods (pods must be restarted to pick up changes)

---

## 6. Kubernetes Secret Model

### Entity: Secret (Sensitive Credentials)

**Purpose**: Secure storage for sensitive data (database password, API keys)

**Attributes**:
- `name`: `todo-chatbot-secrets`
- `namespace`: `default`
- `type`: `Opaque` (generic base64-encoded secrets)
- `data`: Base64-encoded key-value pairs

**Secret Keys**:
```yaml
data:
  DATABASE_PASSWORD: <base64-encoded-password>
  OPENAI_API_KEY: <base64-encoded-api-key>
```

**Pod Reference**:
```yaml
env:
- name: DATABASE_PASSWORD
  valueFrom:
    secretKeyRef:
      name: todo-chatbot-secrets
      key: DATABASE_PASSWORD
```

**Security Notes**:
- Base64 encoding is NOT encryption (mvp acceptable for development)
- Production would enable Secret encryption at rest
- Never commit actual secrets to git; use Helm values injection at deployment time

**Creation Pattern**:
```bash
helm install todo-chatbot ./helm/todo-chatbot \
  --set secrets.databasePassword=<actual-password> \
  --set secrets.openaiApiKey=<actual-api-key>
```

---

## 7. Helm Chart Model

### Entity: Helm Chart (Release Configuration)

**Purpose**: Templated Kubernetes resources with environment-specific values

**Attributes**:
- `name`: `todo-chatbot`
- `version`: `1.0.0` (chart version, independent of application version)
- `appVersion`: `1.0.0` (application version)
- `description`: "Cloud Native Todo Chatbot Kubernetes Deployment"

**Chart Structure**:
```
helm/todo-chatbot/
├── Chart.yaml              # Chart metadata
├── values.yaml             # Default values (Minikube development)
├── templates/
│   ├── deployment-frontend.yaml
│   ├── deployment-backend.yaml
│   ├── service-frontend.yaml
│   ├── service-backend.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── _helpers.tpl        # Template helpers
│   └── NOTES.txt
└── README.md
```

### Entity: Helm Values (Configuration)

**Purpose**: Environment-specific values injected into Helm templates

**Default Values** (`values.yaml`):
```yaml
replicaCount: 1

frontend:
  name: frontend
  replicas: 1
  image:
    repository: localhost:5000/todo-frontend
    pullPolicy: IfNotPresent
    tag: latest
  service:
    type: LoadBalancer
    port: 3000
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi

backend:
  name: backend
  replicas: 1
  image:
    repository: localhost:5000/todo-backend
    pullPolicy: IfNotPresent
    tag: latest
  service:
    type: ClusterIP
    port: 8000
  resources:
    requests:
      cpu: 200m
      memory: 512Mi
    limits:
      cpu: 1000m
      memory: 1Gi

config:
  backendUrl: "http://todo-chatbot-backend:8000"
  databaseHost: "neon-staging.us-east-1.postgres.vercel-storage.com"
  databasePort: "5432"
  databaseName: "todo_chatbot"
  environment: "development"
  logLevel: "DEBUG"

secrets:
  # Injected at deployment time, never committed to git
  databasePassword: ""
  openaiApiKey: ""
```

### Entity: Helm Release (Deployed Instance)

**Purpose**: Instance of a Helm chart deployed to a cluster

**Attributes**:
- `name`: `todo-chatbot` (release name)
- `namespace`: `default`
- `chart`: `todo-chatbot` version `1.0.0`
- `status`: `deployed` (or `pending-install`, `pending-upgrade`, etc.)
- `revision`: `1` (incremented on each install/upgrade)

**Lifecycle**:
1. **Install**: `helm install todo-chatbot ./helm/todo-chatbot/`
2. **Upgrade**: `helm upgrade todo-chatbot ./helm/todo-chatbot/` (rolls out new version)
3. **Rollback**: `helm rollback todo-chatbot` (restores previous revision)
4. **Delete**: `helm uninstall todo-chatbot` (removes all resources)

---

## 8. Helm Template Model

### Template Variable Reference

**Common Variables**:
```yaml
{{ .Release.Name }}           # "todo-chatbot" (release name)
{{ .Chart.Name }}             # "todo-chatbot" (chart name)
{{ .Chart.Version }}          # "1.0.0"
{{ .Values.frontend.replicas }}       # 1 (nested value access)
{{ .Namespace }}              # "default"
```

**Template Functions** (Helm Built-ins):
```yaml
{{ .Values.image.tag | default "latest" }}           # Fallback value
{{ .Values.imageName | quote }}                      # Wrap in quotes
{{ .Values.config.labels | nindent 2 }}             # Indent YAML structure
{{ include "helpers.labels" . }}                     # Call helper template
```

**Conditionals**:
```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}
```

**Loops**:
```yaml
{{- range $key, $value := .Values.env }}
- name: {{ $key }}
  value: {{ $value }}
{{- end }}
```

---

## 9. Relationships & Dependencies

### Architecture Diagram (Text)

```
┌─────────────────────────────────────────────────────────┐
│                  Kubernetes Cluster (Minikube)         │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Deployment: frontend                            │  │
│  │  Replicas: 1                                     │  │
│  ├──────────────────────────────────────────────────┤  │
│  │  Pod: frontend-6f7d9c8b                          │  │
│  │  Image: todo-frontend:latest                     │  │
│  │  Port: 3000                                      │  │
│  │  Resources: 100m/256Mi → 500m/512Mi             │  │
│  │  Health: /health (liveness + readiness)         │  │
│  └──────────────────────────────────────────────────┘  │
│                        ↓                                 │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Service: frontend (LoadBalancer)                │  │
│  │  Port: 3000 → Pod:3000                          │  │
│  └──────────────────────────────────────────────────┘  │
│                        ↓                                 │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Deployment: backend                             │  │
│  │  Replicas: 1-N (configurable)                    │  │
│  ├──────────────────────────────────────────────────┤  │
│  │  Pods: backend-4d2f1a9e, backend-5e3g2b0f ...   │  │
│  │  Image: todo-backend:latest                      │  │
│  │  Port: 8000                                      │  │
│  │  Resources: 200m/512Mi → 1000m/1Gi             │  │
│  │  Health: /health (DB connectivity)              │  │
│  │  Env: DATABASE_URL, OPENAI_API_KEY (Secrets)   │  │
│  └──────────────────────────────────────────────────┘  │
│                        ↓                                 │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Service: backend (ClusterIP)                    │  │
│  │  DNS: backend:8000 (internal)                    │  │
│  │  Port: 8000 → Pod:8000 (round-robin)           │  │
│  └──────────────────────────────────────────────────┘  │
│                                                          │
│  ┌──────────────────────────────────────────────────┐  │
│  │  ConfigMap: config                               │  │
│  │  BACKEND_URL, DATABASE_HOST, etc.               │  │
│  └──────────────────────────────────────────────────┘  │
│                        ↓                                 │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Secret: secrets                                 │  │
│  │  DATABASE_PASSWORD, OPENAI_API_KEY (b64)        │  │
│  └──────────────────────────────────────────────────┘  │
│                                                          │
└─────────────────────────────────────────────────────────┘
                        ↓
                (via DATABASE_URL Secret)
                        ↓
            ┌─────────────────────────┐
            │   Neon PostgreSQL DB    │
            │   (External to cluster) │
            └─────────────────────────┘
```

### Data Flow

1. **User Access**: Browser → LoadBalancer Service:3000 → Frontend Pod:3000
2. **Frontend → Backend**: Frontend Pod → ClusterIP Service → Backend Pod:8000 (DNS resolution `backend:8000`)
3. **Backend → Database**: Backend Pod → uses DATABASE_URL Secret → Neon PostgreSQL (external)
4. **Configuration**: ConfigMap injected as environment variables in Deployment spec
5. **Secrets**: Kubernetes Secret injected as environment variables in Deployment spec

---

## 10. State Transitions & Lifecycle

### Pod Lifecycle

```
Pending → Running → Succeeded OR Failed OR Unknown
           ↓
      (if RestartPolicy: Always)
           ↓
         Running (restart)
```

**Health Check Impact**:
- **Liveness Probe Fails**: Pod marked as failed; restarted by kubelet
- **Readiness Probe Fails**: Pod marked as not ready; removed from service endpoints (but not restarted)

### Deployment Update Lifecycle

```
Current Pods (revision:1)
           ↓
       (helm upgrade)
           ↓
Create New Pod (revision:2, new image)
           ↓
(wait for readiness probe)
           ↓
Service routes to new pod
           ↓
Terminate old pod
           ↓
Updated Deployment (all pods running new image)
           ↓
(on failure, rollback via helm rollback)
           ↓
Back to revision:1 pods
```

---

## Summary: Infrastructure Data Model Complete

✅ Docker Image model (frontend + backend)
✅ Kubernetes Pod model with health checks & resource constraints
✅ Kubernetes Deployment model (rolling updates)
✅ Kubernetes Service model (frontend: LoadBalancer, backend: ClusterIP)
✅ ConfigMap model (non-sensitive config)
✅ Secret model (sensitive credentials)
✅ Helm Chart model (templated deployments)
✅ Architecture diagram (text representation)
✅ Data flow (user → frontend → backend → database)
✅ Lifecycle transitions (pod creation, updates, failures)

**Ready for Phase 1 (API Contracts)**
