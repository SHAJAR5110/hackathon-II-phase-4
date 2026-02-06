# Quick Start: Deploy Todo Chatbot to Minikube

**Estimated Time**: 15-20 minutes
**Prerequisites**: Docker Desktop, kubectl, helm, minikube CLI
**Target**: Local Minikube cluster with frontend & backend running

---

## Prerequisites Checklist

- [ ] Docker Desktop installed (4.53+ for Gordon support)
- [ ] Minikube CLI installed (`minikube version` should work)
- [ ] kubectl CLI installed (`kubectl version` should work)
- [ ] Helm 3+ CLI installed (`helm version` should work)
- [ ] Phase III Todo Chatbot source code available (frontend + backend)
- [ ] Database connection string for Neon PostgreSQL

---

## Step 1: Start Minikube Cluster (2 minutes)

```bash
# Start Minikube with Docker driver
minikube start --driver=docker --cpus=4 --memory=8192 --disk-size=20gb

# Verify cluster is running
kubectl cluster-info
kubectl get nodes

# Output should show:
# NAME       STATUS   ROLES                  AGE   VERSION
# minikube   Ready    control-plane,master   5s    v1.XX.X
```

**Troubleshoot**:
- If `minikube start` fails: Check Docker Desktop is running (`docker ps`)
- If resource-constrained: Reduce `--cpus=2 --memory=4096` (slower but functional)

---

## Step 2: Build Docker Images (5 minutes)

### Option A: Using Gordon (Docker AI) - Recommended

```bash
# Enable Gordon in Docker Desktop Settings > Beta features first

# Build frontend image
docker ai "build an optimized docker image for the next.js frontend application \
located in ./frontend directory, tag it as todo-frontend:latest, \
use alpine base image for minimal size"

# Build backend image
docker ai "build an optimized docker image for the fastapi backend application \
located in ./backend directory, tag it as todo-backend:latest, \
use python slim base image, install requirements.txt dependencies"
```

**Gordon's Actions**:
- Analyzes Dockerfile or creates one with best practices (multi-stage build)
- Suggests optimizations (base image, layer caching, dependencies)
- Builds image with explanations

### Option B: Manual Docker Build (if Gordon unavailable)

```bash
# Frontend build
docker build -f docker/frontend/Dockerfile \
  -t todo-frontend:latest \
  ./frontend

# Backend build
docker build -f docker/backend/Dockerfile \
  -t todo-backend:latest \
  ./backend

# Verify images built
docker images | grep todo-
```

### Option C: Use Claude Code to Generate Dockerfiles

```bash
# Ask Claude Code:
# "Generate an optimized Dockerfile for our Next.js frontend with
#  multi-stage build, Alpine base, health check endpoint at /health"

# "Generate an optimized Dockerfile for our FastAPI backend with
#  Python 3.11-slim base, requirements.txt dependency installation,
#  health check that verifies database connectivity"

# Then build using option B above
```

**Expected Output**:
```
✓ frontend image: todo-frontend:latest (200-300MB)
✓ backend image: todo-backend:latest (200-250MB)
```

---

## Step 3: Load Images into Minikube (2 minutes)

```bash
# Option 1: Use Minikube built-in image loader (recommended)
minikube image load todo-frontend:latest
minikube image load todo-backend:latest

# Option 2: Use Docker Hub (if pushing images)
docker tag todo-frontend:latest docker.io/yourusername/todo-frontend:latest
docker push docker.io/yourusername/todo-frontend:latest
# Update helm values.yaml to reference your Docker Hub images

# Verify images available in Minikube
minikube image ls | grep todo-
```

---

## Step 4: Deploy with Helm (3 minutes)

