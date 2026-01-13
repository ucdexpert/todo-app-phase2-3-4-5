# Dockerfiles Specification

## Backend Dockerfile Specification

### Multi-Stage Build Structure
The backend Dockerfile should implement a multi-stage build to optimize the final image size and security:

```dockerfile
# Build stage
FROM python:3.13-slim as builder

WORKDIR /app

# Install system dependencies needed for building Python packages
RUN apt-get update && apt-get install -y \
    build-essential \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements and install Python dependencies
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Runtime stage
FROM python:3.13-slim

WORKDIR /app

# Install runtime system dependencies only
RUN apt-get update && apt-get install -y \
    && rm -rf /var/lib/apt/lists/*

# Copy installed Python packages from builder stage
COPY --from=builder /root/.local /root/.local

# Copy application code
COPY . .

# Create non-root user for security
RUN useradd --create-home --shell /bin/bash app \
    && chown -R app:app /app
USER app

# Make sure scripts in .local are usable
ENV PATH=/root/.local/bin:$PATH

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Command to run the application
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Backend Dockerfile Requirements
- Use Python 3.13-slim as the base image
- Implement multi-stage build to reduce final image size
- Install only necessary runtime dependencies
- Create a non-root user for security
- Include proper health check endpoint
- Expose port 8000 for the FastAPI application
- Copy only necessary files to optimize build cache

## Frontend Dockerfile Specification

### Multi-Stage Build Structure
The frontend Dockerfile should implement a multi-stage build to optimize the final image size and security:

```dockerfile
# Build stage
FROM node:20-alpine as builder

WORKDIR /app

# Copy package files and install dependencies
COPY package*.json ./
RUN npm ci --only=production

# Copy source code and build the application
COPY . .
RUN npm run build

# Runtime stage
FROM node:20-alpine

WORKDIR /app

# Install only runtime dependencies
RUN apk add --no-cache dumb-init

# Copy built application from builder stage
COPY --from=builder /app/package*.json ./
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public

# Create non-root user for security
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001
USER nextjs

# Use dumb-init to handle PID 1 issues
ENTRYPOINT ["dumb-init", "--"]

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

# Command to run the application
CMD ["npm", "start"]
```

### Frontend Dockerfile Requirements
- Use Node.js 20-alpine as the base image for smaller size
- Implement multi-stage build to reduce final image size
- Install only production dependencies
- Use dumb-init to handle PID 1 issues in containers
- Create a non-root user for security
- Include proper health check endpoint
- Expose port 3000 for the Next.js application
- Copy only necessary files to optimize build cache

## Environment Variables

### Backend Environment Variables
The backend Docker image should support the following environment variables:

- `DATABASE_URL`: Connection string for the PostgreSQL database
- `OPENAI_API_KEY`: API key for OpenAI services
- `BETTER_AUTH_SECRET`: Secret key for authentication
- `BETTER_AUTH_URL`: Base URL for authentication service
- `MCP_SERVER_URL`: URL for the MCP server (if applicable)
- `LOG_LEVEL`: Logging level (DEBUG, INFO, WARNING, ERROR)

### Frontend Environment Variables
The frontend Docker image should support the following environment variables:

- `NEXT_PUBLIC_API_BASE_URL`: Base URL for the backend API
- `NEXT_PUBLIC_OPENAI_DOMAIN_KEY`: Domain key for OpenAI ChatKit
- `NEXT_PUBLIC_BETTER_AUTH_URL`: Base URL for authentication service
- `NEXT_PUBLIC_MCP_SERVER_URL`: URL for the MCP server (if applicable)

## Optimization Strategies

### Image Size Optimization
- Use Alpine Linux variants where appropriate
- Implement multi-stage builds to separate build and runtime environments
- Remove unnecessary build dependencies from the final image
- Use .dockerignore to exclude unnecessary files from the build context
- Use specific version tags for base images to ensure reproducibility

### Security Optimization
- Use non-root users in containers
- Scan images for vulnerabilities using tools like Trivy
- Implement minimal required permissions
- Use read-only root filesystem where possible
- Implement secrets management for sensitive data

### Build Optimization
- Leverage Docker build cache by ordering layers properly
- Use .dockerignore to exclude unnecessary files
- Implement build arguments for flexibility
- Use buildx for multi-platform builds if needed
- Implement build secrets for sensitive build-time data

## Build Context Considerations

### .dockerignore for Backend
```
**/.git
**/.gitignore
**/.env*
**/node_modules
**/Dockerfile*
**/docker-compose*
**/.dockerignore
**/.specify
**/specs
**/history
**/README.md
**/CLAUDE.md
**/QWEN.md
**/CONSTITUTION.md
**/Dockerfile*
```

### .dockerignore for Frontend
```
**/.git
**/.gitignore
**/.env*
**/Dockerfile*
**/docker-compose*
**/.dockerignore
**/.specify
**/specs
**/history
**/README.md
**/CLAUDE.md
**/QWEN.md
**/CONSTITUTION.md
**/__pycache__
**/*.py
**/requirements.txt
**/backend
```

## Docker Build Best Practices

1. Use specific version tags for base images (e.g., python:3.13-slim instead of python:latest)
2. Order Dockerfile instructions to leverage build cache (least changing instructions first)
3. Combine RUN instructions where appropriate to reduce layers
4. Use .dockerignore to exclude unnecessary files from build context
5. Implement health checks for container monitoring
6. Use non-root users for security
7. Implement multi-stage builds to reduce attack surface
8. Scan images for vulnerabilities before deployment