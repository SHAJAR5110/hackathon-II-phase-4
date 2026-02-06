# Implementation Plan: Cloud Native Todo Chatbot with Local Kubernetes Deployment

**Branch**: `002-k8s-deployment` | **Date**: 2026-02-03 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/002-k8s-deployment/spec.md`

## Summary

Phase IV enables deployment of the Phase III Todo Chatbot to a local Kubernetes cluster (Minikube) using containerization (Docker), infrastructure-as-code (Helm charts), and AI-assisted operations (kubectl-ai, Kagent, Gordon). The implementation follows a phased approach: (1) containerize frontend/backend with Dockerfiles and Gordon; (2) define infrastructure with Helm charts; (3) deploy to Minikube with kubectl-ai/Kagent; (4) validate inter-service communication and pod orchestration. MVP focuses on manual scaling with external database (Neon); stateless deployments enable horizontal scalability.

## Technical Context

**Containerization Technologies**:
- **Docker**: Multi-stage builds for optimized frontend (<500MB) and backend (<300MB) images
- **Frontend Build**: Next.js/React → Docker image with Node.js runtime and built assets
- **Backend Build**: FastAPI (Python 3.11+) → Docker image with Python runtime, dependencies, health check endpoint

**Kubernetes & Orchestration**:
- **Cluster**: Minikube (single-node local cluster running on Docker Desktop)
- **Package Manager**: Helm 3+ with charts for frontend and backend deployments
- **Resource Management**: ConfigMaps (non-sensitive config), Secrets (database credentials, API keys), Deployments (replicas, health checks), Services (networking)

**AI DevOps Tools**:
- **Gordon (Docker AI)**: Primary Docker interface for building images, optimizing Dockerfiles, diagnosing container failures
- **kubectl-ai**: Natural language interface for Kubernetes operations (deploy, scale, troubleshoot)
- **Kagent**: Cluster health analysis, resource optimization recommendations
- **Fallback**: Manual Docker CLI, standard kubectl commands if AI tools unavailable

**Database**: PostgreSQL (Neon) - external to cluster; connection via environment variables/Secrets

**Testing**: Helm chart validation (helm lint), image build verification, pod deployment checks, inter-service connectivity tests

**Target Platform**: Local Linux/Windows/macOS via Docker Desktop; single-node Minikube cluster

**Project Type**: Infrastructure-as-code (Dockerfiles + Helm charts) + DevOps tooling documentation

**Performance Goals**: Container startup <60s, pod readiness <30s, deployment scale completion <2 minutes, kubectl-ai command success 90%

**Constraints**: Minikube cluster <8GB RAM, frontend image <500MB, backend image <300MB, no persistent storage in MVP

**Scale/Scope**: Single frontend instance + 1-3 backend replicas; ~200 lines Dockerfile, ~500 lines Helm YAML, ~100 lines documentation

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### Constitution Alignment Assessment

✅ **I. Stateless Server Architecture**: Phase IV doesn't alter backend logic; containerization preserves stateless design. Kubernetes deployments are stateless; database remains single source of truth.

✅ **II. MCP Tools as AI Interface**: No changes to MCP tools; they remain stateless and database-backed whether running in Docker or Kubernetes.

✅ **III. Conversation History as Source of Truth**: Database persistence unchanged. Containerization is transparent to conversation history model.

✅ **IV. Framework Correctness (ChatKit/OpenAI Agents)**: Containerization is infrastructure-layer only; no changes to ChatKit or OpenAI Agents SDK usage.

✅ **V. User-Centric Natural Language**: kubectl-ai/Kagent/Gordon are UI layers over Kubernetes/Docker; they improve UX but don't change underlying chatbot behavior.

⚠️ **VI. Authentication & Authorization**: Minikube is localhost-only (development). No multi-user auth required for MVP. If deployment moves to production Kubernetes, verify network policies and pod security policies don't conflict with auth model.

✅ **VII. Database-First Design**: External Neon database unchanged. Kubernetes connection via Secrets; SQLModel ORM unaffected.

✅ **VIII. Error Handling & Graceful Degradation**: Container health checks (liveness/readiness probes) and pod restart policies ensure graceful recovery. No changes to API error handling.

✅ **IX. Testability & Observability**: Helm charts include pod logs accessibility; kubectl/kubectl-ai enable debugging. No impact on existing unit/integration tests.

✅ **X. Simplicity & YAGNI**: Helm charts are simple; no custom operators or CRDs. Manual scaling keeps MVP focused.

**GATE RESULT**: ✅ **PASS** - Phase IV is infrastructure-only; all constitution principles preserved. Containerization is implementation detail transparent to core chatbot logic.

## Project Structure

### Documentation (this feature)

```text
specs/002-k8s-deployment/
├── spec.md                          # Feature specification (completed)
├── plan.md                          # This file (implementation plan)
├── research.md                      # Phase 0: Research notes (to be generated)
├── data-model.md                    # Phase 1: Infrastructure models (to be generated)
├── quickstart.md                    # Phase 1: Deployment quick-start guide (to be generated)
├── contracts/                       # Phase 1: Docker/Helm specifications (to be generated)
│   ├── dockerfile.spec.md           # Frontend/Backend Dockerfile requirements
│   ├── helm-chart.spec.md           # Helm chart structure and values
│   └── kubernetes-resources.spec.md # K8s Deployments, Services, ConfigMaps, Secrets
├── checklists/
│   └── requirements.md              # Spec quality validation (completed)
└── tasks.md                         # Phase 2: Task breakdown (generated by /sp.tasks)
```

### Source Code (repository root)

```text
# Infrastructure & DevOps deliverables
docker/
├── frontend/
│   └── Dockerfile                    # Frontend (Next.js/React) containerization
├── backend/
│   └── Dockerfile                    # Backend (FastAPI) containerization
└── docker-compose.yml               # Optional: Local multi-container testing

