# Kubernetes Deployment Guide - Quick Reference

This document supplements the DEPLOYMENT.md with quick reference for Kubernetes-specific operations.

---

## 10-Minute Quick Start

### Prerequisites
```bash
✓ Docker Desktop 4.53+
✓ kubectl CLI
✓ helm 3+
✓ minikube CLI
✓ Phase III source code (frontend/, backend/)
✓ Database credentials
```

### Step 1: Start Minikube (1 min)
```bash
minikube start --driver=docker
kubectl cluster-info
```

### Step 2: Build Images (3 min)
```bash
docker build -f docker/frontend/Dockerfile -t todo-frontend:latest .
docker build -f docker/backend/Dockerfile -t todo-backend:latest .
```

### Step 3: Load & Deploy (2 min)
```bash
minikube image load todo-frontend:latest
minikube image load todo-backend:latest
helm install todo-chatbot ./helm/todo-chatbot/ \
  --set secrets.databasePassword="YOUR_PASSWORD" \
  --set secrets.openaiApiKey="YOUR_KEY"
```

### Step 4: Access (1 min)
```bash
kubectl port-forward svc/todo-chatbot-frontend 3000:3000 &
# Open http://localhost:3000
```

### Step 5: Verify (1 min)
```bash
kubectl get pods                    # All should show Running
curl http://localhost:8000/health   # Should return 200 OK
```

---

## Key Kubernetes Concepts

### Pods
- Smallest deployable unit in Kubernetes
- One or more containers (usually one)
- Ephemeral - created and destroyed dynamically

### Deployments
- Manages replicas of Pods
- Rolling updates - new version gradually replaces old
- Health checks ensure pods stay running

### Services
- Stable endpoint for accessing pods
- Types:
  - **ClusterIP**: Internal only (backend)
  - **LoadBalancer**: External access (frontend)

### ConfigMaps
- Non-sensitive configuration (environment variables)
- Example: `BACKEND_URL`, `DATABASE_HOST`

### Secrets
- Sensitive data (credentials, API keys)
- Base64-encoded (not encrypted in MVP)
- Example: `DATABASE_PASSWORD`, `OPENAI_API_KEY`

---

## Essential kubectl Commands

```bash
# View resources
kubectl get pods                        # List pods
kubectl get deployments                 # List deployments
kubectl get services                    # List services
kubectl get configmap                   # List config maps
kubectl get secret                      # List secrets

# Detailed info
kubectl describe pod <pod-name>         # Pod details
kubectl describe deployment <name>      # Deployment details
kubectl describe service <name>         # Service details

# Logs
kubectl logs <pod-name>                 # View logs
kubectl logs -f <pod-name>              # Stream logs
kubectl logs deployment/<name>          # Deployment logs

# Execute commands
kubectl exec -it <pod-name> -- /bin/sh # Shell into pod
kubectl port-forward <pod-name> 8000:8000  # Local port

# Status
kubectl get pods --watch               # Watch pod changes
kubectl rollout status deployment/<name> # Deployment progress
```

---

## Helm Essentials

```bash
# Install
helm install <name> <chart-path>        # Deploy

# Inspect
helm list                               # List releases
helm status <name>                      # Release status
helm get values <name>                  # Current values
helm history <name>                     # Version history

# Update
helm upgrade <name> <chart-path>        # Update release
helm upgrade <name> <chart-path> \
  --set key=value                       # With new values

# Rollback
helm rollback <name>                    # To previous version
helm rollback <name> 1                  # To specific revision

# Template
helm template <name> <chart-path>       # Preview manifests
helm lint <chart-path>                  # Validate chart

# Uninstall
helm uninstall <name>                   # Delete release
```

---

## Troubleshooting Flow

### Pod Issues
```
Pod Pending?    → kubectl describe pod <name> → Check Events
Pod Crashing?   → kubectl logs <name>         → Check logs
Pod OOMKilled?  → Increase memory limit       → helm upgrade
```

### Service Issues
```
Can't reach service?  → kubectl port-forward ... 3000:3000
No endpoints?         → kubectl describe service <name>
                      → Check pod labels vs selector
```

### Image Issues
```
ImagePullBackOff?     → minikube image ls       → Check if loaded
                      → minikube image load ... → Load image
```

---

## Useful Environment Variables

In your deployment:

```yaml
# Frontend
BACKEND_URL: "http://todo-chatbot-backend:8000"
NEXT_PUBLIC_API_URL: (same as BACKEND_URL)
NODE_ENV: "production"

# Backend
DATABASE_HOST: "neon-staging.us-east-1.postgres.vercel-storage.com"
DATABASE_PORT: "5432"
DATABASE_NAME: "todo_chatbot"
DATABASE_PASSWORD: (from Secret)
OPENAI_API_KEY: (from Secret)
ENVIRONMENT: "development"
LOG_LEVEL: "DEBUG"
```

