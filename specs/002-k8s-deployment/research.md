# Research & Best Practices: Cloud Native Todo Chatbot Kubernetes Deployment

**Phase**: 0 (Research)
**Date**: 2026-02-03
**Feature**: 002-k8s-deployment
**Status**: Complete

---

## 1. Docker Containerization Best Practices

### Decision: Multi-Stage Dockerfile with Alpine Base Images

**Rationale**:
- Multi-stage builds reduce final image size by separating build dependencies from runtime
- Alpine Linux (4-5MB base) vs Debian (50-100MB) significantly impacts image pull time and storage
- For local Minikube development, faster image builds improve iteration speed

**Frontend (Next.js/React) Dockerfile Strategy**:
- **Build stage**: Node.js 18-alpine with full dependencies (npm, build tools)
- **Runtime stage**: Node.js 18-alpine with built artifacts only (production mode)
- **Result**: ~200-300MB final image (vs 500MB+ without multi-stage)
- **Health check**: GET `/health` endpoint returning 200 OK (Next.js built-in or custom page)

**Backend (FastAPI) Dockerfile Strategy**:
- **Build stage**: Python 3.11-slim with pip install and compilation (if needed)
- **Runtime stage**: Python 3.11-slim with installed packages from build cache
- **Result**: ~200-250MB final image
- **Health check**: GET `/health` endpoint with database connectivity verification

### Layer Caching Optimization

