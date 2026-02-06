# Multi-stage build for Next.js/React frontend
# Stage 1: Build stage with all dependencies
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files (minimal cache invalidation)
COPY frontend/package.json frontend/package-lock.json* frontend/yarn.lock* ./

# Install dependencies
RUN npm ci --only=production || yarn install --production

# Copy source code and build
COPY frontend/ .

# Build Next.js application
RUN npm run build || yarn build

# Stage 2: Runtime stage with only built artifacts
FROM node:18-alpine

WORKDIR /app

# Copy built application from builder stage
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/public ./public
COPY frontend/next.config.js ./
COPY frontend/package.json ./

# Create non-root user for security
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001

USER nextjs

# Expose port
EXPOSE 3000

# Health check endpoint (Kubernetes liveness/readiness probes)
HEALTHCHECK --interval=10s --timeout=3s --start-period=10s --retries=3 \
  CMD wget -q -O- http://localhost:3000/health || exit 1

# Start application
CMD ["npm", "start"]
