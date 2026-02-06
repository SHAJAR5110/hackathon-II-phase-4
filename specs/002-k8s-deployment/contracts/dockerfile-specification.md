# Dockerfile Specification: Todo Chatbot Images

**Date**: 2026-02-03
**Feature**: 002-k8s-deployment
**Status**: Specification

---

## Frontend Dockerfile (Next.js/React)

### Purpose
Containerize the Phase III frontend (React/Next.js) for Kubernetes deployment.

### Constraints
- **Base Image**: `node:18-alpine` (minimal Node.js runtime)
- **Final Image Size**: <500MB (optimized with multi-stage build)
- **Port**: 3000
- **Health Check**: GET `/health` → 200 OK
- **Strategy**: Multi-stage build (build + runtime stages)

### Dockerfile Structure

```dockerfile
# Stage 1: Build (dependencies + compilation)
FROM node:18-alpine AS builder

WORKDIR /app

# Copy only dependency files (for layer caching)
COPY frontend/package*.json ./

# Install dependencies
RUN npm ci --only=production && \
    npm run build

# Stage 2: Runtime (built artifacts only)
FROM node:18-alpine

WORKDIR /app

# Copy built application from builder stage
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/public ./public
COPY frontend/next.config.js ./
COPY frontend/package*.json ./

# Create health check endpoint
COPY frontend/public/health.json ./public/

# Non-root user (security best practice)
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001

USER nextjs

EXPOSE 3000

# Start application
CMD ["npm", "start"]

# Health check probe (for Kubernetes)
HEALTHCHECK --interval=10s --timeout=3s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

### Environment Variables
- `BACKEND_URL`: Backend service endpoint (e.g., `http://todo-chatbot-backend:8000`)
- `NEXT_PUBLIC_API_URL`: Public API URL (same as BACKEND_URL)
- `NODE_ENV`: Set to `production`

### Health Check Endpoint
- **Path**: `/health`
- **Method**: GET
- **Expected Response**: 200 OK
- **Body**: `{"status": "ok"}` or plain text "OK"
- **Implementation**: Create `public/health.json` or middleware endpoint

### Build Command
```bash
# Build frontend image
docker build -f docker/frontend/Dockerfile \
  -t todo-frontend:latest \
  ./frontend

# Alternative with Gordon (Docker AI):
docker ai "build an optimized docker image for next.js \
using node:18-alpine base, multi-stage build, \
tag as todo-frontend:latest"
```

### Expected Output
- Image name: `todo-frontend:latest`
- Size: 200-300MB
- Layers: ~15-20 (optimize via Docker layer caching)

---

## Backend Dockerfile (FastAPI)

### Purpose
Containerize the Phase III backend (FastAPI) for Kubernetes deployment.

### Constraints
- **Base Image**: `python:3.11-slim` (minimal Python runtime)
- **Final Image Size**: <300MB (optimized with multi-stage build)
- **Port**: 8000
- **Health Check**: GET `/health` → database connectivity verification
- **Strategy**: Multi-stage build

### Dockerfile Structure

```dockerfile
# Stage 1: Build (dependencies installation)
FROM python:3.11-slim AS builder

WORKDIR /app

# Copy requirements file
COPY backend/requirements.txt .

# Install dependencies to user site-packages
RUN pip install --user --no-cache-dir -r requirements.txt

# Stage 2: Runtime (application only)
FROM python:3.11-slim

WORKDIR /app

# Copy Python packages from builder
COPY --from=builder /root/.local /root/.local

# Copy application code
COPY backend/ .

# Update PATH to find pip packages
ENV PATH=/root/.local/bin:$PATH \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

# Non-root user (security)
RUN useradd -m -u 1001 fastapi

USER fastapi

EXPOSE 8000

# Start FastAPI with Uvicorn
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]

# Health check probe (for Kubernetes)
HEALTHCHECK --interval=10s --timeout=3s --start-period=15s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1
```

### Health Check Implementation
The `/health` endpoint in FastAPI:

```python
@app.get("/health")
async def health_check(db: Session = Depends(get_db)):
    try:
        # Verify database connectivity
        db.execute("SELECT 1")
        return {
            "status": "healthy",
            "database": "connected"
        }
    except Exception as e:
        raise HTTPException(
            status_code=503,
            detail="Database connection failed"
        )
```

### Environment Variables
- `DATABASE_URL`: PostgreSQL connection string (injected from Kubernetes Secret)
- `OPENAI_API_KEY`: OpenAI API key (injected from Kubernetes Secret)
- `ENVIRONMENT`: Set to `production`

### Build Command
```bash
# Build backend image
docker build -f docker/backend/Dockerfile \
  -t todo-backend:latest \
  ./backend

# Alternative with Gordon:
docker ai "build an optimized docker image for fastapi \
using python:3.11-slim base, install requirements.txt, \
add database connectivity health check, \
tag as todo-backend:latest"
```