```bash
# Update helm chart values with your database connection
# Edit helm/todo-chatbot/values.yaml:
# - Set secrets.databasePassword to your Neon password
# - Set secrets.openaiApiKey to your OpenAI API key

# Install Helm chart to Minikube
helm install todo-chatbot ./helm/todo-chatbot/ \
  --values ./helm/todo-chatbot/values.yaml \
  --set secrets.databasePassword="YOUR_DB_PASSWORD" \
  --set secrets.openaiApiKey="YOUR_OPENAI_API_KEY"

# Wait for deployment to complete
kubectl wait --for=condition=ready pod \
  -l app=todo-chatbot-frontend \
  --timeout=300s

kubectl wait --for=condition=ready pod \
  -l app=todo-chatbot-backend \
  --timeout=300s

# Check deployment status
kubectl get deployments
kubectl get pods
kubectl get services

# Output should show:
# DEPLOYMENT                       READY   UP-TO-DATE
# todo-chatbot-frontend           1/1     1
# todo-chatbot-backend            1/1     1
```

**Troubleshoot**:
- If pods are `Pending`: Check resource constraints (`kubectl describe node`)
- If pods are `ImagePullBackOff`: Verify images loaded via `minikube image ls`
- If pods are `CrashLoopBackOff`: Check logs via `kubectl logs <pod-name>`

---

## Step 5: Access Applications (2 minutes)

### Access Frontend

```bash
# Port-forward frontend service to localhost:3000
kubectl port-forward svc/todo-chatbot-frontend 3000:3000 &

# Open browser to http://localhost:3000
# Application should load; try adding a task via chatbot

# Or use kubectl-ai (if available)
kubectl-ai "port forward frontend service to localhost:3000"
```

### Access Backend Health Check

```bash
# Port-forward backend service to localhost:8000
kubectl port-forward svc/todo-chatbot-backend 8000:8000 &

# Test health endpoint
curl http://localhost:8000/health
# Expected: 200 OK with {"status": "healthy"} or similar

# Or use kubectl-ai
kubectl-ai "show me the backend logs"
```

---

## Step 6: Monitor Cluster Health (Optional)

### Using kubectl-ai

```bash
# Natural language Kubernetes commands
kubectl-ai "are all pods healthy and running"
kubectl-ai "show me the resource usage of the cluster"
kubectl-ai "list all services and their endpoints"
kubectl-ai "scale backend to 2 replicas"
kubectl-ai "check why the pods are not responding"
```

### Using Kagent

```bash
# Advanced cluster health analysis
kagent "analyze the cluster health"
# Output: Pod status, resource usage, performance bottlenecks

kagent "optimize resource allocation"
# Output: Recommendations for replicas, memory, CPU limits

kagent "what is the network topology"
# Output: Service connections, DNS resolution, latency
```

### Manual Inspection

```bash
# List all resources
kubectl get all

# Inspect specific pod
kubectl describe pod todo-chatbot-frontend-XXXXX

# Stream logs
kubectl logs -f deployment/todo-chatbot-backend

# Check events (errors, warnings)
kubectl get events
```

---

## Step 7: Testing & Verification

### Frontend Test

```bash
curl http://localhost:3000/
# Expected: HTML response (Next.js home page)

curl http://localhost:3000/health
# Expected: 200 OK
```

### Backend Test

```bash
curl http://localhost:8000/health
# Expected: 200 OK with database connectivity status

# If backend has /api/chat endpoint:
curl -X POST http://localhost:8000/api/chat \
  -H "Content-Type: application/json" \
  -d '{"user_id": 1, "message": "Add a task to buy groceries"}'
# Expected: 200 OK with chatbot response
```

### Chatbot Test (via Frontend)

1. Open http://localhost:3000 in browser
2. Log in to dashboard
3. Open chatbot widget
4. Type: "Add a task to buy groceries"
5. Chatbot should respond and task should appear in list
6. Type: "Show me all my tasks"
7. Chatbot should list created task

---

## Step 8: Scaling (Optional)

### Scale Backend to 3 Replicas

```bash
# Using kubectl-ai
kubectl-ai "scale backend deployment to 3 replicas"

# Or manually
kubectl scale deployment todo-chatbot-backend --replicas=3

# Verify
kubectl get pods -l app=todo-chatbot-backend
# Should show 3 pods running
```

### Scale Down