helm/
├── todo-chatbot/                    # Parent Helm chart (values-driven)
│   ├── Chart.yaml                   # Chart metadata
│   ├── values.yaml                  # Default values (Minikube development)
│   ├── values-prod.yaml             # Production overrides (if needed later)
│   ├── templates/
│   │   ├── deployment-frontend.yaml
│   │   ├── deployment-backend.yaml
│   │   ├── service-frontend.yaml
│   │   ├── service-backend.yaml
│   │   ├── configmap.yaml
│   │   ├── secret.yaml
│   │   ├── _helpers.tpl
│   │   └── NOTES.txt
│   └── README.md

docs/
├── DEPLOYMENT.md                    # Step-by-step deployment guide
├── KUBECTL-AI.md                    # kubectl-ai command examples
├── KAGENT.md                        # Kagent monitoring guide
├── GORDON.md                        # Docker AI (Gordon) usage guide
└── TROUBLESHOOTING.md              # Common issues and diagnostics

tests/
├── helm-validation/
│   └── test-helm-charts.sh         # helm lint, helm template validation
├── docker-build/
│   └── test-docker-builds.sh       # Docker image build verification
└── integration/
    └── test-deployment.sh          # End-to-end Minikube deployment test

# Existing Phase III application (unchanged)
backend/                             # Phase III FastAPI backend
frontend/                            # Phase III React/Next.js frontend
```

**Structure Decision**: Infrastructure-as-code approach with separate `docker/` and `helm/` directories. Helm chart uses subchart pattern for reusability. Comprehensive deployment documentation. Tests focus on infrastructure validation (lint, build, deploy) rather than application logic (handled in Phase III tests).

## Complexity Tracking

**Status**: ✅ No constitution violations. No complexity justification needed.

All design choices follow YAGNI principle:
- Manual replica scaling (no HPA) keeps MVP simple; can add in Phase V
- External database (no PersistentVolumes) avoids storage provisioning complexity
- Gordon integrated but with fallback to manual Docker CLI (no forced dependency)
- Simple Helm charts without custom operators or CRDs
- No GitOps, multi-cluster, or service mesh additions

---

## Phase 0: Research Tasks

**Status**: Ready to generate research artifacts

Research topics to consolidate:
1. **Dockerfile Best Practices**: Multi-stage builds, base image selection (Alpine vs Debian), layer caching optimization
2. **Helm Chart Structure**: Best practices for production-grade charts, templating patterns, values override strategy
3. **Minikube Configuration**: Resource allocation, ingress setup, storage provisioner, network policies
4. **kubectl-ai Integration**: CLI syntax, command templates, error handling patterns
5. **Kagent Capabilities**: Health check interpretation, resource recommendation algorithms
6. **Gordon Workflow**: Image optimization suggestions, Dockerfile generation, fallback command generation
7. **Kubernetes Security Practices**: Pod security policies, network policies (development scope), RBAC (for future)

---

## Phase 1: Design Artifacts

### Data Model (Infrastructure Model)

**Docker Images**:
- **Frontend Image**: Node.js 18+ base, Next.js build artifacts, health check at `/health`
- **Backend Image**: Python 3.11 base, FastAPI with dependencies, health check at `/health` with database connectivity check
- **Image Registry**: Local Docker Desktop registry or Docker Hub (configurable)

**Helm Chart Structure**:
- **Values**: replicas (default 1 frontend, 1 backend), image tags, resource requests/limits, port mappings
- **Deployments**: Pod template with health checks, environment variable injection from ConfigMaps/Secrets
- **Services**: ClusterIP for inter-service discovery, optional LoadBalancer for frontend (dev only)
- **ConfigMaps**: Non-sensitive environment variables (API URLs, database host)
- **Secrets**: Database credentials, API keys (base64-encoded)

**Kubernetes Cluster**:
- **Minikube**: Single node, Docker Desktop driver, 4GB RAM recommended
- **Namespaces**: default (MVP scope; production would use separate namespaces)
- **Pod Networking**: Kubernetes DNS for service discovery (frontend discovers backend via `backend:8000`)

### API Contracts (Infrastructure Specifications)

**Dockerfile Frontend**:
```
Base: node:18-alpine
Build: npm/yarn build → Next.js production bundle
Expose: 3000
Health: GET /health → 200 OK
Env Vars: BACKEND_URL, NEXT_PUBLIC_API_URL
```

**Dockerfile Backend**:
```
Base: python:3.11-slim
Build: pip install -r requirements.txt
Expose: 8000
Health: GET /health → checks DB connectivity, returns 200 OK
Env Vars: DATABASE_URL, OPENAI_API_KEY, MCP_SERVER_ENDPOINT
```

**Helm Chart Values** (development):
```yaml
frontend:
  replicas: 1
  image:
    repository: localhost:5000/todo-frontend  # or docker.io/username/todo-frontend
    tag: latest
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi
  service:
    type: LoadBalancer
    port: 3000