---

## Common Scenarios

### Scale Backend to 3 Replicas
```bash
kubectl scale deployment todo-chatbot-backend --replicas=3
# Or via Helm:
helm upgrade todo-chatbot ./helm/todo-chatbot/ \
  --set backend.replicaCount=3
```

### View Backend Logs
```bash
kubectl logs -f deployment/todo-chatbot-backend

# Or last 100 lines
kubectl logs deployment/todo-chatbot-backend --tail=100
```

### Port-Forward Both Services
```bash
# Terminal 1
kubectl port-forward svc/todo-chatbot-frontend 3000:3000

# Terminal 2
kubectl port-forward svc/todo-chatbot-backend 8000:8000

# Access: http://localhost:3000 and http://localhost:8000
```

### Update Configuration
```bash
# Update ConfigMap
kubectl set env deployment/todo-chatbot-backend \
  LOG_LEVEL=INFO

# Or rebuild with Helm
helm upgrade todo-chatbot ./helm/todo-chatbot/ \
  --set config.logLevel=INFO
```

### Recreate Pods (Restart)
```bash
kubectl rollout restart deployment/todo-chatbot-frontend
kubectl rollout restart deployment/todo-chatbot-backend
```

### Delete Everything and Start Over
```bash
helm uninstall todo-chatbot
kubectl get pods --watch          # Wait for deletion
minikube delete                   # Fresh cluster
minikube start --driver=docker    # Restart
```

---

## Health Check Endpoints

### Frontend Health
```bash
kubectl port-forward svc/todo-chatbot-frontend 3000:3000
curl http://localhost:3000/health
# Expected: 200 OK
```

### Backend Health
```bash
kubectl port-forward svc/todo-chatbot-backend 8000:8000
curl http://localhost:8000/health
# Expected: 200 OK with {"status": "healthy", "database": "connected"}
```

---

## Monitoring & Debugging

### Check All Resources
```bash
kubectl get all          # All deployments, services, pods
kubectl get events       # Recent cluster events
kubectl top nodes        # Node resource usage
kubectl top pods         # Pod resource usage
```

### Deep Diagnostics
```bash
# Describe everything about a deployment
kubectl describe deployment todo-chatbot-frontend

# Get resource YAML
kubectl get deployment todo-chatbot-frontend -o yaml

# Check readiness/liveness probes
kubectl describe pod <pod-name> | grep -A 10 "Probes"
```

### Real-Time Monitoring
```bash
# Watch pods as they change
kubectl get pods -w

# Watch specific pod
kubectl get pod <pod-name> -w

# Watch events
kubectl get events -w
```

---

## Helm Chart Structure Reference

```
helm/todo-chatbot/
├── Chart.yaml              # Chart metadata
├── values.yaml            # Configuration values
├── values-dev.yaml        # Dev overrides (optional)
└── templates/
    ├── deployment-frontend.yaml   # Frontend Pod manager
    ├── deployment-backend.yaml    # Backend Pod manager
    ├── service-frontend.yaml      # Frontend network
    ├── service-backend.yaml       # Backend network
    ├── configmap.yaml            # App config
    ├── secret.yaml               # Credentials
    ├── _helpers.tpl              # Template helpers
    └── NOTES.txt                 # Post-install info
```

---

## Next Steps

1. **Deploy**: Follow [DEPLOYMENT.md](./DEPLOYMENT.md)
2. **Understand kubectl**: Read [KUBECTL-AI.md](./KUBECTL-AI.md)
3. **Optimize Docker**: Check [GORDON.md](./GORDON.md)
4. **Fix Issues**: See [TROUBLESHOOTING.md](./TROUBLESHOOTING.md)

---

## Quick Command Reference Card

```bash
# Check status
minikube status
kubectl cluster-info
kubectl get pods
kubectl get deployments

# Deploy
helm install todo-chatbot ./helm/todo-chatbot/
helm upgrade todo-chatbot ./helm/todo-chatbot/

# Access
kubectl port-forward svc/todo-chatbot-frontend 3000:3000
kubectl port-forward svc/todo-chatbot-backend 8000:8000

# Debug
kubectl logs <pod>
kubectl describe pod <pod>
kubectl exec -it <pod> -- /bin/sh

# Scale
kubectl scale deployment <name> --replicas=3

# Cleanup
helm uninstall todo-chatbot
minikube stop
```

---

**Time to deploy**: ~15 minutes | **Difficulty**: Beginner-friendly
