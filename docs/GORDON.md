# Docker AI Agent (Gordon) Integration Guide

**Overview**: Use Gordon, Docker Desktop's AI agent, to streamline Docker operations like building images, managing containers, and diagnosing issues.

**Note**: Gordon requires Docker Desktop 4.53+ and is available in select regions/subscription tiers. Fallback to manual Docker CLI if unavailable.

---

## Table of Contents

1. [Setup & Installation](#setup--installation)
2. [Image Building](#image-building)
3. [Container Diagnostics](#container-diagnostics)
4. [Image Optimization](#image-optimization)
5. [Troubleshooting](#troubleshooting)
6. [Fallback to Manual Docker CLI](#fallback-to-manual-docker-cli)

---

## Setup & Installation

### Enable Gordon in Docker Desktop

1. Open **Docker Desktop** application
2. Go to **Settings** (gear icon)
3. Navigate to **Features in development** (or **Experimental Features**)
4. Look for **"Use Gordon"** option
5. Toggle it **ON**
6. Restart Docker Desktop if prompted

### Verify Gordon is Working

```bash
# Test Gordon capabilities
docker ai "what can you do?"

# Expected response:
# Gordon can help you with:
# - Building optimized Docker images
# - Understanding and fixing container errors
# - Analyzing Dockerfiles and suggesting improvements
# - ...
```

### Check Regional Availability

Gordon is available in these regions:
- United States
- Europe
- Some Asian regions

If Gordon is not available in your region, use the [fallback commands](#fallback-to-manual-docker-cli).

---

## Image Building

### Build Frontend Image with Gordon

```bash
# Optimized build with suggestions
docker ai "build an optimized docker image for the next.js frontend \
located in ./frontend directory, use node:18-alpine base image \
for minimal size, include a health check at /health endpoint, \
and tag it as todo-frontend:latest"
```

**What Gordon does:**
- Analyzes your Next.js application
- Suggests Docker best practices
- Recommends multi-stage build if not present
- Optimizes base image selection
- Ensures health check implementation
- Builds the image

### Build Backend Image with Gordon

```bash
# Optimized FastAPI build
docker ai "build an optimized docker image for the fastapi backend \
located in ./backend directory, use python:3.11-slim base, \
install dependencies from requirements.txt, include health check \
that verifies database connectivity, tag as todo-backend:latest"
```

### Build with Custom Requirements

```bash
# Specify particular constraints
docker ai "build a docker image for fastapi with minimal layers, \
use layer caching optimization, ensure final image is under 300MB, \
tag as todo-backend:latest"

# With port specification
docker ai "build docker image that exposes port 8000 and includes \
liveness probe at /health endpoint"
```

---

## Container Diagnostics

### Diagnose Container Failures

```bash
# Understand why a container fails to start
docker ai "why is my backend container failing to start"

# Gordon will:
# - Analyze container logs
# - Check for missing dependencies
# - Identify configuration issues
# - Suggest fixes
```

### Analyze Container Logs

```bash
# Get summary of error logs
docker ai "analyze these container logs and explain what's wrong"

# Debug specific error
docker ai "why is my container getting 'ModuleNotFoundError: No module named fastapi'"

# Check database connection
docker ai "why can't the backend container connect to the database"
```

### Check Port Conflicts

```bash
docker ai "why can't the container bind to port 8000"

# Gordon will:
# - Check if port is already in use
# - Show what process is using it
# - Suggest solutions (different port, stop conflicting service)
```

---

## Image Optimization

### Optimize Image Size

```bash
# Analyze and optimize image
docker ai "optimize my backend image, it's currently 400MB and I want it under 300MB"

# Gordon will:
# - Analyze each layer
# - Suggest removing unnecessary dependencies
# - Recommend Alpine base images
# - Propose multi-stage build improvements
```

### Improve Build Performance

```bash
# Speed up builds
docker ai "my docker builds are slow, how can I optimize layer caching"

# Gordon will:
# - Analyze current Dockerfile layer order
# - Suggest moving frequently-changing code to later layers
# - Recommend dependency caching improvements
# - Show before/after build times
```

### Dockerfile Review

```bash
# Get comprehensive review
docker ai "review my Dockerfile and suggest security improvements"

# Gordon will:
# - Check for security best practices
# - Recommend running as non-root user
# - Identify unnecessary packages
# - Suggest health check additions
```

---

## Common Gordon Commands

### Image Analysis

```bash
# What's in the image
docker ai "tell me what's in my todo-backend:latest image"

# Why it's large
docker ai "why is my frontend image 500MB"

# Layer analysis
docker ai "show me the size of each layer in my backend image"
```

### Build Optimization

```bash
# Suggest improvements
docker ai "suggest docker best practices for a node.js application"

# Security hardening
docker ai "make my dockerfile more secure"

# Multi-stage optimization
docker ai "create a multi-stage dockerfile for my fastapi backend"
```

### Running Containers

```bash
# Generate run command
docker ai "give me the docker run command for todo-backend with these env vars: DATABASE_URL, OPENAI_API_KEY"

# Port mapping
docker ai "how do I run the backend container with port 8000 exposed"

# Health checks
docker ai "add a health check to my backend container"
```

---

## Troubleshooting

### Gordon Misunderstands Command

If Gordon generates incorrect suggestions:

1. **Be more specific**: Add more context to your request
2. **Use manual command**: Fall back to [manual Docker CLI](#fallback-to-manual-docker-cli)
3. **Clarify constraints**: Include framework details (Node.js, FastAPI, etc.)

### Gordon Not Available

If you see "Gordon is not available in your region":

- Check [regional availability](#check-regional-availability) above
- Use fallback [manual Docker commands](#fallback-to-manual-docker-cli)
- Contact Docker support to request availability in your region

### Slow Response

If Gordon responses are slow:

- Network latency to Docker's AI service
- Try again or use manual Docker CLI
- Report performance issues to Docker

---

## Fallback to Manual Docker CLI

If Gordon is unavailable or you prefer manual control, use these equivalent commands:

### Building Images

```bash
# Frontend image (multi-stage)
docker build \
  -f docker/frontend/Dockerfile \
  -t todo-frontend:latest .

# Backend image (multi-stage)
docker build \
  -f docker/backend/Dockerfile \
  -t todo-backend:latest .

# Build with custom base image
docker build \
  -f docker/backend/Dockerfile \
  --build-arg BASE_IMAGE=python:3.11-slim \
  -t todo-backend:latest .

# Build with caching
docker build \
  --cache-from todo-backend:latest \
  -f docker/backend/Dockerfile \
  -t todo-backend:latest .
```

### Running Containers Locally

```bash
# Run frontend with port mapping
docker run -d \
  --name todo-frontend \
  -p 3000:3000 \
  -e BACKEND_URL=http://localhost:8000 \
  todo-frontend:latest

# Run backend with environment variables
docker run -d \
  --name todo-backend \
  -p 8000:8000 \
  -e DATABASE_URL="postgresql://user:pass@host/db" \
  -e OPENAI_API_KEY="sk-..." \
  todo-backend:latest

# Stop and remove containers
docker stop todo-frontend todo-backend
docker rm todo-frontend todo-backend
```

### Analyzing Images

```bash
# View image layers
docker history todo-backend:latest

# View with human-readable sizes
docker history todo-backend:latest --human --no-trunc

# Inspect image
docker inspect todo-backend:latest

# View image size
docker images todo-backend:latest

# List all images
docker images | grep todo-
```

### Container Debugging

```bash
# View logs
docker logs todo-backend

# Stream logs
docker logs -f todo-backend

# View last 100 lines
docker logs --tail 100 todo-backend

# With timestamps
docker logs --timestamps todo-backend

# Check container status
docker ps -a | grep todo-

# Inspect running container
docker inspect todo-backend

# Execute command in running container
docker exec -it todo-backend /bin/sh
```

### Image Security Scanning

```bash
# Scan image for vulnerabilities (requires docker scan)
docker scan todo-backend:latest

# Using Trivy (if installed)
trivy image todo-backend:latest

# View image details
docker inspect todo-backend:latest | grep -i "env\|labels"
```

---

## Best Practices

### When to Use Gordon

✓ **Use Gordon for:**
- Learning Docker best practices
- Exploring image optimization ideas
- Generating Dockerfile suggestions
- Diagnosing container failures
- Getting build recommendations

### When to Use Manual CLI

✓ **Use manual CLI for:**
- Production builds (reproducible, version-controlled)
- CI/CD pipelines (need deterministic results)
- Advanced Docker features (build args, secrets mounting)
- When Gordon is unavailable
- When you want full control

### Workflow Recommendation

```
1. Use Gordon to explore and learn
2. Review Gordon's suggestions
3. Implement suggestions in manual Dockerfile
4. Test and refine
5. Use manual Docker CLI in production/CI-CD
```

---

## Examples

### Example 1: Build Frontend with Gordon

```bash
# Gordon command
docker ai "build an optimized next.js docker image with node:18-alpine, \
include /health endpoint, final size under 500MB, tag as todo-frontend:latest"

# What to check:
docker images todo-frontend:latest       # Verify size
docker run -p 3000:3000 todo-frontend   # Test locally
curl localhost:3000/health                # Check health endpoint
```

### Example 2: Diagnose Backend Failure

```bash
# Container failed to start
docker run -p 8000:8000 \
  -e DATABASE_URL="postgresql://..." \
  todo-backend:latest

# Get Gordon's help
docker ai "why does my backend container fail with permission denied error"

# Check logs
docker logs <container-id>
```

### Example 3: Optimize Image Size

```bash
# Current image is too large
docker ai "my backend image is 500MB, how can I reduce it to under 300MB with multi-stage build"

# Check size reduction
docker images todo-backend:latest
# Before: 500MB
# After: 220MB (with Gordon's suggestions)
```

---

## Regional Availability Status

Check current Gordon availability:

```bash
# Try Gordon command - if it works, you have access
docker ai "test"

# If you see error: "Gordon is not available"
# Then try manual commands instead
```

### Regions with Gordon Support
- 🟢 United States
- 🟢 Europe
- 🟢 Selected Asian regions
- 🔴 Other regions (check Docker website)

---

## Getting Help

```bash
# Ask Gordon for help
docker ai "what can you help me with"

# Ask about Docker best practices
docker ai "what are docker security best practices"

# Get command suggestions
docker ai "how do I build a docker image"

# Official Docker AI docs
docker docs ai
```

---

## Conclusion

**Gordon** provides a conversational interface to Docker, making it easier to:
- ✓ Build optimized images
- ✓ Understand error messages
- ✓ Learn Docker best practices
- ✓ Troubleshoot container issues

**When Gordon is unavailable**, use the [manual Docker CLI commands](#fallback-to-manual-docker-cli) documented above.

---

**See also:**
- [DEPLOYMENT.md](./DEPLOYMENT.md) - Complete deployment guide
- [KUBECTL-AI.md](./KUBECTL-AI.md) - Kubernetes management commands
- [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) - Common issues and fixes
