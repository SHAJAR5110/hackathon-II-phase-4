# kubectl-ai Integration Guide

**Overview**: Use natural language commands with `kubectl-ai` to manage your Kubernetes cluster without writing complex kubectl commands.

---

## Installation & Setup

### Install kubectl-ai

```bash
# Using npm
npm install -g @isovalent/kubectl-ai

# Using Homebrew (macOS)
brew install kubectl-ai

# Manual download
# Visit: https://github.com/kubernetes/kubectl-ai
```

### Verify Installation

```bash
kubectl-ai --version
which kubectl-ai
```

---

## Common Deployment Commands

### Deploy Application

```bash
# Deploy using Helm
kubectl-ai "deploy todo chatbot using helm chart"

# Equivalent kubectl command:
helm install todo-chatbot ./helm/todo-chatbot/

# Deploy with custom values
kubectl-ai "deploy todo chatbot with 2 frontend replicas and 3 backend replicas"

# Equivalent:
helm install todo-chatbot ./helm/todo-chatbot/ \
  --set frontend.replicaCount=2 \
  --set backend.replicaCount=3
```

### Check Pod Status

```bash
# List all running pods
kubectl-ai "list all running pods"

# Equivalent:
kubectl get pods

# Show pods with more details
kubectl-ai "show me all pods with their status and resource usage"

# Equivalent:
kubectl get pods -o wide

# Show specific deployment pods
kubectl-ai "show all pods for the todo-chatbot frontend"

# Equivalent:
kubectl get pods -l app=todo-chatbot-frontend
```

### Check Deployment Status

```bash
# Check if deployments are ready
kubectl-ai "are all pods healthy and running"

# Equivalent:
kubectl rollout status deployment/todo-chatbot-frontend
kubectl rollout status deployment/todo-chatbot-backend

# Show deployment details
kubectl-ai "describe the todo-chatbot frontend deployment"

# Equivalent:
kubectl describe deployment todo-chatbot-frontend
```

---

## Scaling Commands

### Scale Replicas

```bash
# Scale backend to 3 replicas
kubectl-ai "scale backend to 3 replicas"

# Equivalent:
kubectl scale deployment todo-chatbot-backend --replicas=3

# Scale frontend to 2 replicas
kubectl-ai "scale frontend deployment to 2 replicas"

# Equivalent:
kubectl scale deployment todo-chatbot-frontend --replicas=2

# Scale both deployments
kubectl-ai "scale backend to 5 and frontend to 2 replicas"

# Equivalent:
kubectl scale deployment todo-chatbot-backend --replicas=5
kubectl scale deployment todo-chatbot-frontend --replicas=2
```

---

## Log Commands

### View Logs

```bash
# Show backend logs
kubectl-ai "show me the backend logs"

# Equivalent:
kubectl logs -l app=todo-chatbot-backend

# Stream logs in real-time
kubectl-ai "stream the backend deployment logs"

# Equivalent:
kubectl logs -f deployment/todo-chatbot-backend

# Show logs from specific pod
kubectl-ai "show logs from frontend pod"

# Equivalent:
kubectl logs <frontend-pod-name>

# Show logs from previous container (if crashed)
kubectl-ai "show me the logs from the crashed backend container"

# Equivalent:
kubectl logs <pod-name> --previous
```

### Filter Logs

```bash
# Show error logs
kubectl-ai "show error logs from backend"

# Equivalent:
kubectl logs deployment/todo-chatbot-backend | grep -i error

# Show database connection logs
kubectl-ai "show database connection logs from backend"

# Equivalent:
kubectl logs deployment/todo-chatbot-backend | grep -i database
```

---

## Port Forwarding Commands

### Access Services Locally

```bash
# Port-forward frontend to localhost:3000
kubectl-ai "port forward frontend service to localhost:3000"

# Equivalent:
kubectl port-forward svc/todo-chatbot-frontend 3000:3000

# Port-forward backend to localhost:8000
kubectl-ai "port forward backend service to localhost:8000"

# Equivalent:
kubectl port-forward svc/todo-chatbot-backend 8000:8000

# Forward multiple ports
kubectl-ai "port forward frontend to 3000 and backend to 8000"

# Then access in browser/curl:
# Frontend: http://localhost:3000
# Backend health: http://localhost:8000/health
```

---

## Troubleshooting Commands

### Diagnose Issues

```bash
# Check why pods are failing
kubectl-ai "why are the pods not starting"

# Equivalent:
kubectl describe pod <pod-name>
kubectl get events
kubectl logs <pod-name>

# Check resource constraints
kubectl-ai "check why pods are pending"

# Equivalent:
kubectl describe node minikube

# Check service connectivity
kubectl-ai "can the frontend pod reach the backend service"

# Equivalent:
kubectl exec -it <frontend-pod> -- nslookup todo-chatbot-backend

# Check if services exist
kubectl-ai "list all services"

# Equivalent:
kubectl get services
```

### Container Issues

