# Troubleshooting Guide

**Quick Fix Index**: Jump to your issue below for specific solutions.

---

## Table of Contents

1. [Pod Issues](#pod-issues)
2. [Service & Networking](#service--networking)
3. [Image Issues](#image-issues)
4. [Database Connection](#database-connection)
5. [Resource Issues](#resource-issues)
6. [Helm/Deployment Issues](#helmdeployment-issues)
7. [Kubernetes Cluster Issues](#kubernetes-cluster-issues)

---

## Pod Issues

### Pods Won't Start (Pending State)

**Symptoms:**
- Pods stuck in "Pending" state
- `kubectl get pods` shows STATUS: Pending
- Pod hasn't moved to Running after 5+ minutes

**Diagnosis:**
```bash
# Check pod details
kubectl describe pod <pod-name>

# Look for:
# - Events section showing why it's pending
# - ResourceQuota exceeded?
# - PersistentVolume not found?
```

**Common Causes & Solutions:**

1. **Insufficient Resources**
   ```bash
   # Check node resources
   kubectl describe node minikube

   # Look for: "MemoryPressure", "DiskPressure"

   # Solution: Increase Minikube resources
   minikube stop
   minikube start --driver=docker --cpus=4 --memory=8192 --disk-size=20gb
   ```

2. **Image Pull Backoff**
   - See [ImagePullBackOff](#imagepullbackoff-error) below

3. **Container Port Already in Use**
   ```bash
   # Check port conflicts
   netstat -an | grep 3000
   netstat -an | grep 8000

   # Kill process using port
   lsof -ti:3000 | xargs kill -9
   ```

---

### CrashLoopBackOff (Pod Restart Loops)

**Symptoms:**
- Pod shows STATUS: CrashLoopBackOff
- Restart count keeps incrementing
- Application exits immediately after starting

**Diagnosis:**
```bash
# Check crash logs
kubectl logs <pod-name> --previous

# Check pod events
kubectl describe pod <pod-name>

# Real-time monitoring
kubectl get pod <pod-name> --watch
```

**Common Causes & Solutions:**

1. **Missing Environment Variables**
   ```bash
   # Check what env vars are set
   kubectl exec -it <pod-name> -- env | grep -i database

   # Solution: Verify Secret and ConfigMap
   kubectl get configmap
   kubectl get secret
   kubectl get secret <secret-name> -o yaml
   ```

2. **Database Connection Failed**
   ```bash
   # Check backend logs
   kubectl logs deployment/todo-chatbot-backend | grep -i database

   # Solution:
   # 1. Verify DATABASE_URL is correct
   # 2. Check Neon database is accessible
   # 3. Verify password is correct
   ```

3. **Application Error on Startup**
   ```bash
   # View full logs
   kubectl logs <pod-name>

   # Check for:
   # - Module not found (missing dependencies)
   # - Syntax errors
   # - Configuration errors

   # Solution: Fix issue in Dockerfile, rebuild image
   ```

4. **Port Already in Use**
   ```bash
   # Check port binding
   kubectl logs <pod-name>

   # Look for: "Address already in use" or "Permission denied"

   # Solution: Check if another pod uses same port
   ```

---

### ImagePullBackOff Error

**Symptoms:**
- Pod shows STATUS: ImagePullBackOff
- Error: `Failed to pull image "..."`

**Diagnosis:**
```bash
# Check pod events
kubectl describe pod <pod-name>

# Look for events section showing:
# - ErrImagePull: Failed to pull image
# - ImagePullBackOff: Retry after backoff
```

**Common Causes & Solutions:**

1. **Image Not in Minikube Registry**
   ```bash
   # Check what images are available
   minikube image ls | grep todo-

   # Solution: Load images into Minikube
   minikube image load todo-frontend:latest
   minikube image load todo-backend:latest

   # Verify
   minikube image ls | grep todo-
   ```

2. **Wrong Image Name in Helm Values**
   ```bash
   # Check what image is specified
   helm get values todo-chatbot

   # Look at: frontend.image.repository, backend.image.repository

   # Solution: Update values.yaml with correct image names
   helm upgrade todo-chatbot ./helm/todo-chatbot/ \
     --set frontend.image.repository=localhost:5000/todo-frontend
   ```

3. **Using Docker Hub Image Not Built Yet**
   ```bash
   # If using docker.io/username/...
   # Verify image is pushed to Docker Hub
   docker images | grep <image-name>

   # Push to Docker Hub
   docker push <image-name>:latest
   ```

---

### Pod Health Check Failures

**Symptoms:**
- Pod shows STATUS: Running but ReadinessProbe failing
- Pod keeps restarting due to LivenessProbe failing

**Diagnosis:**
```bash
# Check probe status in describe
kubectl describe pod <pod-name>

# Look for: Readiness Probe, Liveness Probe sections
# Check: "failed liveness probe"

# Test probe manually
kubectl port-forward <pod-name> 8000:8000
curl http://localhost:8000/health
```

**Solutions:**

1. **Health Endpoint Not Implemented**
   ```bash
   # Check if endpoint responds
   kubectl exec -it <pod-name> -- curl http://localhost:8000/health

   # Solution: Implement health endpoint in application
   # - FastAPI: Add @app.get("/health") endpoint
   # - Next.js: Add /health route
   ```

2. **Database Not Reachable from Health Check**
   ```bash
   # Backend health check includes DB connectivity
   # Solution: Fix database connection
   # - Verify DATABASE_URL
   # - Check database is accessible
   # - Verify credentials
   ```

3. **Probe Timeout Too Short**
   ```bash
   # Check probe config
   kubectl describe pod <pod-name> | grep -A 5 "Probe"

   # Solution: Increase timeout in values.yaml
   readinessProbe:
     initialDelaySeconds: 20  # Increase from 10
     timeoutSeconds: 5        # Increase from 3
   ```

---

## Service & Networking

### Services Not Accessible

**Symptoms:**
- Cannot connect to `http://localhost:3000` or `http://localhost:8000`
- `curl: (7) Failed to connect to localhost:3000`

**Diagnosis:**
```bash
# Check if service exists
kubectl get services

# Check service details
kubectl describe service todo-chatbot-frontend

# Check if port-forward is running
ps aux | grep "port-forward"

# Test with port-forward
kubectl port-forward svc/todo-chatbot-frontend 3000:3000
# Then try: curl http://localhost:3000
```

**Solutions:**

1. **Port-Forward Not Running**
   ```bash
   # Kill existing port-forwards
   pkill -f "port-forward"

   # Start port-forward in foreground to debug
   kubectl port-forward svc/todo-chatbot-frontend 3000:3000

   # In another terminal, test
   curl http://localhost:3000/health
   ```

2. **Wrong Port Number**
   ```bash
   # Check what port service is using
   kubectl get service todo-chatbot-frontend -o yaml | grep port:

   # Use correct port in port-forward
   kubectl port-forward svc/todo-chatbot-frontend 3000:3000
   ```

3. **Service LoadBalancer Not Getting IP (Minikube)**
   ```bash
   # In Minikube, LoadBalancer services need tunneling
   minikube tunnel

   # Then check service IP
   kubectl get service todo-chatbot-frontend
   ```

---

### Inter-Service Communication Failure

**Symptoms:**
- Frontend pod can't reach backend
- Logs show: "Failed to resolve backend:8000"

**Diagnosis:**
```bash
# Check if backend service exists
kubectl get service todo-chatbot-backend

# Test DNS from frontend pod
kubectl exec -it <frontend-pod> -- nslookup todo-chatbot-backend

# Test TCP connection
kubectl exec -it <frontend-pod> -- telnet todo-chatbot-backend 8000

# Check service endpoints
kubectl get endpoints todo-chatbot-backend
```

**Solutions:**

1. **Service Not Found (DNS Issue)**
   ```bash
   # Check service name
   kubectl get services | grep backend

   # Solution: Use full DNS name if needed
   # Short: backend:8000
   # Full: backend.default.svc.cluster.local:8000
   ```

2. **Service Has No Endpoints**
   ```bash
   # Check endpoints
   kubectl get endpoints todo-chatbot-backend
   # Should show IP addresses of backend pods

   # Solution: Fix pod selection
   # Verify pod labels match service selector
   kubectl get pods -L app
   kubectl describe service todo-chatbot-backend
   ```

3. **Firewall/Network Policy**
   ```bash
   # Check network policies (unlikely in MVP)
   kubectl get networkpolicies

   # Solution: Remove network policies or add rules allowing traffic
   ```

---

## Image Issues

### Image Too Large

**Symptoms:**
- Frontend image > 500MB
- Backend image > 300MB
- Slow image pull times

**Diagnosis:**
```bash
# Check image size
docker images todo-frontend:latest
docker images todo-backend:latest

# Analyze layers
docker history todo-frontend:latest --human

# Find large layers
docker history todo-frontend:latest --human --no-trunc | grep -v "sha256"
```

**Solutions:**

1. **Using Full Node.js Debian Image**
   ```bash
   # Current: node:18 (900MB+)
   # Better: node:18-alpine (150MB)

   # Update Dockerfile
   FROM node:18-alpine AS builder  # Instead of node:18
   ```

2. **Node Modules Included in Final Image**
   ```bash
   # Problem: Multi-stage not used
   FROM node:18
   COPY . .
   RUN npm install
   # Result: node_modules in final image (500MB+)

   # Solution: Use multi-stage build
   FROM node:18-alpine AS builder
   COPY . .
   RUN npm install  # Large layer

   FROM node:18-alpine  # Fresh stage
   COPY --from=builder /app/.next .next
   COPY --from=builder /app/node_modules node_modules
   # Result: Only needed files in final image
   ```

3. **Package Manager Cache Not Cleaned**
   ```bash
   # Problem
   RUN npm install    # Leaves cache

   # Solution
   RUN npm ci --only=production && npm cache clean --force

   # Or for pip
   RUN pip install --no-cache-dir -r requirements.txt
   ```

---

### Build Failures

**Symptoms:**
- `docker build` fails with errors
- "RUN command returned non-zero exit status"

**Diagnosis:**
```bash
# Run build with verbose output
docker build -f docker/backend/Dockerfile -t todo-backend:test . --progress=plain

# Check specific layer that failed
# Look at error message in output
```

**Solutions:**

1. **Missing Dependencies**
   ```bash
   # Error: "ModuleNotFoundError" or "apt-get: command not found"

   # Solution: Install in Dockerfile
   # For Node: RUN npm install
   # For Python: RUN pip install -r requirements.txt
   ```

2. **Requirements.txt Not Found**
   ```bash
   # Error: "COPY backend/requirements.txt: no such file or directory"

   # Solution: Check path and ensure file exists
   ls -la backend/requirements.txt

   # Update Dockerfile COPY path if needed
   ```

3. **Build Context Issues**
   ```bash
   # Error: "COPY frontend/package.json: not found in context"

   # Solution: Run docker build from repo root
   cd /path/to/repo
   docker build -f docker/frontend/Dockerfile -t todo-frontend .
   ```

---

## Database Connection

### Database Connection Timeout

**Symptoms:**
- Backend pod stuck in CrashLoopBackOff
- Logs show: "timeout waiting for database"
- Health check failing: "Database connection failed"

**Diagnosis:**
```bash
# Check backend logs
kubectl logs deployment/todo-chatbot-backend | tail -30

# Check if DATABASE_URL is set
kubectl describe pod <backend-pod> | grep DATABASE_URL

# Test connection from pod
kubectl exec -it <backend-pod> -- psql $DATABASE_URL -c "SELECT 1"
```

**Solutions:**

1. **Wrong DATABASE_URL**
   ```bash
   # Check current value
   kubectl get secret todo-chatbot-secrets -o yaml | grep DATABASE_URL

   # Should be: postgresql://user:pass@host:port/dbname

   # Solution: Update secret
   kubectl delete secret todo-chatbot-secrets
   helm upgrade todo-chatbot ./helm/todo-chatbot/ \
     --set secrets.databasePassword="CORRECT_PASSWORD"
   ```

2. **Database Not Accessible**
   ```bash
   # Check if Neon database is accessible
   # Try from your machine first
   psql postgresql://user:pass@neon-host/dbname

   # If fails: Contact Neon support or verify connection string

   # If succeeds from machine but not pod:
   # - Check if pod has network access to Neon
   # - Verify no firewall blocking
   ```

3. **Wrong Credentials**
   ```bash
   # Verify credentials in Neon dashboard
   # Update secret with correct values
   kubectl set env deployment/todo-chatbot-backend \
     DATABASE_PASSWORD="correct_password"
   ```

---

## Resource Issues

### Out of Memory

**Symptoms:**
- Pod OOMKilled
- `kubectl describe pod` shows "OOMKilled"
- STATUS: Error

**Diagnosis:**
```bash
# Check pod events
kubectl describe pod <pod-name> | grep -i "out of memory\|oom"

# Check node resources
kubectl describe node minikube

# Check resource limits
kubectl describe pod <pod-name> | grep -A 5 "Limits"
```

**Solutions:**

1. **Increase Memory Limit**
   ```bash
   # Update values.yaml
   backend:
     resources:
       limits:
         memory: 1Gi  # Increase from current value

   # Apply change
   helm upgrade todo-chatbot ./helm/todo-chatbot/
   ```

2. **Increase Minikube Memory**
   ```bash
   minikube stop
   minikube start --memory=8192 --cpus=4
   ```

---

### CPU Throttling

**Symptoms:**
- Application running slowly
- `kubectl top pods` shows high CPU usage
- Requests taking longer than expected

**Diagnosis:**
```bash
# Check CPU usage
kubectl top pods
kubectl top nodes

# Check resource requests
kubectl describe pod <pod-name> | grep -A 5 "Requests"
```

**Solutions:**

1. **Increase CPU Limit**
   ```bash
   # Update values.yaml
   backend:
     resources:
       limits:
         cpu: 1000m  # Increase from current value

   # Apply change
   helm upgrade todo-chatbot ./helm/todo-chatbot/
   ```

2. **Increase Minikube CPU**
   ```bash
   minikube stop
   minikube start --cpus=4 --memory=8192
   ```

---

## Helm/Deployment Issues

### Helm Install Fails

**Symptoms:**
- `helm install` returns error
- Chart validation fails

**Diagnosis:**
```bash
# Validate chart
helm lint ./helm/todo-chatbot/

# Check template rendering
helm template todo-chatbot ./helm/todo-chatbot/

# Dry-run to see what would happen
helm install todo-chatbot ./helm/todo-chatbot/ --dry-run --debug
```

**Solutions:**

1. **Chart.yaml Issues**
   ```bash
   # Check format
   cat helm/todo-chatbot/Chart.yaml

   # Verify YAML syntax
   # apiVersion, name, version should be present
   ```

2. **Template Errors**
   ```bash
   # Check template syntax
   helm template todo-chatbot ./helm/todo-chatbot/

   # Look for error messages
   # Fix syntax and retry
   ```

3. **Missing Required Values**
   ```bash
   # Check what values are required
   helm template todo-chatbot ./helm/todo-chatbot/ \
     --show-only templates/deployment-backend.yaml

   # Verify values.yaml has all needed keys
   ```

---

### Helm Upgrade Fails

**Symptoms:**
- `helm upgrade` fails
- Deployment doesn't update

**Diagnosis:**
```bash
# Check helm status
helm status todo-chatbot

# Check release history
helm history todo-chatbot

# Check what's deployed
kubectl get all -l app=todo-chatbot
```

**Solutions:**

1. **Revert to Previous Release**
   ```bash
   # Rollback to previous working version
   helm rollback todo-chatbot

   # Verify
   kubectl get pods
   ```

2. **Delete and Reinstall**
   ```bash
   # If stuck in bad state
   helm uninstall todo-chatbot

   # Wait for pods to terminate
   kubectl get pods --watch

   # Reinstall
   helm install todo-chatbot ./helm/todo-chatbot/
   ```

---

## Kubernetes Cluster Issues

### Minikube Not Starting

**Symptoms:**
- `minikube start` fails
- "Error creating machine"

**Diagnosis:**
```bash
# Check Minikube status
minikube status

# Check logs
minikube logs

# Check Docker is running
docker ps
```

**Solutions:**

1. **Docker Desktop Not Running**
   ```bash
   # Start Docker Desktop application
   # Wait for it to fully initialize
   # Then: minikube start --driver=docker
   ```

2. **Hyper-V Permission Issues**
   ```bash
   # If using Hyper-V driver
   # Run PowerShell as Administrator
   # Then: minikube start --driver=hyperv
   ```

3. **Delete and Recreate Cluster**
   ```bash
   minikube delete
   minikube start --driver=docker --cpus=4 --memory=8192
   ```

---

### kubectl Not Connecting to Cluster

**Symptoms:**
- `kubectl cluster-info` fails
- "Unable to connect to the server"

**Diagnosis:**
```bash
# Check cluster context
kubectl config current-context

# Check available contexts
kubectl config get-contexts

# Check cluster-info
kubectl cluster-info
```

**Solutions:**

1. **Wrong Context**
   ```bash
   # Switch to Minikube context
   kubectl config use-context minikube

   # Verify
   kubectl cluster-info
   ```

2. **Minikube Not Running**
   ```bash
   # Start Minikube
   minikube start

   # Configure kubectl
   minikube update-context

   # Verify
   kubectl cluster-info
   ```

---

## Getting Help

### Enable Verbose Logging

```bash
# Get detailed error information
kubectl describe pod <pod-name>
kubectl logs <pod-name> -f
kubectl events

# For Helm
helm install ... --debug --dry-run

# For Docker
docker build --progress=plain -f Dockerfile .
```

### Collect Diagnostics

```bash
# Cluster info
kubectl cluster-info dump > cluster-info.txt

# Pod logs
kubectl logs deployment/todo-chatbot-frontend > frontend-logs.txt
kubectl logs deployment/todo-chatbot-backend > backend-logs.txt

# All resources
kubectl get all > resources.txt
kubectl describe all >> resources.txt
```

### Reach Out

- Check [DEPLOYMENT.md](./DEPLOYMENT.md) for setup instructions
- Review [KUBECTL-AI.md](./KUBECTL-AI.md) for cluster commands
- See [GORDON.md](./GORDON.md) for Docker issues
- Check Kubernetes docs: https://kubernetes.io/docs/
- Minikube docs: https://minikube.sigs.k8s.io/