### Expected Output
- Image name: `todo-backend:latest`
- Size: 200-250MB
- Layers: ~10-15

---

## Image Best Practices

### Multi-Stage Build Benefits
✅ Reduced final image size (build dependencies excluded)
✅ Faster builds (layer caching when code changes)
✅ Improved security (no build tools in final image)
✅ Easier maintenance (clear separation of concerns)

### Layer Caching Strategy
1. Copy dependency files first (rarely changes)
2. Install dependencies (builds cached layer)
3. Copy application code last (changes frequently)
4. Result: Code changes reuse dependency cache

### Base Image Selection
- **Frontend**: `node:18-alpine` (~150MB)
  - Lightweight, standard Node.js image
  - Alpine significantly smaller than Debian
- **Backend**: `python:3.11-slim` (~150MB)
  - Slim variant reduces size vs full Python image
  - Compatible with most packages (C extensions compiled)

### Security Best Practices
✅ Run as non-root user (RUN useradd, USER directive)
✅ No secrets in image (injected via environment variables)
✅ Exclude build artifacts from final image (multi-stage)
✅ Minimal base image (reduces attack surface)

### Health Check Configuration
✅ Liveness probe (detects dead containers, triggers restart)
✅ Readiness probe (detects startup delays, removes from service)
✅ For backend: Include database connectivity check
✅ Timeout: 3-5s, Period: 10-15s, Retries: 2-3

---

## Build & Deployment Workflow

### Local Testing

```bash
# Build
docker build -f docker/backend/Dockerfile -t todo-backend:test .

# Run
docker run -d --name test-backend \
  -p 8000:8000 \
  -e DATABASE_URL="postgresql://user:pass@host/db" \
  -e OPENAI_API_KEY="sk-..." \
  todo-backend:test

# Test health
curl http://localhost:8000/health

# Stop
docker stop test-backend && docker rm test-backend
```

### Load into Minikube

```bash
# Option 1: Use Minikube built-in
minikube image load todo-backend:latest

# Option 2: Push to registry
docker tag todo-backend:latest docker.io/username/todo-backend:latest
docker push docker.io/username/todo-backend:latest
# Update helm values.yaml to use docker.io/username/todo-backend:latest
```

### Deploy via Helm

```bash
# Helm automatically:
# 1. Pulls image from registry (or uses minikube local)
# 2. Creates container with specified port, env vars
# 3. Runs health checks
# 4. Restarts if health check fails
```

---

## Dockerfile Validation

### Lint Check
```bash
# Using Hadolint (Docker best practices linter)
hadolint docker/frontend/Dockerfile
hadolint docker/backend/Dockerfile

# Using docker CLI
docker build --dry-run -f docker/backend/Dockerfile .
```

### Security Scan
```bash
# Scan image for vulnerabilities
docker scan todo-backend:latest

# Or use Trivy
trivy image todo-backend:latest
```

### Image History
```bash
# View layer sizes
docker history todo-backend:latest

# Identify optimization opportunities
docker history todo-backend:latest --human --no-trunc
```

---

## Optimization Tips

### Reduce Image Size

1. **Use Alpine base images**: `-alpine` variants are 80% smaller
2. **Multi-stage builds**: Exclude build dependencies
3. **Cleanup package manager cache**:
   ```dockerfile
   RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
   RUN pip install --no-cache-dir -r requirements.txt
   ```
4. **Exclude unnecessary files**:
   ```dockerfile
   COPY --from=builder /app/.next ./.next
   COPY --from=builder /app/node_modules ./node_modules
   # Don't copy: node_modules, .git, .next/cache, etc.
   ```

### Improve Build Speed

1. **Leverage layer caching**: Copy dependency files before code
2. **Use .dockerignore**:
   ```
   node_modules
   npm-debug.log
   .git
   .next/cache
   __pycache__
   *.pyc
   .pytest_cache
   ```
3. **Parallel builds** (if applicable)

### Kubernetes Readiness

1. **Health check endpoint**: Required for readiness/liveness probes
2. **Environment variable handling**: Support injection from ConfigMap/Secret
3. **Graceful shutdown**: Handle SIGTERM signal for rolling updates
4. **Logging**: Write to stdout/stderr (Kubernetes collects these)

---

## Success Criteria

✅ Frontend image builds without errors
✅ Backend image builds without errors
✅ Final images < 500MB (frontend) and < 300MB (backend)
✅ Containers start successfully in 30 seconds
✅ Health check endpoints respond with 200 OK
✅ Environment variables properly injected
✅ Images run correctly in Kubernetes pods
✅ No security vulnerabilities detected
