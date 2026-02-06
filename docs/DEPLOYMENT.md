# Deployment Guide: Cloud Native Todo Chatbot on Minikube

**Time to Complete**: 15-20 minutes
**Prerequisites**: Docker Desktop, kubectl, helm, minikube CLI, Chocolatey (optional)

This guide walks you through deploying the Todo Chatbot to a local Minikube Kubernetes cluster.

---

## Table of Contents

1. [Prerequisites Checklist](#prerequisites-checklist)
2. [Step 1: Start Minikube Cluster](#step-1-start-minikube-cluster)
3. [Step 2: Build Docker Images](#step-2-build-docker-images)
4. [Step 3: Load Images into Minikube](#step-3-load-images-into-minikube)
5. [Step 4: Deploy with Helm](#step-4-deploy-with-helm)
6. [Step 5: Access Applications](#step-5-access-applications)
7. [Step 6: Verify Deployment](#step-6-verify-deployment)
8. [Step 7: Troubleshooting](#step-7-troubleshooting)

---

## Prerequisites Checklist

Before starting, ensure you have installed:

- [ ] **Docker Desktop** 4.53+ with Kubernetes support enabled
- [ ] **kubectl** CLI - Verify: `kubectl version --client`
- [ ] **helm** 3+ CLI - Verify: `helm version`
- [ ] **minikube** CLI - Verify: `minikube version`
- [ ] **Phase III Todo Chatbot** source code (frontend + backend directories)
- [ ] **Neon PostgreSQL** connection string (DATABASE_URL)
- [ ] **OpenAI API Key** (OPENAI_API_KEY)

### Install via Chocolatey (Windows)

If using Chocolatey, you can install tools with:

```bash
choco install minikube
choco install kubernetes-cli
choco install helm
```

---

## Step 1: Start Minikube Cluster (5 minutes)

### Option A: Docker Driver (Recommended)

```bash
minikube start --driver=docker --cpus=4 --memory=8192 --disk-size=20gb
```

### Option B: Hyper-V Driver (Windows Pro/Enterprise)

```bash
# Requires Administrator PowerShell
minikube start --driver=hyperv --cpus=4 --memory=8192 --disk-size=20gb
```

### Verify Cluster is Running

```bash
kubectl cluster-info
kubectl get nodes
```

**Expected output:**
```
NAME       STATUS   ROLES                  AGE   VERSION
minikube   Ready    control-plane,master   2m    v1.XX.X
```

---

## Step 2: Build Docker Images (5 minutes)

### Option A: Using Gordon (Docker AI) - Recommended

If you have Docker Desktop 4.53+ with Gordon enabled:

```bash
# Enable Gordon in Docker Desktop Settings > Beta features first

# Build frontend image with Gordon
docker ai "build an optimized docker image for the next.js frontend \
located in ./frontend directory, use node:18-alpine base, \
include health check at /health, tag as todo-frontend:latest"

# Build backend image with Gordon
docker ai "build an optimized docker image for the fastapi backend \
located in ./backend directory, use python:3.11-slim base, \
include health check at /health with database connectivity, \
tag as todo-backend:latest"
```

### Option B: Manual Docker Build

```bash
# Build frontend
docker build -f docker/frontend/Dockerfile \
  -t todo-frontend:latest .

# Build backend
docker build -f docker/backend/Dockerfile \
  -t todo-backend:latest .

# Verify images
docker images | grep todo-
```

**Expected output:**
```
REPOSITORY         TAG       IMAGE ID            SIZE
todo-frontend      latest    abcd1234efgh        250MB
todo-backend       latest    ijkl5678mnop        220MB
```

### Verify Image Sizes

Confirm images are under size targets:
- Frontend: < 500MB ✓
- Backend: < 300MB ✓

---

## Step 3: Load Images into Minikube (2 minutes)

### Load Images

```bash
# Option 1: Use Minikube built-in loader (Recommended)
minikube image load todo-frontend:latest
minikube image load todo-backend:latest

# Option 2: Use Docker Hub (if pushing to cloud registry)
docker tag todo-frontend:latest docker.io/yourusername/todo-frontend:latest
docker tag todo-backend:latest docker.io/yourusername/todo-backend:latest
docker push docker.io/yourusername/todo-frontend:latest
docker push docker.io/yourusername/todo-backend:latest
# Then update helm/todo-chatbot/values.yaml to reference your Docker Hub images
```

### Verify Images in Minikube

```bash
minikube image ls | grep todo-
```

**Expected output:**
```
docker.io/library/todo-frontend:latest
docker.io/library/todo-backend:latest
```

---

## Step 4: Deploy with Helm (3 minutes)

### Create Helm Values Override

Before deploying, you need to provide database credentials. Create a file or use --set flags.

### Deploy Application

```bash
helm install todo-chatbot ./helm/todo-chatbot/ \
  --set secrets.databasePassword="YOUR_NEON_PASSWORD" \
  --set secrets.openaiApiKey="YOUR_OPENAI_API_KEY" \
  --values ./helm/todo-chatbot/values.yaml
```

### Wait for Deployment

```bash
# Wait for frontend pod to be ready (max 5 minutes)
kubectl wait --for=condition=ready pod \
  -l app=todo-chatbot-frontend \
  --timeout=300s

# Wait for backend pod to be ready
kubectl wait --for=condition=ready pod \
  -l app=todo-chatbot-backend \
  --timeout=300s
```

### Check Deployment Status

```bash
kubectl get deployments
kubectl get pods
kubectl get services

# Expected output:
# NAME                       READY   UP-TO-DATE
# todo-chatbot-frontend      1/1     1
# todo-chatbot-backend       1/1     1
```

---

## Step 5: Access Applications (2 minutes)

### Port-Forward Frontend

```bash
kubectl port-forward svc/todo-chatbot-frontend 3000:3000 &
```

**Access in browser:** http://localhost:3000

### Port-Forward Backend

```bash
kubectl port-forward svc/todo-chatbot-backend 8000:8000 &
```

**Test health endpoint:**
```bash
curl http://localhost:8000/health
```

**Expected response:**
```json
{"status": "healthy", "database": "connected"}
```

---

## Step 6: Verify Deployment (3 minutes)

### Test Frontend

1. Open http://localhost:3000 in your browser
2. Dashboard should load without errors
3. Create a task via chatbot widget:
   - Type: "Add a task to buy groceries"
   - Chatbot should respond
   - Task should appear in list
4. Type: "Show me all my tasks"
   - Chatbot should list created task

### Test Backend Connectivity

```bash
# Check backend logs
kubectl logs deployment/todo-chatbot-backend

# Verify database connection
curl -s http://localhost:8000/health | jq .

# Test API endpoint (if available)
curl -X GET http://localhost:8000/api/tasks
```

### Check Pod Health

```bash
# List all pods
kubectl get pods

# Check pod details
kubectl describe pod <frontend-pod-name>
kubectl describe pod <backend-pod-name>

# Verify no restart loops
# Look for Restart count: 0
```

---

## Step 7: Troubleshooting

### Pods Won't Start (Pending State)

```bash
# Check resource constraints
kubectl describe node minikube

# Check pod events for details
kubectl describe pod <pod-name>

# Solution: Increase Minikube resources
minikube stop
minikube start --driver=docker --cpus=4 --memory=8192 --disk-size=20gb
```

### ImagePullBackOff Error

```bash
# Verify images are loaded in Minikube
minikube image ls | grep todo-

# Solution: Reload images
minikube image load todo-frontend:latest
minikube image load todo-backend:latest
```

### Database Connection Failure

```bash
# Check backend logs
kubectl logs deployment/todo-chatbot-backend | grep -i database

# Verify DATABASE_URL in Secret
kubectl get secret todo-chatbot-secrets -o yaml | grep DATABASE

# Solution: Update secret
helm upgrade todo-chatbot ./helm/todo-chatbot/ \
  --set secrets.databasePassword="CORRECT_PASSWORD"
```

### Pod Restart Loops (CrashLoopBackOff)

```bash
# Check detailed logs
kubectl logs <pod-name> --previous

# Check pod events
kubectl describe pod <pod-name>

# Solution: Fix Dockerfile or environment variables, rebuild image
```

---

## Next Steps

1. **Monitor Cluster**: See [KUBECTL-AI.md](./KUBECTL-AI.md) for monitoring commands
2. **Optimize Resources**: See [KAGENT.md](./KAGENT.md) for optimization
3. **Docker Operations**: See [GORDON.md](./GORDON.md) for Docker AI integration
4. **Scaling**: Scale backend replicas:
   ```bash
   kubectl scale deployment todo-chatbot-backend --replicas=3
   ```
5. **Cleanup**: When done:
   ```bash
   helm uninstall todo-chatbot
   minikube stop
   ```

---

## Common Commands Reference

```bash
# Cluster status
kubectl cluster-info
kubectl get nodes
minikube status

# Pods and deployments
kubectl get pods
kubectl get deployments
kubectl describe pod <pod-name>

# Logs
kubectl logs <pod-name>
kubectl logs -f <pod-name>        # Stream logs
kubectl logs deployment/todo-chatbot-backend

# Port forwarding
kubectl port-forward svc/todo-chatbot-frontend 3000:3000
kubectl port-forward svc/todo-chatbot-backend 8000:8000

# Scaling
kubectl scale deployment todo-chatbot-backend --replicas=3

# Helm
helm install todo-chatbot ./helm/todo-chatbot/
helm upgrade todo-chatbot ./helm/todo-chatbot/
helm rollback todo-chatbot
helm uninstall todo-chatbot

# Minikube
minikube start
minikube stop
minikube delete
minikube logs
minikube dashboard
```

---

**For detailed troubleshooting**: See [TROUBLESHOOTING.md](./TROUBLESHOOTING.md)