```bash
# Check why container is failing to start
kubectl-ai "why is the backend container failing"

# Equivalent:
kubectl logs <pod-name>
kubectl describe pod <pod-name>

# Check resource limits
kubectl-ai "show me the resource limits for backend pods"

# Equivalent:
kubectl describe pod <backend-pod> | grep -A 5 "Limits"

# Check health check status
kubectl-ai "check the health status of all pods"

# Equivalent:
kubectl describe pod -l app=todo-chatbot
```

---

## Service Commands

### Manage Services

```bash
# List all services
kubectl-ai "list all services"

# Equivalent:
kubectl get services

# Show service details
kubectl-ai "describe the todo-chatbot frontend service"

# Equivalent:
kubectl describe service todo-chatbot-frontend

# Check service endpoints
kubectl-ai "show me the endpoints for the backend service"

# Equivalent:
kubectl get endpoints todo-chatbot-backend
```

---

## Configuration Commands

### Manage ConfigMaps and Secrets

```bash
# List all config maps
kubectl-ai "list all config maps"

# Equivalent:
kubectl get configmaps

# Show config values
kubectl-ai "show me the configuration for todo-chatbot"

# Equivalent:
kubectl get configmap todo-chatbot-config -o yaml

# List secrets
kubectl-ai "list all secrets"

# Equivalent:
kubectl get secrets
```

---

## Helm Commands via kubectl-ai

### Helm Operations

```bash
# Check Helm release status
kubectl-ai "show me the helm release status"

# Equivalent:
helm status todo-chatbot

# Show Helm values
kubectl-ai "show me the helm values for todo-chatbot"

# Equivalent:
helm get values todo-chatbot

# Upgrade deployment
kubectl-ai "upgrade todo-chatbot to use 3 replicas"

# Equivalent:
helm upgrade todo-chatbot ./helm/todo-chatbot/ \
  --set backend.replicaCount=3

# Rollback to previous release
kubectl-ai "rollback the todo-chatbot deployment"

# Equivalent:
helm rollback todo-chatbot
```

---

## Resource Monitoring

### Check Resource Usage

```bash
# Check CPU and memory usage
kubectl-ai "what is the cpu and memory usage"

# Equivalent:
kubectl top nodes
kubectl top pods

# Show node resource allocation
kubectl-ai "show me the resource allocation"

# Equivalent:
kubectl describe node minikube

# Check pod resource requests and limits
kubectl-ai "show me the resource requests and limits"

# Equivalent:
kubectl describe pod <pod-name> | grep -A 5 "Requests\|Limits"
```

---

## Advanced Troubleshooting

### Deep Diagnostics

```bash
# Get all events in cluster
kubectl-ai "show me all cluster events"

# Equivalent:
kubectl get events

# Check pod restart count
kubectl-ai "show me pods that have restarted"

# Equivalent:
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.containerStatuses[*].restartCount}{"\n"}{end}'

# Test DNS resolution in cluster
kubectl-ai "test if frontend pod can resolve backend service name"

# Equivalent:
kubectl exec -it <frontend-pod> -- nslookup todo-chatbot-backend

# Check pod details
kubectl-ai "show me detailed information about the frontend pod"

# Equivalent:
kubectl describe pod <frontend-pod-name>
```

---

## Fallback to Manual kubectl

If kubectl-ai is unavailable or misinterprets your command, use these manual equivalents:

```bash
# Check all resources
kubectl get all

# Inspect specific resource
kubectl describe deployment todo-chatbot-frontend
kubectl describe service todo-chatbot-backend
kubectl describe pod <pod-name>

# View resource definitions
kubectl get deployment todo-chatbot-frontend -o yaml
kubectl get service todo-chatbot-backend -o json

# Apply changes
kubectl apply -f <manifest-file>
kubectl patch deployment todo-chatbot-frontend -p '{"spec":{"replicas":2}}'

# Delete resources
kubectl delete pod <pod-name>
kubectl delete deployment todo-chatbot-frontend
```

---

## Best Practices

1. **Always preview commands**: kubectl-ai shows the actual kubectl command before execution - review it
2. **Test in development**: Run kubectl-ai commands in Minikube first before production
3. **Use full descriptions**: More specific natural language = better command matching
4. **Check status after changes**: Always verify with `kubectl-ai "show me the status"`
5. **Keep context**: Reference the application name (todo-chatbot) in commands for clarity

---

## Examples of Effective Commands

### Good ✓
- "Scale the backend deployment to 3 replicas"
- "Show me the logs from the todo-chatbot backend"
- "Port forward the frontend service to localhost:3000"
- "Check why the pods are not starting"

### Less Effective ✗
- "Scale backend up"
- "Show logs"
- "Port forward"
- "Why not working?"

---

## Getting Help

```bash
# Get kubectl-ai help
kubectl-ai --help

# Get help with specific command type
kubectl-ai "how do I scale a deployment"

# Ask about available commands
kubectl-ai "what commands can you help me with"
```

---

**For more kubectl-ai info**: https://github.com/kubernetes/kubectl-ai
**For kubectl reference**: https://kubernetes.io/docs/reference/kubectl/