**Best Practice**:
1. COPY requirements.txt/package.json first (they change infrequently)
2. RUN pip install/npm install (builds cached layer)
3. COPY source code last (changes frequently, doesn't invalidate dependency cache)
4. Result: Rebuilding after code changes reuses dependency layer (seconds vs minutes)

### Environment Variables & Configuration

**Docker Container Design**:
- Health check endpoint requires no external dependencies (self-contained liveness check)
- Database connection string injected at runtime via `-e` flag or environment file
- Secrets (API keys) passed via environment variables, never hardcoded in image
- Port mappings: Frontend 3000, Backend 8000 (standard conventions)

---

## 2. Kubernetes & Helm Chart Architecture

### Decision: Helm 3+ with Simple Chart Structure (No Subcharts)

**Rationale**:
- Helm 3 simplified chart installation and security (no Tiller required)
- Simple monolithic chart easier to understand and modify for MVP
- Subcharts add complexity not needed until managing 10+ microservices
- Values-driven approach enables environment-specific overrides (dev, staging, prod)

**Chart Organization**:
```
todo-chatbot/
├── Chart.yaml          # Chart metadata (name, version)
├── values.yaml         # Default development values
├── values-prod.yaml    # Production overrides (if needed)
├── templates/
│   ├── deployment-frontend.yaml
│   ├── deployment-backend.yaml
│   ├── service-frontend.yaml
│   ├── service-backend.yaml
│   ├── configmap.yaml  # Non-sensitive environment variables
│   ├── secret.yaml     # Template for Secrets (values injected)
│   ├── _helpers.tpl    # Template helpers (labels, selectors)
│   └── NOTES.txt       # Post-deployment instructions
└── README.md
```

### Kubernetes Resource Specifications

**Deployments**:
- **Replicas**: Configurable via values.yaml (default 1 frontend, 1 backend)
- **Health Checks**:
  - **Liveness Probe**: Checks if pod is alive; restarts if failing
  - **Readiness Probe**: Checks if pod is ready to serve traffic; removes from service if failing
  - **Init Check**: Simple HTTP GET to `/health` with 3s timeout, 10s period
- **Resource Requests/Limits**:
  - Frontend requests: 100m CPU, 256Mi memory; limits: 500m CPU, 512Mi memory
  - Backend requests: 200m CPU, 512Mi memory; limits: 1000m CPU, 1Gi memory
  - Prevents pod eviction on resource-constrained nodes; enables scheduler to pack pods efficiently

**Services**:
- **Frontend Service**:
  - Type: `LoadBalancer` (in Minikube, simulates external IP for localhost access)
  - Port: 3000, TargetPort: 3000
- **Backend Service**:
  - Type: `ClusterIP` (internal only; no external access)
  - Port: 8000, TargetPort: 8000
  - Stable DNS: `backend:8000` accessible from frontend pod

**ConfigMap**:
- Non-sensitive configuration (API base URLs, database host)
- Mounted as environment variables in pod spec
- Example: `DATABASE_HOST=postgres.neon.tech`, `BACKEND_URL=http://backend:8000`

**Secret**:
- Base64-encoded sensitive data (database password, API keys)
- Kubernetes does NOT encrypt at rest by default (development MVP scope)
- Should be injected via environment variables or volume mounts
- Values template uses placeholder; actual values provided at deployment time

### Helm Templating Patterns

**Template Syntax**:
```yaml
{{ .Release.Name }}               # Release name (e.g., "todo-chatbot")
{{ .Chart.Name }}                 # Chart name
{{ .Values.frontend.replicas }}   # Access nested values
{{ include "helpers.labels" . }}  # Call helper templates
{{ .Values.image.tag | default "latest" }}  # Filters (default, quote, nindent)
```

**Conditional Logic**:
```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}
```

**Looping**:
```yaml
{{- range .Values.env }}
- name: {{ .name }}
  value: {{ .value }}
{{- end }}
```

---

## 3. Minikube Configuration & Local Development

### Decision: Docker Desktop Driver for Minikube

**Rationale**:
- Docker Desktop (Windows/macOS/Linux) manages Minikube cluster as Docker container
- No hypervisor complexity; leverages existing Docker installation
- Automatic resource allocation; developers specify CPU/memory via Docker Desktop settings
- Easy cluster reset (delete and recreate) for debugging

### Minikube Setup Best Practices

**Cluster Initialization**:
```bash
minikube start --driver=docker --cpus=4 --memory=8192 --disk-size=20gb
minikube addons enable metrics-server  # For Kagent resource analysis
minikube addons enable ingress         # Optional: for production routing
```

**Resource Constraints**:
- **Minimum for MVP**: 2 CPU cores, 4GB RAM
- **Recommended**: 4 CPU cores, 8GB RAM (allows comfortable pod overhead)
- **Disk Space**: 20GB for images and persistent data

**Local Registry**:
- Option 1: Use Minikube built-in registry (`minikube image load`)
- Option 2: Use local Docker Desktop registry (localhost:5000 with docker run -d registry:latest)
- Option 3: Use Docker Hub (public images for testing)

**Port Forwarding** (accessing services from host):
```bash
kubectl port-forward svc/frontend 3000:3000
kubectl port-forward svc/backend 8000:8000
```

### Service Discovery in Kubernetes

**DNS Pattern**:
- Service name as hostname: `backend:8000` resolves to backend service
- Full DNS: `backend.default.svc.cluster.local` (namespace.svc.cluster.local)
- Frontend environment variable: `BACKEND_URL=http://backend:8000`

---

## 4. kubectl-ai Integration & Natural Language Commands

### Decision: kubectl-ai as Primary Kubernetes Interface

**Rationale**:
- Lowers learning curve; developers unfamiliar with kubectl can operate clusters
- Translates natural language to kubectl/helm commands; shows user the actual command
- Enables exploration-based learning (users see what kubectl commands are generated)
- Fallback: standard kubectl if kubectl-ai unavailable or misinterprets commands

### Common kubectl-ai Command Patterns

**Deployment Operations**:
```bash
kubectl-ai "deploy the todo frontend with 2 replicas"
kubectl-ai "scale backend to 3 replicas"
kubectl-ai "show me the backend logs"
kubectl-ai "check why the frontend pod is failing"
```

**Cluster Inspection**:
```bash
kubectl-ai "list all running pods"
kubectl-ai "describe the backend deployment"
kubectl-ai "show me the status of services"
```

**Troubleshooting**:
```bash
kubectl-ai "why are the pods not starting"
kubectl-ai "what's the cpu and memory usage"
kubectl-ai "port forward frontend to 3000"
```

### kubectl-ai Output & Validation

**Expected Behavior**:
1. User enters natural language command
2. kubectl-ai analyzes intent and generates appropriate kubectl/helm command
3. kubectl-ai shows command to user for confirmation (best practice: always review before execution)
4. User approves; command executes
5. kubectl-ai returns result (pods created, scaled, logs displayed)

**Error Handling**:
- If kubectl-ai misunderstands, user can refine command or use manual kubectl
- Fallback: "kubectl describe pod <name>" provides detailed troubleshooting info

---

## 5. Kagent: Cluster Health Analysis & Optimization

### Decision: Kagent for P2 Health Monitoring (Phase IV MVP)

**Rationale**:
- Provides advanced cluster diagnostics beyond basic kubectl-ai
- Analyzes resource utilization, pod health, inter-service communication
- Generates optimization recommendations (replica count, resource limits)
- Read-only operations (no destructive changes); safe to experiment

### Kagent Capabilities for Todo Chatbot MVP

**Cluster Health Analysis**:
```bash
kagent "analyze the cluster health"
# Output: Pod status, resource usage (CPU/memory), error rates, bottlenecks
```

**Resource Optimization**:
```bash
kagent "optimize resource allocation"
# Output: Recommendations (increase frontend replicas, adjust memory limits)
```

**Inter-Service Communication**:
```bash
kagent "analyze network topology"
# Output: Service connections, latency, DNS resolution status
```

**Performance Bottleneck Detection**:
```bash
kagent "what are the performance bottlenecks"
# Output: High latency services, resource-constrained nodes, pod evictions
```

### Kagent Integration Pattern

1. Deploy baseline configuration
2. Run Kagent analysis to establish baseline metrics
3. Identify optimization opportunities
4. Apply recommendations (e.g., increase replicas)
5. Re-analyze to measure improvement

---

## 6. Docker AI Agent (Gordon) - Fully Integrated

### Decision: Gordon as Primary Docker Interface (with Manual CLI Fallback)

**Rationale**:
- Automates Dockerfile optimization and image build process
- Reduces manual trial-and-error in containerization
- Provides diagnostics when containers fail to start
- Fallback: Manual Docker CLI and Claude Code command generation if Gordon unavailable (regional limitations, tier restrictions)

### Gordon Capabilities for Todo Chatbot

**Image Optimization**:
```bash
docker ai "build an optimized image for the fastapi backend"
# Gordon may suggest: multi-stage build, Alpine base, dependency caching
```

**Dockerfile Generation**:
```bash
docker ai "generate a production-grade dockerfile for next.js frontend"
# Gordon generates Dockerfile with best practices (layer caching, health checks, etc.)
```

**Container Diagnostics**:
```bash
docker ai "why is the backend container failing to start"
# Gordon analyzes logs and suggests root cause (missing env var, port conflict, etc.)
```

**Image Analysis**:
```bash
docker ai "why is my backend image 800MB, how can I optimize it"
# Gordon suggests layer optimization, base image alternatives, build caching improvements
```

### Gordon Integration Pattern

1. **Enable Gordon**: Docker Desktop Settings > Beta features > toggle Gordon on
2. **Verify Capability**: Run `docker ai "what can you do?"`
3. **Use for Builds**: `docker ai "build optimized image for backend"` instead of manual docker build
4. **Fallback**: If Gordon unavailable, use manual commands or Claude Code generation

### Fallback Strategy (Regional/Tier Limitations)

If Gordon unavailable:
1. Use standard Docker CLI: `docker build -f docker/backend/Dockerfile -t todo-backend:latest .`
2. Reference manual Dockerfile best practices from research.md
3. Ask Claude Code to generate optimized Dockerfiles with explanations
4. Gordon can be added in Phase V post-MVP when fully available

---

## 7. Integration: Docker → Kubernetes → Development Workflow

### End-to-End Deployment Flow

1. **Local Development**:
   - Modify application code (backend or frontend)
   - Use Gordon or manual docker build to create image: `docker build -f docker/backend/Dockerfile -t todo-backend:latest .`
   - Test locally: `docker run -p 8000:8000 -e DATABASE_URL=... todo-backend:latest`

2. **Push to Registry**:
   - Tag image: `docker tag todo-backend:latest localhost:5000/todo-backend:latest`
   - Push: `docker push localhost:5000/todo-backend:latest` (or use Docker Hub for sharing)

3. **Update Helm Values**:
   - Edit `helm/todo-chatbot/values.yaml` to reference new image tag
   - Example: `backend.image.tag: latest`

4. **Deploy to Minikube**:
   - `helm upgrade --install todo-chatbot ./helm/todo-chatbot -f helm/todo-chatbot/values.yaml`
   - Or use kubectl-ai: `kubectl-ai "deploy todo chatbot with new backend image"`

5. **Verify Deployment**:
   - `kubectl get pods` (check pod status)
   - `kubectl logs -f deployment/todo-backend` (stream logs)
   - `kubectl port-forward svc/frontend 3000:3000` (access frontend)
   - Or use kubectl-ai: `kubectl-ai "check if pods are healthy and running"`

6. **Monitor & Optimize**:
   - `kagent "analyze cluster health"` (get performance insights)
   - `kubectl-ai "scale backend to 3 replicas"` (increase capacity if needed)

---

## 8. Testing Strategy for Infrastructure

### Helm Chart Validation

**Syntax Validation**:
```bash
helm lint ./helm/todo-chatbot/
# Output: No errors if chart is valid; validation errors otherwise
```

**Template Rendering**:
```bash
helm template todo-chatbot ./helm/todo-chatbot/ -f values.yaml
# Output: Complete Kubernetes YAML manifests (review for correctness)
```

**Dry-Run Deployment**:
```bash
helm install todo-chatbot ./helm/todo-chatbot/ --dry-run --debug
# Simulates deployment without creating actual resources
```

### Docker Image Testing

**Build Verification**:
```bash
docker build -f docker/frontend/Dockerfile -t todo-frontend:test . && echo "Build successful"
```

**Runtime Testing**:
```bash
docker run -d --name test-frontend -p 3000:3000 todo-frontend:test
curl http://localhost:3000/health  # Should return 200 OK
docker stop test-frontend && docker rm test-frontend
```

**Image Size Check**:
```bash
docker images todo-frontend:test
# Verify image size < 500MB for frontend, < 300MB for backend
```

### Integration Testing

**Minikube Deployment Test**:
```bash
minikube start --driver=docker
helm install todo-chatbot ./helm/todo-chatbot/ -f values.yaml
kubectl wait --for=condition=ready pod -l app=frontend --timeout=300s
kubectl wait --for=condition=ready pod -l app=backend --timeout=300s
kubectl get svc  # Verify services created
kubectl port-forward svc/frontend 3000:3000 &
curl http://localhost:3000/health  # Verify frontend health
curl http://localhost:3000  # Verify frontend loads
kubectl port-forward svc/backend 8000:8000 &
curl http://localhost:8000/health  # Verify backend health
```

---

## 9. Security Considerations (Development MVP Scope)

### Docker Image Security

**Best Practices**:
- Use specific base image versions (not `latest`)
- Multi-stage builds exclude build tools from final image
- Run as non-root user in container (add USER directive in Dockerfile)
- Scan images for vulnerabilities (docker scan todo-backend:latest)

**MVP Scope**:
- Minikube is localhost-only; no public exposure
- Secrets not encrypted at rest in MVP (acceptable for development)
- No RBAC or network policies required for MVP (single-user, local cluster)

### Kubernetes Secrets Handling

**Best Practice**:
- Store database credentials in Kubernetes Secret, not in values.yaml
- Reference Secret in Deployment pod spec via environment variables
- Never commit actual secrets to git

**MVP Implementation**:
```yaml
# secret.yaml template
apiVersion: v1
kind: Secret
metadata:
  name: todo-secrets
type: Opaque
data:
  DATABASE_PASSWORD: {{ .Values.secrets.databasePassword | b64enc }}
  OPENAI_API_KEY: {{ .Values.secrets.openaiApiKey | b64enc }}
```

```bash
# Deploy with actual secrets:
helm install todo-chatbot ./helm/todo-chatbot \
  --set secrets.databasePassword=mypassword \
  --set secrets.openaiApiKey=sk-...
```

---

## 10. Documentation & Troubleshooting

### Deployment Guide Structure

**docs/DEPLOYMENT.md** (5-minute quick start):
1. Prerequisites (Docker Desktop, minikube, helm, kubectl)
2. Enable Gordon in Docker Desktop settings
3. Build images: `docker ai "build optimized images for frontend and backend"`
4. Deploy: `helm install todo-chatbot ./helm/todo-chatbot/ -f values.yaml`
5. Verify: `kubectl get pods` and `kubectl port-forward svc/frontend 3000:3000`

**docs/KUBECTL-AI.md** (command examples):
- Common deployment commands
- Scaling operations
- Troubleshooting commands
- Log retrieval patterns

**docs/KAGENT.md** (health monitoring):
- Cluster health analysis
- Resource optimization workflow
- Performance bottleneck detection

**docs/GORDON.md** (Docker AI):
- Enable/disable Gordon
- Common build optimizations
- Dockerfile troubleshooting
- Fallback to manual CLI

**docs/TROUBLESHOOTING.md** (common issues):
- Pod pending (resource constraints, image pull failures)
- Pod crash loops (health check failures, database connection issues)
- Service connectivity (DNS resolution, port forwarding)
- Image size issues (optimization suggestions)

---

## Summary: Research Complete

✅ Dockerfile best practices (multi-stage, Alpine base)
✅ Helm chart architecture (simple, values-driven)
✅ Minikube configuration (Docker Desktop driver)
✅ kubectl-ai integration patterns
✅ Kagent health monitoring workflow
✅ Gordon (Docker AI) full integration with manual fallback
✅ End-to-end deployment flow (code → container → k8s)
✅ Infrastructure testing strategy
✅ Security considerations for MVP
✅ Documentation structure

**Ready for Phase 1 (Design & Contracts)**
