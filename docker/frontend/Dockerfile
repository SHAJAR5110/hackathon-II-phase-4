# Multi-stage build for Next.js/React frontend
# Using node:20-slim (Debian) with build tools for native module compilation
FROM node:20-slim AS builder

WORKDIR /app

# Install build tools for native module compilation
RUN apt-get update && apt-get install -y python3 make g++ && rm -rf /var/lib/apt/lists/*

# Copy package files
COPY frontend/package.json frontend/package-lock.json* frontend/yarn.lock* ./

# Install dependencies and rebuild native modules
RUN npm ci --legacy-peer-deps || yarn install
RUN npm rebuild

# Copy source code and build
COPY frontend/ .

# Build Next.js application with SWC instead of Turbopack (avoid lightningcss issues)
RUN NEXT_SKIP_ENV_VALIDATION=1 npm run build || yarn build

# Stage 2: Runtime stage with only built artifacts
FROM node:20-slim

WORKDIR /app

# Copy built application from builder stage
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/public ./public
COPY --from=builder /app/package.json ./

# Create non-root user for security
RUN useradd -m -u 1001 nextjs

USER nextjs

EXPOSE 3000

# Health check endpoint (Kubernetes liveness/readiness probes)
HEALTHCHECK --interval=10s --timeout=3s --start-period=10s --retries=3 \
  CMD wget -q -O- http://localhost:3000/health || exit 1

# Start application
CMD ["npm", "start"]