backend:
  replicas: 1
  image:
    repository: localhost:5000/todo-backend
    tag: latest
  resources:
    requests:
      cpu: 200m
      memory: 512Mi
    limits:
      cpu: 1000m
      memory: 1Gi
  service:
    type: ClusterIP
    port: 8000
```

**Kubernetes Manifest Contract** (generated from Helm):
- Deployments: one for frontend, one for backend with health check probes
- Services: stable DNS names for internal routing
- ConfigMaps: environment variable configuration
- Secrets: sensitive credential management

### Agent Context Update

Run `.specify/scripts/bash/update-agent-context.sh claude` to:
- Detect current agent (Claude Haiku 4.5)
- Update agent-specific context file with:
  - Docker & Dockerfile syntax patterns
  - Helm chart templating syntax
  - Kubernetes YAML structure
  - kubectl-ai command patterns
  - Kagent analysis output format

---

## Implementation Roadmap

### Phase 0 Outputs (Research)
- `research.md`: Consolidated findings on Docker best practices, Helm patterns, Kubernetes configuration, AI tool workflows

### Phase 1 Outputs (Design)
- `data-model.md`: Infrastructure entities (Docker images, Helm charts, K8s resources)
- `contracts/`: Specifications for Dockerfiles, Helm charts, K8s manifests
- `quickstart.md`: 5-minute deployment walkthrough for Minikube
- Agent context file: Updated with new technologies and patterns

### Phase 2 Outputs (Tasks - via /sp.tasks)
- `tasks.md`: Granular, testable implementation tasks:
  - Task 1: Create frontend Dockerfile
  - Task 2: Create backend Dockerfile
  - Task 3: Build and test Docker images
  - Task 4: Create Helm chart structure
  - Task 5: Define Helm values and templates
  - Task 6: Validate Helm charts
  - Task 7: Deploy to Minikube
  - Task 8: Verify pod health and inter-service communication
  - Task 9: Document kubectl-ai workflows
  - Task 10: Document Kagent monitoring
  - Task 11: Document Gordon (Docker AI) integration
  - Task 12: Write integration tests
  - Task 13: Create troubleshooting guide