```bash
kubectl scale deployment todo-chatbot-backend --replicas=1
```

---

## Cleanup

```bash
# Delete Helm release (removes all pods, services, etc.)
helm uninstall todo-chatbot

# Stop Minikube cluster
minikube stop

# (Optional) Delete cluster completely
minikube delete
```

---

## Common Commands Reference

```bash
# Check pod status
kubectl get pods
kubectl describe pod <pod-name>

# View logs
kubectl logs <pod-name>
kubectl logs -f <pod-name>  # Stream logs

# Port forwarding
kubectl port-forward svc/<service-name> <local-port>:<service-port>

# Scaling
kubectl scale deployment <name> --replicas=3

# Apply updates
helm upgrade todo-chatbot ./helm/todo-chatbot/

# Rollback
helm rollback todo-chatbot

# Validate Helm chart
helm lint ./helm/todo-chatbot/
helm template ./helm/todo-chatbot/  # Preview manifests

# Minikube
minikube start
minikube stop
minikube logs
minikube dashboard  # Open Kubernetes dashboard
```

---

## Troubleshooting

### Pods Won't Start (Pending)

```bash
# Check resource constraints
kubectl describe node minikube

# Check pod events
kubectl describe pod <pod-name>

# Solution: Increase Minikube resources
minikube stop
minikube start --cpus=4 --memory=8192
```

### ImagePullBackOff Error

```bash
# Images not available in Minikube
# Solution: Reload images
minikube image load todo-frontend:latest
minikube image load todo-backend:latest
```

### Database Connection Failure

```bash
# Check logs for connection errors
kubectl logs deployment/todo-chatbot-backend | grep -i database

# Verify DATABASE_URL is correct in Secret
kubectl get secret todo-chatbot-secrets -o yaml | grep DATABASE_URL

# Solution: Update secret with correct connection string
kubectl delete secret todo-chatbot-secrets
helm upgrade todo-chatbot ./helm/todo-chatbot/ \
  --set secrets.databasePassword="CORRECT_PASSWORD"
```

### Pod Restart Loops (CrashLoopBackOff)

```bash
# Check detailed logs
kubectl logs <pod-name> --previous  # Last container logs before crash

# Check events
kubectl describe pod <pod-name>

# Common causes:
# - Missing environment variables
# - Database unreachable
# - Port already in use
# - Out of memory

# Solution: Fix issue in Dockerfile, rebuild, redeploy
```

### Services Not Accessible

```bash
# Verify service exists
kubectl get svc

# Port forward and test
kubectl port-forward svc/todo-chatbot-frontend 3000:3000
curl http://localhost:3000  # Should work

# If service not accessible from pod (inter-service):
# Verify DNS: kubectl run -it debug --image=alpine --restart=Never \
#   -- nslookup todo-chatbot-backend
```

---

## Success Criteria

✅ Minikube cluster running (`minikube status` = Running)
✅ Both frontend and backend pods in `Running` state
✅ Both pods passing health checks (green status)
✅ Services created and accessible
✅ Frontend loads at http://localhost:3000
✅ Backend health check responds at http://localhost:8000/health
✅ Can create tasks via chatbot
✅ Can view tasks in dashboard
✅ Can scale backend replicas with kubectl

---

## Next Steps

1. **Monitoring**: Use Kagent to analyze cluster health
2. **Logging**: View pod logs with `kubectl logs` or kubectl-ai
3. **Scaling**: Test scaling backend to multiple replicas
4. **Troubleshooting**: Use kubectl-ai to diagnose issues
5. **Production**: Adapt Helm values for cloud deployment (AWS, GCP, Azure)

---

For detailed documentation, see:
- [DEPLOYMENT.md](./DEPLOYMENT.md) - Comprehensive deployment guide
- [KUBECTL-AI.md](./KUBECTL-AI.md) - kubectl-ai command examples
- [KAGENT.md](./KAGENT.md) - Cluster health monitoring
- [GORDON.md](./GORDON.md) - Docker AI integration
- [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) - Common issues
