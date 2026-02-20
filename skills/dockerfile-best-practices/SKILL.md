---
name: dockerfile-best-practices
description: Apply modern Dockerfile best practices for secure, efficient, and production-ready container images. Use when writing, reviewing, or optimizing Dockerfiles, or when the user asks about Docker image size, security, build caching, multi-stage builds, .dockerignore, secrets handling, health checks, or container hardening.
---

# Dockerfile Best Practices

Apply these 10 rules when writing or reviewing any Dockerfile.

## 1. Use a .dockerignore File

Exclude `.git`, `node_modules`, and local secrets to keep builds clean. This reduces build context size and prevents accidental leakage of sensitive files into the image.

```dockerignore
.git
node_modules
.env
*.secret
```

## 2. Clean Up in the Same Layer

Never leave package cache in the final image. Run install and cleanup in the same `RUN` instruction:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*
```

## 3. Don't Use ENV for Secrets

`ENV` values are visible via `docker inspect`. Use `ARG` for build-time variables that don't persist. For runtime secrets, mount them securely using volumes or secret managers (e.g., Docker BuildKit secrets, Vault).

```dockerfile
# Bad
ENV DB_PASSWORD=hunter2

# Good — build-time only
ARG DB_PASSWORD

# Good — runtime mount
# docker run -v /secrets/db_password:/run/secrets/db_password myapp
```

## 4. Run as a Non-Root User

Running as root is a major security risk. Create a dedicated user and switch immediately:

```dockerfile
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
USER appuser
```

## 5. Optimize Caching Order

Copy dependency files before source code so Docker caches dependency installation across code changes:

```dockerfile
COPY package.json package-lock.json ./
RUN npm ci --production
COPY . .
```

## 6. Pin Your Versions

Avoid `:latest` or unversioned packages — they produce unpredictable builds:

```dockerfile
# Bad
FROM node:latest

# Good
FROM node:20.11-alpine3.19
RUN apk add --no-cache curl=8.5.0-r0
```

## 7. Use Minimal Base Images

Prefer `alpine`, `distroless`, or `-slim` tags over full OS images. This reduces attack surface and image size dramatically.

| Base Image         | Approx Size |
|--------------------|-------------|
| `ubuntu:24.04`     | ~78 MB      |
| `node:20`          | ~350 MB     |
| `node:20-alpine`   | ~50 MB      |
| `gcr.io/distroless/static` | ~2 MB |

## 8. Use Multi-Stage Builds

Separate build environment from runtime. Keep compilers and build tools out of production:

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY . .
RUN go build -o /app/server .

FROM gcr.io/distroless/static
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]
```

## 9. Add a HEALTHCHECK

A running process doesn't mean a working application. Define a check so orchestrators can restart unhealthy containers:

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1
```

## 10. Combine RUN Instructions

Every `RUN` line creates a new filesystem layer. Combine related commands to reduce layer count and image size:

```dockerfile
# Bad — 3 layers
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# Good — 1 layer
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*
```
