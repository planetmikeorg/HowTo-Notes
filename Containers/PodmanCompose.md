# Podman Compose Comprehensive Guide

## Table of Contents
- [Overview](#overview)
- [Installation](#installation)
- [Compose File Format](#compose-file-format)
- [Service Configuration](#service-configuration)
- [Networks](#networks)
- [Volumes](#volumes)
- [Environment Variables](#environment-variables)
- [Build Configuration](#build-configuration)
- [Commands](#commands)
- [Advanced Features](#advanced-features)
- [Migration from Docker Compose](#migration-from-docker-compose)
- [Real-World Examples](#real-world-examples)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)

---

## Overview

Podman Compose is a Python-based implementation of Docker Compose for Podman. It allows you to define and run multi-container applications using YAML files compatible with Docker Compose.

### Key Features
- **Docker Compose Compatible**: Uses standard docker-compose.yml syntax
- **Rootless Support**: Works with rootless Podman
- **Pod Integration**: Can create Podman pods from compose files
- **No Daemon**: Leverages Podman's daemonless architecture
- **Drop-in Replacement**: Minimal changes needed from Docker Compose

### Podman Compose vs Docker Compose
| Feature | Podman Compose | Docker Compose |
|---------|----------------|----------------|
| Daemon Required | No | Yes |
| Rootless Mode | Native support | Limited |
| Pod Creation | Yes (optional) | No |
| Swarm Mode | No | Yes (v3+) |
| Syntax | Docker Compose compatible | Native |

---

## Installation

### RHEL 8/9/10

#### Method 1: DNF (Recommended)
```bash
# Install from official repositories
sudo dnf install -y podman-compose

# Verify installation
podman-compose --version
```

#### Method 2: Python Pip
```bash
# Install Python pip if not already installed
sudo dnf install -y python3-pip

# Install podman-compose
pip3 install podman-compose

# Add to PATH if needed
export PATH=$PATH:~/.local/bin

# Verify installation
podman-compose --version
```

#### Method 3: From Source
```bash
# Clone repository
git clone https://github.com/containers/podman-compose.git
cd podman-compose

# Install
pip3 install .

# Or run directly
python3 podman_compose.py --version
```

### Dependencies
```bash
# Ensure podman is installed
sudo dnf install -y podman

# Install additional tools
sudo dnf install -y python3-yaml python3-dotenv

# For rootless mode
sudo dnf install -y slirp4netns fuse-overlayfs
```

---

## Compose File Format

### Basic Structure
```yaml
version: '3.8'  # Optional for podman-compose

services:
  # Define your services here
  service-name:
    image: nginx:latest
    # Service configuration

volumes:
  # Define named volumes
  volume-name:

networks:
  # Define custom networks
  network-name:

secrets:
  # Define secrets (v3.1+)
  secret-name:

configs:
  # Define configs (v3.3+)
  config-name:
```

### Version Compatibility
```yaml
# Podman Compose supports most Docker Compose versions
# Recommended: version 3.x (for compatibility)

# Version 2.x
version: '2.4'

# Version 3.x (recommended)
version: '3.8'

# No version (defaults to latest)
# services: ...
```

### File Naming
```bash
# Default file names (searched in order)
docker-compose.yml
docker-compose.yaml
compose.yml
compose.yaml

# Custom file name
podman-compose -f custom-compose.yml up

# Multiple files (merged)
podman-compose -f base.yml -f override.yml up
```

---

## Service Configuration

### Image Specification
```yaml
services:
  # Pull from registry
  web:
    image: nginx:1.24-alpine
  
  # Specific registry
  app:
    image: registry.example.com/myapp:v1.0
  
  # By digest
  db:
    image: postgres@sha256:abc123...
  
  # Rocky Linux
  rocky:
    image: rockylinux:9
```

### Container Configuration
```yaml
services:
  web:
    image: nginx:latest
    
    # Container name
    container_name: web-server
    
    # Hostname
    hostname: webserver
    
    # Domain name
    domainname: example.com
    
    # Restart policy
    restart: always  # no, always, on-failure, unless-stopped
    
    # User
    user: "1000:1000"
    # user: nginx
    
    # Working directory
    working_dir: /app
    
    # Entrypoint override
    entrypoint: /docker-entrypoint.sh
    
    # Command override
    command: nginx -g 'daemon off;'
    # command: ["nginx", "-g", "daemon off;"]
    
    # Stop signal
    stop_signal: SIGTERM
    
    # Stop grace period
    stop_grace_period: 30s
    
    # Labels
    labels:
      app: web
      environment: production
      version: "1.0"
```

### Ports
```yaml
services:
  web:
    image: nginx
    ports:
      # HOST:CONTAINER
      - "8080:80"
      
      # HOST:CONTAINER/PROTOCOL
      - "8443:443/tcp"
      - "53:53/udp"
      
      # Specific interface
      - "127.0.0.1:8080:80"
      
      # Random host port
      - "80"
      
      # Range mapping
      - "3000-3005:3000-3005"
      
    # Expose ports (documentation only)
    expose:
      - "80"
      - "443"
```

### Environment Variables
```yaml
services:
  app:
    image: myapp
    
    # Direct definition
    environment:
      - DATABASE_HOST=db
      - DATABASE_PORT=5432
      - DEBUG=false
    
    # Map format
    environment:
      DATABASE_HOST: db
      DATABASE_PORT: 5432
      DEBUG: "false"
    
    # From file
    env_file:
      - .env
      - ./config/.env.production
```

### Dependencies
```yaml
services:
  web:
    image: nginx
    depends_on:
      - app
      - cache
  
  app:
    image: myapp
    depends_on:
      - db
  
  db:
    image: postgres
  
  cache:
    image: redis

# With conditions (v2.1+, limited in v3)
services:
  app:
    depends_on:
      db:
        condition: service_healthy
```

### Health Checks
```yaml
services:
  app:
    image: myapp
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      # Or string format
      # test: curl -f http://localhost:8080/health || exit 1
      
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    
  # Disable inherited healthcheck
  web:
    image: nginx
    healthcheck:
      disable: true
```

### Resource Limits
```yaml
services:
  app:
    image: myapp
    
    # Memory limits
    mem_limit: 512m
    mem_reservation: 256m
    
    # CPU limits
    cpus: 1.5  # 1.5 cores
    cpu_shares: 512
    cpu_quota: 50000
    cpu_period: 100000
    
    # PID limit
    pids_limit: 100
    
    # Deploy resources (v3)
    deploy:
      resources:
        limits:
          cpus: '1.5'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
```

### Security
```yaml
services:
  app:
    image: myapp
    
    # Privileged mode (avoid in production)
    privileged: false
    
    # Capabilities
    cap_add:
      - NET_ADMIN
      - SYS_TIME
    cap_drop:
      - ALL
    
    # Security options
    security_opt:
      - label=type:svirt_apache_t
      - no-new-privileges:true
      - seccomp=unconfined
    
    # Read-only root filesystem
    read_only: true
    
    # Tmpfs for writable dirs
    tmpfs:
      - /tmp
      - /run
    
    # SELinux
    # Handled automatically by Podman with :Z on volumes
```

---

## Networks

### Default Network
```yaml
# All services automatically join default network
services:
  web:
    image: nginx
  app:
    image: myapp
# web and app can communicate using service names
```

### Custom Networks
```yaml
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # No external access

services:
  web:
    image: nginx
    networks:
      - frontend
  
  app:
    image: myapp
    networks:
      - frontend
      - backend
  
  db:
    image: postgres
    networks:
      - backend
```

### Network Configuration
```yaml
networks:
  custom:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.name: br-custom
    
    ipam:
      driver: default
      config:
        - subnet: 172.28.0.0/16
          gateway: 172.28.0.1
          ip_range: 172.28.5.0/24
    
    labels:
      environment: production
```

### Service Network Settings
```yaml
services:
  web:
    image: nginx
    
    # Multiple networks
    networks:
      frontend:
        ipv4_address: 172.28.1.10
        aliases:
          - webserver
          - www
      backend:
        ipv4_address: 172.29.1.10
    
    # Network mode
    # network_mode: "host"
    # network_mode: "none"
    # network_mode: "service:other-service"
    # network_mode: "container:container-name"
    
    # DNS
    dns:
      - 8.8.8.8
      - 8.8.4.4
    
    dns_search:
      - example.com
    
    # Extra hosts
    extra_hosts:
      - "somehost:162.242.195.82"
      - "otherhost:50.31.209.229"
    
    # MAC address
    mac_address: 02:42:ac:11:00:02
```

---

## Volumes

### Named Volumes
```yaml
version: '3.8'

volumes:
  db-data:
  app-data:
  cache-data:

services:
  db:
    image: postgres
    volumes:
      - db-data:/var/lib/postgresql/data:Z
  
  app:
    image: myapp
    volumes:
      - app-data:/app/data:Z
  
  cache:
    image: redis
    volumes:
      - cache-data:/data:Z
```

### Volume Configuration
```yaml
volumes:
  db-data:
    driver: local
    driver_opts:
      type: none
      device: /mnt/storage/db
      o: bind
    
    labels:
      environment: production
      backup: daily
  
  nfs-data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.100,rw
      device: ":/path/to/nfs"
```

### Bind Mounts
```yaml
services:
  web:
    image: nginx
    volumes:
      # Relative path (relative to compose file)
      - ./html:/usr/share/nginx/html:Z
      
      # Absolute path
      - /host/config:/etc/nginx/conf.d:ro,Z
      
      # Home directory
      - ~/data:/app/data:Z
      
      # SELinux labels
      # :z - shared between containers
      # :Z - private to container
      - ./shared:/app/shared:z
```

### Anonymous Volumes
```yaml
services:
  app:
    image: myapp
    volumes:
      # Anonymous volume (managed by Podman)
      - /app/data
```

### Tmpfs Mounts
```yaml
services:
  app:
    image: myapp
    tmpfs:
      - /tmp
      - /run
    
    # Or with options
    tmpfs:
      - type: tmpfs
        target: /tmp
        tmpfs:
          size: 100000000  # bytes
          mode: 1777
```

### Volume Mount Options
```yaml
services:
  app:
    image: myapp
    volumes:
      # Long syntax
      - type: volume
        source: app-data
        target: /app/data
        read_only: false
        volume:
          nocopy: false
      
      - type: bind
        source: ./config
        target: /app/config
        read_only: true
        bind:
          propagation: rprivate
      
      - type: tmpfs
        target: /tmp
        tmpfs:
          size: 1000000000
```

---

## Environment Variables

### .env File
```bash
# .env file (automatically loaded)
# Located in same directory as compose file

# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME=myapp
DB_USER=admin
DB_PASSWORD=secretpass

# Application
APP_ENV=production
APP_DEBUG=false
APP_PORT=8000

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
```

### Using Variables in Compose File
```yaml
version: '3.8'

services:
  db:
    image: postgres:${PG_VERSION:-15}
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    ports:
      - "${DB_PORT}:5432"
  
  app:
    image: myapp:${APP_VERSION:-latest}
    environment:
      DATABASE_URL: postgresql://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}
      REDIS_URL: redis://cache:${REDIS_PORT}
      APP_ENV: ${APP_ENV}
      DEBUG: ${APP_DEBUG}
    ports:
      - "${APP_PORT}:8000"
  
  cache:
    image: redis:${REDIS_VERSION:-7-alpine}
    ports:
      - "${REDIS_PORT}:6379"
```

### Multiple .env Files
```yaml
# Load specific env files
services:
  app:
    image: myapp
    env_file:
      - .env
      - .env.production
      - ./secrets/.env.secrets
```

### Interpolation
```yaml
# Default values
image: postgres:${PG_VERSION:-15}

# Required variables (fails if not set)
image: myapp:${APP_VERSION:?APP_VERSION not set}

# Conditional values
environment:
  DEBUG: ${DEBUG:-false}
  PORT: ${PORT:-8000}
```

---

## Build Configuration

### Simple Build
```yaml
services:
  app:
    build: .
    # Builds from ./Containerfile or ./Dockerfile
```

### Build Context
```yaml
services:
  app:
    build:
      context: ./app
      dockerfile: Containerfile.prod
```

### Build Arguments
```yaml
services:
  app:
    build:
      context: .
      args:
        - VERSION=1.0
        - BUILD_DATE=2026-03-03
        - ENV=production
    
    # Or map format
    build:
      context: .
      args:
        VERSION: 1.0
        BUILD_DATE: 2026-03-03
        ENV: production
```

### Build Target (Multi-stage)
```yaml
services:
  app-dev:
    build:
      context: .
      target: development
  
  app-prod:
    build:
      context: .
      target: production
```

### Build with Cache
```yaml
services:
  app:
    build:
      context: .
      cache_from:
        - myapp:latest
        - myapp:cache
      
      # Additional options
      network: host
      shm_size: '2gb'
```

### Complete Build Example
```yaml
services:
  web:
    build:
      context: ./web
      dockerfile: Containerfile
      args:
        NODE_VERSION: 18
        APP_ENV: production
      target: production
      cache_from:
        - web:latest
      labels:
        - "com.example.version=1.0"
        - "com.example.environment=prod"
    image: myapp/web:v1.0
    ports:
      - "3000:3000"
```

---

## Commands

### Lifecycle Management

#### Up - Start Services
```bash
# Start all services
podman-compose up

# Detached mode (background)
podman-compose up -d

# Build before starting
podman-compose up --build

# Recreate containers
podman-compose up --force-recreate

# No deps (don't start linked services)
podman-compose up --no-deps web

# Scale services
podman-compose up -d --scale app=3

# Remove orphans
podman-compose up --remove-orphans

# Exit code from service
podman-compose up --exit-code-from app
```

#### Down - Stop and Remove
```bash
# Stop and remove containers, networks
podman-compose down

# Remove volumes too
podman-compose down -v

# Remove images
podman-compose down --rmi all  # all images
podman-compose down --rmi local  # only local images

# Timeout for stop
podman-compose down -t 30
```

#### Start/Stop/Restart
```bash
# Start existing containers
podman-compose start

# Start specific service
podman-compose start web

# Stop containers (don't remove)
podman-compose stop

# Stop with timeout
podman-compose stop -t 30

# Restart services
podman-compose restart

# Restart specific service
podman-compose restart web
```

#### Pause/Unpause
```bash
# Pause services
podman-compose pause

# Pause specific service
podman-compose pause web

# Unpause services
podman-compose unpause

# Unpause specific service
podman-compose unpause web
```

### Container Management

#### List Containers
```bash
# List running containers
podman-compose ps

# List all containers (including stopped)
podman-compose ps -a

# Filter output
podman-compose ps --services
podman-compose ps --filter "status=running"
```

#### Execute Commands
```bash
# Execute command in service
podman-compose exec web sh

# Run as specific user
podman-compose exec --user nginx web sh

# Execute without TTY
podman-compose exec -T web ls -la

# Pass environment variables
podman-compose exec -e DEBUG=true web env

# With working directory
podman-compose exec -w /app web pwd
```

#### Run One-off Commands
```bash
# Run command in new container
podman-compose run app python manage.py migrate

# Don't start deps
podman-compose run --no-deps app python script.py

# Remove container after run
podman-compose run --rm app python test.py

# Override entrypoint
podman-compose run --entrypoint /bin/sh app

# Publish ports
podman-compose run -p 8080:80 web

# Run as different user
podman-compose run --user root app bash
```

### Logging

```bash
# View logs
podman-compose logs

# Follow logs
podman-compose logs -f

# Specific service
podman-compose logs web
podman-compose logs -f web app

# Timestamps
podman-compose logs -t web

# Tail last N lines
podman-compose logs --tail=100 web

# Since timestamp
podman-compose logs --since 2026-03-01T10:00:00 web
podman-compose logs --since 1h web

# Until timestamp
podman-compose logs --until 2026-03-03T23:59:59 web
```

### Building

```bash
# Build all services
podman-compose build

# Build specific service
podman-compose build web

# Build without cache
podman-compose build --no-cache

# Build with pull
podman-compose build --pull

# Parallel build
podman-compose build --parallel

# Build with build args
podman-compose build --build-arg VERSION=2.0 web
```

### Image Management

```bash
# Pull service images
podman-compose pull

# Pull specific service
podman-compose pull web

# Pull quietly
podman-compose pull -q

# Push images
podman-compose push

# Push specific service
podman-compose push web
```

### Configuration

```bash
# Validate and view compose file
podman-compose config

# Resolve and display
podman-compose config --resolve-image-digests

# Show only services
podman-compose config --services

# Show volumes
podman-compose config --volumes

# Convert to JSON
podman-compose config --format json
```

### Port Mapping

```bash
# Display port mappings
podman-compose port web 80

# Show all ports
podman-compose port web
```

### Top

```bash
# Display running processes
podman-compose top

# Specific service
podman-compose top web
```

---

## Advanced Features

### Profiles (v3.9+)
```yaml
services:
  web:
    image: nginx
    # Always starts
  
  app:
    image: myapp
    profiles:
      - dev
  
  debug:
    image: busybox
    profiles:
      - debug
    command: sleep infinity

# Start without profiles
podman-compose up  # Only starts web

# Start with profile
podman-compose --profile dev up  # Starts web and app
podman-compose --profile dev --profile debug up  # Starts all
```

### Extends
```yaml
# common.yml
services:
  base:
    image: alpine
    environment:
      - ENV=dev
    volumes:
      - /tmp:/tmp

# app-compose.yml
version: '3.8'
services:
  web:
    extends:
      file: common.yml
      service: base
    image: nginx
```

### Multiple Compose Files
```bash
# Override settings with multiple files
podman-compose -f docker-compose.yml -f docker-compose.prod.yml up

# Files are merged in order
# Later files override earlier ones
```

```yaml
# docker-compose.yml (base)
services:
  app:
    image: myapp:latest
    environment:
      - ENV=dev

# docker-compose.prod.yml (override)
services:
  app:
    environment:
      - ENV=production
    deploy:
      replicas: 3
```

### Secrets Management
```yaml
secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    external: true

services:
  app:
    image: myapp
    secrets:
      - db_password
      - api_key
    environment:
      - DB_PASSWORD_FILE=/run/secrets/db_password
      - API_KEY_FILE=/run/secrets/api_key
```

### Using Podman Pods
```yaml
# Create services in a pod
# Use --pod-args flag

# All services will share network namespace
services:
  web:
    image: nginx
  
  app:
    image: myapp

# Start with pod
podman-compose --pod-args="--name=webapp,--publish=8080:80" up -d
```

### Custom Container Names
```yaml
services:
  web:
    image: nginx
    container_name: custom-web-server
  
  app:
    image: myapp
    container_name: myapp-${ENV:-dev}
```

---

## Migration from Docker Compose

### Compatibility
```bash
# Most Docker Compose files work without changes
# Replace docker-compose with podman-compose

# Docker Compose
docker-compose up -d

# Podman Compose
podman-compose up -d
```

### Known Differences

#### Rootless Mode
```yaml
# Docker Compose (runs as root typically)
services:
  web:
    ports:
      - "80:80"  # Privileged port

# Podman Compose (rootless)
services:
  web:
    ports:
      - "8080:80"  # Use non-privileged port
```

#### SELinux Labels
```yaml
# Add :Z for SELinux systems
services:
  web:
    volumes:
      - ./html:/usr/share/nginx/html:Z  # Private label
      - ./shared:/shared:z  # Shared label
```

#### Swarm Mode
```yaml
# Docker Compose v3 deploy section (Swarm)
services:
  app:
    deploy:
      replicas: 3  # NOT SUPPORTED in podman-compose
      
# Use --scale instead
# podman-compose up -d --scale app=3
```

### Migration Steps

```bash
# 1. Test compose file validation
podman-compose config

# 2. Update port mappings (if rootless)
# Change privileged ports (<1024) to non-privileged

# 3. Add SELinux labels to volumes
# Add :Z or :z to volume mounts

# 4. Update image references
# Ensure images are accessible from configured registries

# 5. Test in non-production first
podman-compose up

# 6. Update CI/CD pipelines
# Replace docker-compose with podman-compose

# 7. Update documentation
# Note any differences or limitations
```

---

## Real-World Examples

### Web Application Stack
```yaml
version: '3.8'

services:
  nginx:
    image: nginx:1.24-alpine
    container_name: webapp-nginx
    ports:
      - "8080:80"
      - "8443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro,Z
      - ./nginx/conf.d:/etc/nginx/conf.d:ro,Z
      - ./html:/usr/share/nginx/html:Z
      - nginx-logs:/var/log/nginx:Z
    depends_on:
      - app
    networks:
      - frontend
    restart: always

  app:
    build:
      context: ./app
      dockerfile: Containerfile
      args:
        - PYTHON_VERSION=3.11
    image: myapp:latest
    container_name: webapp-app
    environment:
      - DATABASE_URL=postgresql://webapp:password@db:5432/webapp
      - REDIS_URL=redis://cache:6379/0
      - SECRET_KEY=${SECRET_KEY}
      - DEBUG=false
    volumes:
      - app-data:/app/data:Z
      - ./app/static:/app/static:Z
    depends_on:
      - db
      - cache
    networks:
      - frontend
      - backend
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: always

  db:
    image: postgres:15-alpine
    container_name: webapp-db
    environment:
      - POSTGRES_DB=webapp
      - POSTGRES_USER=webapp
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - db-data:/var/lib/postgresql/data:Z
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro,Z
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U webapp"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: always

  cache:
    image: redis:7-alpine
    container_name: webapp-cache
    command: redis-server --appendonly yes
    volumes:
      - cache-data:/data:Z
    networks:
      - backend
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
    restart: always

volumes:
  app-data:
  db-data:
  cache-data:
  nginx-logs:

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true
```

### Development Environment
```yaml
version: '3.8'

services:
  dev:
    build:
      context: .
      target: development
    image: myapp:dev
    container_name: dev-environment
    volumes:
      - .:/app:Z
      - dev-cache:/app/.cache:Z
    ports:
      - "3000:3000"
      - "9229:9229"  # Debug port
    environment:
      - NODE_ENV=development
      - DEBUG=*
    command: npm run dev
    stdin_open: true
    tty: true

  db:
    image: postgres:15-alpine
    container_name: dev-db
    environment:
      - POSTGRES_DB=dev
      - POSTGRES_USER=dev
      - POSTGRES_PASSWORD=dev
    ports:
      - "5432:5432"
    volumes:
      - dev-db:/var/lib/postgresql/data:Z

volumes:
  dev-cache:
  dev-db:
```

### Microservices Architecture
```yaml
version: '3.8'

services:
  gateway:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./gateway/nginx.conf:/etc/nginx/nginx.conf:ro,Z
    depends_on:
      - auth
      - users
      - orders
    networks:
      - public

  auth:
    image: mycompany/auth-service:latest
    environment:
      - JWT_SECRET=${JWT_SECRET}
      - DB_HOST=auth-db
    depends_on:
      - auth-db
    networks:
      - public
      - auth-backend

  users:
    image: mycompany/user-service:latest
    environment:
      - DB_HOST=users-db
    depends_on:
      - users-db
    networks:
      - public
      - users-backend

  orders:
    image: mycompany/order-service:latest
    environment:
      - DB_HOST=orders-db
      - RABBITMQ_URL=amqp://queue:5672
    depends_on:
      - orders-db
      - queue
    networks:
      - public
      - orders-backend
      - messaging

  auth-db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=auth
      - POSTGRES_PASSWORD=${AUTH_DB_PASS}
    volumes:
      - auth-db-data:/var/lib/postgresql/data:Z
    networks:
      - auth-backend

  users-db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=users
      - POSTGRES_PASSWORD=${USERS_DB_PASS}
    volumes:
      - users-db-data:/var/lib/postgresql/data:Z
    networks:
      - users-backend

  orders-db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=orders
      - POSTGRES_PASSWORD=${ORDERS_DB_PASS}
    volumes:
      - orders-db-data:/var/lib/postgresql/data:Z
    networks:
      - orders-backend

  queue:
    image: rabbitmq:3-management-alpine
    ports:
      - "15672:15672"
    volumes:
      - queue-data:/var/lib/rabbitmq:Z
    networks:
      - messaging

volumes:
  auth-db-data:
  users-db-data:
  orders-db-data:
  queue-data:

networks:
  public:
  auth-backend:
    internal: true
  users-backend:
    internal: true
  orders-backend:
    internal: true
  messaging:
    internal: true
```

---

## Troubleshooting

### Common Issues

#### Service Won't Start
```bash
# Check logs
podman-compose logs service-name

# Check config
podman-compose config

# Verify images exist
podman images

# Pull images
podman-compose pull

# Rebuild
podman-compose build --no-cache service-name
```

#### Port Already in Use
```bash
# Check what's using port
sudo ss -tulpn | grep :8080

# Use different port
# Edit docker-compose.yml
ports:
  - "8081:80"  # Changed from 8080

# Or stop conflicting service
podman stop conflicting-container
```

#### Permission Denied (Rootless)
```bash
# Check subuid/subgid
cat /etc/subuid
cat /etc/subgid

# Fix SELinux labels
# Add :Z to volumes in compose file
volumes:
  - ./data:/app/data:Z

# Check file ownership
ls -la ./data
podman unshare chown -R 0:0 ./data
```

#### Network Issues
```bash
# Inspect networks
podman network ls
podman network inspect projectname_default

# Recreate network
podman-compose down
podman network rm projectname_default
podman-compose up -d

# Check DNS resolution
podman-compose exec service-name ping other-service
```

#### Volume Issues
```bash
# List volumes
podman volume ls

# Inspect volume
podman volume inspect projectname_volume-name

# Remove and recreate
podman-compose down -v
podman-compose up -d

# Check volume permissions
podman volume inspect projectname_volume-name
```

#### Build Failures
```bash
# Build with verbose output
podman-compose build --progress=plain

# Check Containerfile
cat Containerfile

# Build manually
podman build -t myapp:test .

# Check for disk space
df -h
```

### Debugging Strategies

```bash
# 1. Validate compose file
podman-compose config

# 2. Check service status
podman-compose ps

# 3. View all logs
podman-compose logs

# 4. Check specific service
podman-compose logs -f service-name

# 5. Execute shell in service
podman-compose exec service-name sh

# 6. Inspect running containers
podman inspect $(podman-compose ps -q)

# 7. Check networks
podman network ls
podman network inspect networkname

# 8. Check volumes
podman volume ls
podman volume inspect volumename

# 9. Test connectivity
podman-compose exec service1 ping service2

# 10. Restart services
podman-compose restart
```

---

## Best Practices

### Security

1. **Use specific image tags**
```yaml
services:
  web:
    image: nginx:1.24.0-alpine  # Not 'latest'
```

2. **Run as non-root user**
```yaml
services:
  app:
    image: myapp
    user: "1000:1000"
```

3. **Use secrets for sensitive data**
```yaml
secrets:
  db_password:
    file: ./secrets/db_password.txt

services:
  app:
    secrets:
      - db_password
```

4. **Limit resource usage**
```yaml
services:
  app:
    mem_limit: 512m
    cpus: 1.0
```

5. **Use read-only filesystem**
```yaml
services:
  app:
    read_only: true
    tmpfs:
      - /tmp
```

### Performance

1. **Use volumes for persistent data**
2. **Minimize image layers**
3. **Use multi-stage builds**
4. **Enable healthchecks**
5. **Set appropriate restart policies**

### Development

1. **Use profiles for different environments**
```yaml
services:
  debug:
    profiles: ["debug"]
```

2. **Use .env for local config**
```bash
# .env.example (committed)
DB_PASSWORD=changeme

# .env (gitignored)
DB_PASSWORD=actualpassword
```

3. **Mount code as volume in dev**
```yaml
services:
  app:
    volumes:
      - ./src:/app/src:Z  # Live reload
```

### Production

1. **Use specific versions**
2. **Set resource limits**
3. **Enable monitoring/logging**
4. **Use health checks**
5. **Set restart policies**
6. **Use systemd for service management**

```bash
# Generate systemd units
podman-compose up -d
podman generate systemd --new --name container-name > service.unit
```

### Maintenance

```bash
# Regular cleanup
podman-compose down -v  # Stop and remove
podman system prune -a  # Clean up

# Update images
podman-compose pull
podman-compose up -d

# Backup volumes
podman run --rm \
  -v projectname_volume:/data:Z \
  -v $(pwd):/backup:Z \
  alpine tar czf /backup/volume-backup.tar.gz -C /data .

# View disk usage
podman-compose down
podman system df
```

---

## Quick Reference

### Common Commands
```bash
# Start
podman-compose up -d

# Stop
podman-compose down

# Logs
podman-compose logs -f

# Execute
podman-compose exec service sh

# Rebuild
podman-compose build --no-cache

# Restart
podman-compose restart

# Status
podman-compose ps

# Cleanup
podman-compose down -v
```

### Environment Files
```bash
# .env
DB_HOST=localhost
APP_ENV=production

# Load in compose
environment:
  - DATABASE_HOST=${DB_HOST}
```

### Configuration Locations
```bash
# Compose files
docker-compose.yml
docker-compose.yaml
compose.yml

# Environment
.env
.env.production
```

### Getting Help
```bash
# Command help
podman-compose --help
podman-compose up --help

# Config validation
podman-compose config

# Version
podman-compose --version
```
