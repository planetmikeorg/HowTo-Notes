# Containerfile In-Depth Guide

## Table of Contents
- [Overview](#overview)
- [Basic Instructions](#basic-instructions)
- [Advanced Instructions](#advanced-instructions)
- [Multi-Stage Builds](#multi-stage-builds)
- [Build Arguments and Variables](#build-arguments-and-variables)
- [Best Practices](#best-practices)
- [Optimization Techniques](#optimization-techniques)
- [Security Considerations](#security-considerations)
- [Real-World Examples](#real-world-examples)
- [Troubleshooting](#troubleshooting)

---

## Overview

Containerfile (also known as Dockerfile) is a text file containing instructions to build a container image. Podman and Buildah use these instructions to create layered, reproducible container images.

### Containerfile vs Dockerfile
- **Identical syntax**: Containerfile and Dockerfile are interchangeable
- **Naming preference**: Use "Containerfile" with Podman/Buildah
- **OCI Standard**: Both produce OCI-compliant images
- **Compatibility**: Podman reads both Containerfile and Dockerfile

### Basic Structure
```dockerfile
# Base image
FROM rockylinux:9

# Metadata
LABEL maintainer="admin@example.com"

# Install packages
RUN dnf install -y httpd && dnf clean all

# Copy files
COPY index.html /var/www/html/

# Set working directory
WORKDIR /app

# Expose port
EXPOSE 8080

# Define startup command
CMD ["/usr/sbin/httpd", "-D", "FOREGROUND"]
```

---

## Basic Instructions

### FROM - Base Image

```dockerfile
# Simple base image
FROM alpine:3.18

# From Docker Hub
FROM rockylinux:9

# Rocky Linux minimal image
FROM rockylinux:9-minimal

# With platform
FROM --platform=linux/amd64 alpine:3.18

# Named stage (multi-stage builds)
FROM node:18-alpine AS builder

# Scratch (empty base)
FROM scratch
```

### RUN - Execute Commands

```dockerfile
# Single command
RUN dnf install -y httpd

# Multiple commands (shell form)
RUN dnf update -y && \
    dnf install -y httpd nginx && \
    dnf clean all

# Exec form (preferred for clarity)
RUN ["/bin/bash", "-c", "echo hello"]

# Best practice: Chain commands
RUN dnf update -y \
 && dnf install -y \
      httpd \
      mod_ssl \
      httpd-tools \
 && dnf clean all \
 && rm -rf /var/cache/dnf

# Create users/groups
RUN groupadd -r appgroup && \
    useradd -r -g appgroup -s /bin/false appuser

# Set permissions
RUN chown -R appuser:appgroup /app && \
    chmod -R 755 /app
```

### COPY - Copy Files

```dockerfile
# Copy file
COPY index.html /var/www/html/

# Copy directory
COPY ./app /app/

# Copy with wildcards
COPY *.conf /etc/nginx/conf.d/

# Copy multiple sources
COPY file1.txt file2.txt /dest/

# Copy and rename
COPY config.yaml /app/config.yaml

# Change ownership (--chown)
COPY --chown=appuser:appgroup app/ /app/

# From specific stage (multi-stage)
COPY --from=builder /app/dist /app/

# Copy from external image
COPY --from=nginx:latest /etc/nginx/nginx.conf /etc/nginx/
```

### ADD - Add Files (Extended Copy)

```dockerfile
# ADD can extract archives
ADD archive.tar.gz /app/

# ADD can fetch URLs (not recommended)
ADD https://example.com/file.txt /app/

# Best practice: Use COPY instead of ADD
# except for auto-extraction
COPY file.txt /app/  # Preferred
ADD archive.tar.gz /app/  # OK for archives
```

### WORKDIR - Set Working Directory

```dockerfile
# Set working directory
WORKDIR /app

# Multiple WORKDIR (relative)
WORKDIR /app
WORKDIR subdir  # Now in /app/subdir
WORKDIR ../other  # Now in /app/other

# Using variables
ARG APP_DIR=/application
WORKDIR ${APP_DIR}

# Create if doesn't exist
WORKDIR /nonexistent/path  # Will be created
```

### ENV - Environment Variables

```dockerfile
# Set environment variable
ENV NODE_ENV=production

# Multiple variables
ENV NODE_ENV=production \
    PORT=3000 \
    DEBUG=false

# Map syntax (v3+)
ENV NODE_ENV production
ENV PORT 3000

# Use in subsequent instructions
ENV APP_HOME /app
WORKDIR ${APP_HOME}
COPY . ${APP_HOME}

# Common examples
ENV PATH="/app/bin:${PATH}"
ENV LANG=en_US.UTF-8
ENV TZ=America/New_York
```

### EXPOSE - Document Ports

```dockerfile
# Single port
EXPOSE 8080

# Multiple ports
EXPOSE 80 443

# With protocol
EXPOSE 8080/tcp
EXPOSE 53/udp

# Note: EXPOSE is documentation only
# Actual publishing happens at runtime with -p flag
```

### USER - Set User

```dockerfile
# Switch to user
USER appuser

# User and group
USER appuser:appgroup

# By UID:GID
USER 1000:1000

# Best practice: Create and switch
RUN useradd -r -u 1000 -g appgroup appuser
USER appuser

# Run as root for setup, then switch
RUN dnf install -y package
USER appuser
```

### VOLUME - Define Mount Points

```dockerfile
# Single volume
VOLUME /data

# Multiple volumes
VOLUME ["/data", "/logs"]

# Note: Creates anonymous volumes at runtime
# Named volumes specified with -v flag

# Best practice: Document expected mount points
VOLUME /var/lib/postgresql/data
VOLUME /var/log/app
```

### CMD - Default Command

```dockerfile
# Exec form (preferred)
CMD ["nginx", "-g", "daemon off;"]

# Shell form
CMD nginx -g "daemon off;"

# With ENTRYPOINT
CMD ["--help"]  # Passed as args to ENTRYPOINT

# Only last CMD is effective
CMD ["echo", "first"]
CMD ["echo", "second"]  # This one runs
```

### ENTRYPOINT - Configure Container Executable

```dockerfile
# Exec form (preferred)
ENTRYPOINT ["python3", "app.py"]

# Shell form
ENTRYPOINT python3 app.py

# With CMD (CMD provides default args)
ENTRYPOINT ["python3", "app.py"]
CMD ["--help"]

# Can be overridden at runtime
# podman run --entrypoint /bin/sh image
```

### CMD vs ENTRYPOINT

```dockerfile
# Example 1: ENTRYPOINT alone
ENTRYPOINT ["nginx", "-g", "daemon off;"]
# Run: podman run image
# Executes: nginx -g daemon off;

# Example 2: CMD alone
CMD ["nginx", "-g", "daemon off;"]
# Run: podman run image
# Executes: nginx -g daemon off;
# Run: podman run image echo hello
# Executes: echo hello (CMD replaced)

# Example 3: Both ENTRYPOINT and CMD
ENTRYPOINT ["python3"]
CMD ["app.py"]
# Run: podman run image
# Executes: python3 app.py
# Run: podman run image script.py
# Executes: python3 script.py (CMD replaced)

# Example 4: Wrapper script
ENTRYPOINT ["/entrypoint.sh"]
CMD ["start"]
# entrypoint.sh handles initialization
# CMD provides default action
```

---

## Advanced Instructions

### ARG - Build Arguments

```dockerfile
# Define build argument
ARG VERSION=1.0
ARG BUILD_DATE
ARG PYTHON_VERSION=3.11

# Use in FROM
ARG BASE_IMAGE=alpine:3.18
FROM ${BASE_IMAGE}

# Use in instructions
ARG APP_DIR=/app
WORKDIR ${APP_DIR}

# ARG scope: Available during build only
RUN echo "Building version ${VERSION}"

# Combine ARG and ENV
ARG ENVIRONMENT=dev
ENV APP_ENV=${ENVIRONMENT}

# Predefined ARGs (automatically available)
# HTTP_PROXY, HTTPS_PROXY, FTP_PROXY, NO_PROXY
# http_proxy, https_proxy, ftp_proxy, no_proxy

# Example with multiple stages
ARG NODE_VERSION=18
FROM node:${NODE_VERSION} AS builder
```

Build with arguments:
```bash
podman build --build-arg VERSION=2.0 --build-arg BUILD_DATE=$(date) .
```

### LABEL - Metadata

```dockerfile
# Single label
LABEL version="1.0"

# Multiple labels
LABEL maintainer="admin@example.com" \
      version="1.0" \
      description="My application" \
      org.opencontainers.image.authors="admin@example.com" \
      org.opencontainers.image.version="1.0.0"

# OCI standard labels
LABEL org.opencontainers.image.created="2026-03-03T10:00:00Z" \
      org.opencontainers.image.authors="admin@example.com" \
      org.opencontainers.image.url="https://example.com" \
      org.opencontainers.image.documentation="https://example.com/docs" \
      org.opencontainers.image.source="https://github.com/org/repo" \
      org.opencontainers.image.version="1.0.0" \
      org.opencontainers.image.revision="abc123" \
      org.opencontainers.image.vendor="Company Name" \
      org.opencontainers.image.licenses="MIT" \
      org.opencontainers.image.title="My App" \
      org.opencontainers.image.description="Application description"

# Custom labels
LABEL com.example.environment="production" \
      com.example.team="backend" \
      com.example.monitoring.prometheus.port="9090"
```

### ONBUILD - Trigger Instructions

```dockerfile
# In base image
FROM node:18-alpine
ONBUILD COPY package*.json ./
ONBUILD RUN npm install
ONBUILD COPY . .

# When this image is used as base, ONBUILD instructions execute
FROM mybaseimage:latest  # ONBUILD instructions run here
```

### STOPSIGNAL - Stop Signal

```dockerfile
# Set signal for container stop
STOPSIGNAL SIGTERM  # Default
STOPSIGNAL SIGINT
STOPSIGNAL SIGKILL

# Example: Graceful shutdown
STOPSIGNAL SIGTERM
CMD ["app", "--graceful-shutdown"]
```

### HEALTHCHECK - Container Health

```dockerfile
# Command-based healthcheck
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

# With all options
HEALTHCHECK --interval=30s \
            --timeout=10s \
            --start-period=40s \
            --retries=3 \
  CMD curl -f http://localhost/ || exit 1

# Shell form
HEALTHCHECK CMD curl -f http://localhost/health || exit 1

# Disable inherited healthcheck
HEALTHCHECK NONE

# Examples for different apps
# HTTP service
HEALTHCHECK CMD curl -f http://localhost:8080/health || exit 1

# Database
HEALTHCHECK CMD pg_isready -U postgres || exit 1

# Custom script
HEALTHCHECK CMD /app/healthcheck.sh

# TCP socket
HEALTHCHECK CMD nc -z localhost 3000 || exit 1
```

### SHELL - Default Shell

```dockerfile
# Change default shell
SHELL ["/bin/bash", "-c"]

# For Windows containers
SHELL ["powershell", "-command"]

# Multiple SHELL instructions
SHELL ["/bin/bash", "-c"]
RUN echo "Using bash"

SHELL ["/bin/sh", "-c"]
RUN echo "Using sh"

# With options
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
```

---

## Multi-Stage Builds

### Basic Multi-Stage

```dockerfile
# Stage 1: Build
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### Named Stages

```dockerfile
# Development stage
FROM node:18-alpine AS development
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["npm", "run", "dev"]

# Build stage
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Production stage
FROM node:18-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/index.js"]

# Test stage
FROM development AS test
RUN npm run test
```

Build specific stage:
```bash
# Build dev
podman build --target development -t myapp:dev .

# Build prod
podman build --target production -t myapp:prod .

# Build test
podman build --target test -t myapp:test .
```

### Go Application Example

```dockerfile
# Builder stage
FROM golang:1.21-alpine AS builder
WORKDIR /build

# Install dependencies
COPY go.mod go.sum ./
RUN go mod download

# Copy source
COPY . .

# Build with optimizations
RUN CGO_ENABLED=0 GOOS=linux go build \
    -a -installsuffix cgo \
    -ldflags="-w -s" \
    -o app .

# Final stage - minimal image
FROM alpine:3.18
RUN apk --no-cache add ca-certificates
WORKDIR /app
COPY --from=builder /build/app .
USER nobody:nobody
EXPOSE 8080
CMD ["./app"]
```

### Python Application Example

```dockerfile
# Builder stage
FROM python:3.11-slim AS builder
WORKDIR /app

# Install build dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc && \
    rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip wheel --no-cache-dir --no-deps --wheel-dir /app/wheels -r requirements.txt

# Final stage
FROM python:3.11-slim
WORKDIR /app

# Create user
RUN useradd -r -u 1000 appuser

# Copy wheels and install
COPY --from=builder /app/wheels /wheels
COPY --from=builder /app/requirements.txt .
RUN pip install --no-cache /wheels/*

# Copy application
COPY --chown=appuser:appuser . .

USER appuser
EXPOSE 8000
CMD ["gunicorn", "app:app", "--bind", "0.0.0.0:8000"]
```

### Copy from External Image

```dockerfile
FROM alpine:3.18

# Copy from another image
COPY --from=nginx:latest /etc/nginx/nginx.conf /etc/nginx/

# Copy from specific tag
COPY --from=myapp:v1.0 /app/config /app/config

# Use as base for binaries
FROM alpine:3.18
COPY --from=golang:1.21 /usr/local/go /usr/local/go
ENV PATH="/usr/local/go/bin:${PATH}"
```

---

## Build Arguments and Variables

### Scope and Precedence

```dockerfile
# ARG before FROM: Available for FROM only
ARG BASE_IMAGE=alpine:3.18
FROM ${BASE_IMAGE}

# ARG after FROM: Available for this stage
ARG VERSION=1.0
RUN echo "Version: ${VERSION}"

# ENV: Available at build AND runtime
ENV APP_ENV=production

# ARG to ENV persistence
ARG BUILD_ENV=dev
ENV APP_ENV=${BUILD_ENV}
```

### Default Values

```dockerfile
# With default
ARG NODE_VERSION=18
FROM node:${NODE_VERSION}

# Without default (must be provided at build)
ARG API_KEY
RUN echo "API Key: ${API_KEY}"

# Multiple defaults
ARG ENVIRONMENT=dev
ARG DEBUG=true
ARG PORT=8000
```

### Build-Time Variables

```dockerfile
# Build metadata
ARG BUILD_DATE
ARG GIT_COMMIT
ARG VERSION

LABEL build.date="${BUILD_DATE}" \
      build.commit="${GIT_COMMIT}" \
      build.version="${VERSION}"

# Conditional logic
ARG ENABLE_FEATURE=false
RUN if [ "${ENABLE_FEATURE}" = "true" ]; then \
      echo "Feature enabled"; \
      dnf install -y feature-package; \
    fi
```

Build example:
```bash
podman build \
  --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
  --build-arg GIT_COMMIT=$(git rev-parse HEAD) \
  --build-arg VERSION=1.0.0 \
  -t myapp:1.0.0 .
```

### Environment Substitution

```dockerfile
# Simple substitution
ENV APP_HOME=/app
WORKDIR ${APP_HOME}

# With default
ENV PORT=${PORT:-8000}

# String operations (bash)
SHELL ["/bin/bash", "-c"]
RUN echo "Uppercase: ${APP_NAME^^}"
RUN echo "Lowercase: ${APP_NAME,,}"
```

---

## Best Practices

### Layer Optimization

```dockerfile
# BAD: Creates multiple layers
FROM alpine:3.18
RUN apk update
RUN apk add curl
RUN apk add wget
RUN rm -rf /var/cache/apk/*

# GOOD: Single layer
FROM alpine:3.18
RUN apk update && \
    apk add --no-cache curl wget
```

### Order Matters

```dockerfile
# BAD: Changes to code rebuild everything
FROM node:18-alpine
COPY . /app
WORKDIR /app
RUN npm install

# GOOD: Dependencies cached separately
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
```

### Use .containerignore / .dockerignore

```bash
# .containerignore
.git
.gitignore
node_modules
*.log
*.md
.env
.env.*
Dockerfile
Containerfile
docker-compose.yml
.dockerignore
.containerignore
```

### Minimize Image Size

```dockerfile
# Use minimal base images
FROM alpine:3.18  # ~7MB
# vs
FROM ubuntu:22.04  # ~77MB

# Multi-stage builds
FROM golang:1.21 AS builder
# ... build
FROM alpine:3.18
COPY --from=builder /app .

# Clean up in same layer
RUN dnf install -y package && \
    dnf clean all && \
    rm -rf /var/cache/dnf

# Remove unnecessary files
RUN rm -rf /tmp/* /var/tmp/*

# Use specific package versions
RUN dnf install -y httpd-2.4.* && dnf clean all
```

### Security

```dockerfile
# Don't run as root
RUN useradd -r -u 1000 appuser
USER appuser

# Don't expose secrets
# BAD
ENV API_KEY=secret123

# GOOD
# Pass at runtime: podman run -e API_KEY=secret

# Use trusted base images
FROM rockylinux:9

# Scan images
# podman build -t myapp .
# trivy image myapp

# Update packages
RUN dnf update -y && dnf clean all

# Minimal privileges
USER nobody:nobody

# Read-only root
# podman run --read-only myapp
```

### Metadata

```dockerfile
# Add comprehensive labels
LABEL org.opencontainers.image.created="2026-03-03T10:00:00Z" \
      org.opencontainers.image.authors="team@example.com" \
      org.opencontainers.image.url="https://example.com" \
      org.opencontainers.image.documentation="https://example.com/docs" \
      org.opencontainers.image.source="https://github.com/org/repo" \
      org.opencontainers.image.version="1.0.0" \
      org.opencontainers.image.revision="${GIT_COMMIT}" \
      org.opencontainers.image.vendor="Company Name" \
      org.opencontainers.image.licenses="MIT" \
      org.opencontainers.image.title="My App" \
      org.opencontainers.image.description="Application description"
```

---

## Optimization Techniques

### Build Cache

```dockerfile
# Leverage cache by ordering
# 1. Base setup (changes rarely)
FROM alpine:3.18
RUN apk update && apk add ca-certificates

# 2. Dependencies (changes occasionally)
COPY requirements.txt .
RUN pip install -r requirements.txt

# 3. Code (changes frequently)
COPY . .
```

### Parallel Builds

```dockerfile
# Independent stages can build in parallel
FROM base AS deps
RUN install-dependencies

FROM base AS lint
RUN run-linter

FROM base AS test
COPY --from=deps /app/node_modules ./node_modules
RUN run-tests

FROM base AS final
COPY --from=deps /app/node_modules ./node_modules
COPY . .
```

### BuildKit Features

```dockerfile
# syntax=docker/dockerfile:1.4

# Mount secrets (not in image)
RUN --mount=type=secret,id=npm_token \
    npm config set //registry.npmjs.org/:_authToken $(cat /run/secrets/npm_token) && \
    npm install

# Mount cache
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Mount SSH
RUN --mount=type=ssh \
    git clone git@github.com:private/repo.git
```

### Squash Layers

```bash
# Build and squash all layers into one
podman build --squash -t myapp:latest .

# Note: Loses layer caching benefits
```

---

## Security Considerations

### Secure Containerfile

```dockerfile
# Use specific versions
FROM rockylinux:9.1

# Update packages
RUN dnf update -y && \
    dnf clean all && \
    rm -rf /var/cache/dnf

# Create non-root user
RUN useradd -r -u 1000 -g appgroup -s /bin/false appuser && \
    mkdir -p /app && \
    chown -R appuser:appgroup /app

# Install only needed packages
RUN dnf install -y \
      python3-3.9.* \
      python3-pip-21.* \
 && dnf clean all

# Don't copy sensitive files
COPY --chown=appuser:appgroup app/ /app/

# Set working directory
WORKDIR /app

# Switch to non-root
USER appuser

# Don't expose unnecessary ports
EXPOSE 8080

# Use exec form
CMD ["python3", "app.py"]

# Add healthcheck
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:8080/health || exit 1

# Add security labels
LABEL security.vulnerability.scan="passed" \
      security.scan.date="2026-03-03"
```

### Secrets Management

```dockerfile
# BAD: Secrets in image
ENV API_KEY=secret123
ENV DB_PASSWORD=pass123

# GOOD: Pass at runtime
# podman run -e API_KEY=secret myapp

# GOOD: Use secrets mount (BuildKit)
RUN --mount=type=secret,id=api_key \
    export API_KEY=$(cat /run/secrets/api_key) && \
    configure-app

# GOOD: Multi-stage without secrets
FROM base AS config
ARG API_KEY
RUN configure-app --key=${API_KEY}

FROM base
COPY --from=config /app/config /app/config
# API_KEY not in final image
```

### Minimal Attack Surface

```dockerfile
# Use distroless or minimal base
FROM gcr.io/distroless/static-debian11

# Or alpine
FROM alpine:3.18
RUN apk add --no-cache ca-certificates

# Remove shell/package managers in final image
FROM scratch
COPY --from=builder /app/binary /app/binary
CMD ["/app/binary"]

# Read-only filesystem
# podman run --read-only myapp

# Drop capabilities
# podman run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp
```

---

## Real-World Examples

### Node.js Application

```dockerfile
# Build stage
FROM node:18-alpine AS builder
WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --only=production && \
    npm cache clean --force

# Copy source
COPY . .

# Build application
RUN npm run build

# Production stage
FROM node:18-alpine
WORKDIR /app

# Create user
RUN addgroup -g 1000 appgroup && \
    adduser -D -u 1000 -G appgroup appuser

# Copy from builder
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --chown=appuser:appgroup package.json ./

# Switch to non-root
USER appuser

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD node healthcheck.js || exit 1

# Start application
CMD ["node", "dist/index.js"]
```

### Python Django Application

```dockerfile
# Builder stage
FROM python:3.11-slim AS builder

# Install build dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      gcc \
      postgresql-client \
      libpq-dev \
 && rm -rf /var/lib/apt/lists/*

# Create wheels
WORKDIR /app
COPY requirements.txt .
RUN pip wheel --no-cache-dir --no-deps --wheel-dir /app/wheels -r requirements.txt

# Final stage
FROM python:3.11-slim

# Install runtime dependencies only
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      libpq5 \
      postgresql-client \
 && rm -rf /var/lib/apt/lists/*

# Create user
RUN useradd -r -u 1000 django

WORKDIR /app

# Install Python packages
COPY --from=builder /app/wheels /wheels
COPY --from=builder /app/requirements.txt .
RUN pip install --no-cache /wheels/* && \
    rm -rf /wheels

# Copy application
COPY --chown=django:django . .

# Collect static files
RUN python manage.py collectstatic --noinput

USER django

EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD python -c "import requests; requests.get('http://localhost:8000/health')" || exit 1

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "4", "project.wsgi:application"]
```

### Go Application

```dockerfile
# Build stage
FROM golang:1.21-alpine AS builder

# Install git for dependencies
RUN apk add --no-cache git ca-certificates

WORKDIR /build

# Copy go mod files
COPY go.mod go.sum ./
RUN go mod download

# Copy source
COPY . .

# Build with optimizations
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -a -installsuffix cgo \
    -ldflags="-w -s -X main.Version=${VERSION} -X main.BuildTime=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
    -o app ./cmd/server

# Final stage - scratch
FROM scratch

# Copy CA certificates
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copy binary
COPY --from=builder /build/app /app

# Use non-root user
USER 65534:65534

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s \
  CMD ["/app", "healthcheck"] || exit 1

ENTRYPOINT ["/app"]
```

### NGINX with Custom Config

```dockerfile
FROM nginx:1.24-alpine

# Remove default config
RUN rm /etc/nginx/conf.d/default.conf

# Copy custom configs
COPY nginx.conf /etc/nginx/nginx.conf
COPY conf.d/ /etc/nginx/conf.d/

# Copy static files
COPY --chown=nginx:nginx html/ /usr/share/nginx/html/

# Create cache directory
RUN mkdir -p /var/cache/nginx/client_temp && \
    chown -R nginx:nginx /var/cache/nginx

# Use non-root user
USER nginx

EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget --quiet --tries=1 --spider http://localhost:8080/health || exit 1

CMD ["nginx", "-g", "daemon off;"]
```

---

## Troubleshooting

### Build Failures

```bash
# Verbose output
podman build --progress=plain -t myapp .

# No cache
podman build --no-cache -t myapp .

# Specific target
podman build --target=builder -t myapp:builder .

# Show build context
podman build --no-cache --progress=plain . 2>&1 | grep "COPY"

# Check specific layer
podman build -t test --target=stage-name .
podman run -it test /bin/sh
```

### Common Errors

#### COPY failed: no such file or directory
```dockerfile
# Check .containerignore
# Verify file exists in build context
# Use correct relative path

# Debug
COPY . /tmp/context
RUN ls -la /tmp/context
```

#### Permission denied
```dockerfile
# Fix permissions in build
RUN chmod +x /app/script.sh

# Or use --chown
COPY --chown=appuser:appgroup app/ /app/
```

#### Package not found
```dockerfile
# Update package lists first
RUN dnf update -y && \
    dnf install -y package-name

# Or for Alpine
RUN apk update && \
    apk add --no-cache package-name
```

### Layer Inspection

```bash
# View image history
podman history myapp:latest

# Inspect specific layer
podman inspect myapp:latest

# View layers
podman image tree myapp:latest

# Save and examine
podman save myapp:latest -o myapp.tar
tar -tf myapp.tar
```

### Debug Build

```dockerfile
# Add debug output
RUN echo "Debug: Building version ${VERSION}" && \
    ls -la /app && \
    env

# Stop at specific point
RUN exit 1  # Force failure to inspect up to this point

# Or build to stage
# podman build --target=debug-stage -t debug .
```

---

## Quick Reference

### Common Patterns

```dockerfile
# Minimal Python app
FROM python:3.11-alpine
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]

# Minimal Node app
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
CMD ["node", "index.js"]

# Minimal Go app
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY . .
RUN go build -o app
FROM alpine:3.18
COPY --from=builder /app/app /app
CMD ["/app"]
```

### Build Commands

```bash
# Basic build
podman build -t myapp:latest .

# With args
podman build --build-arg VERSION=1.0 -t myapp:v1 .

# Specific file
podman build -f Containerfile.prod -t myapp:prod .

# Specific target
podman build --target production -t myapp:prod .

# No cache
podman build --no-cache -t myapp:latest .

# Platform
podman build --platform linux/amd64 -t myapp:latest .
```

### Best Practices Checklist

- [ ] Use specific base image versions
- [ ] Run as non-root user
- [ ] Minimize layers (combine RUN commands)
- [ ] Order instructions by change frequency
- [ ] Use .containerignore
- [ ] Add healthcheck
- [ ] Add metadata labels
- [ ] Clean up in same layer
- [ ] Use multi-stage builds
- [ ] Don't include secrets
- [ ] Pin package versions
- [ ] Use COPY instead of ADD
- [ ] Prefer exec form for CMD/ENTRYPOINT
