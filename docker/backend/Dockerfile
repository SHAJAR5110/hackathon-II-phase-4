# Multi-stage build for FastAPI backend
# Stage 1: Build stage with dependencies
FROM python:3.11-slim AS builder

WORKDIR /app

# Copy requirements file
COPY backend/requirements.txt .

# Install dependencies to user site-packages
RUN pip install --user --no-cache-dir -r requirements.txt

# Stage 2: Runtime stage with only built artifacts
FROM python:3.11-slim

WORKDIR /app

# Copy Python packages from builder stage
COPY --from=builder /root/.local /root/.local

# Copy application code
COPY backend/ .

# Set environment variables for Python
ENV PATH=/root/.local/bin:$PATH \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

# Create non-root user for security
RUN useradd -m -u 1001 fastapi

USER fastapi

# Expose port
EXPOSE 8000

# Health check endpoint (Kubernetes liveness/readiness probes)
HEALTHCHECK --interval=10s --timeout=3s --start-period=15s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health').read()" || exit 1

# Start FastAPI with Uvicorn
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
