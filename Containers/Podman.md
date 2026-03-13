# Podman User Guide for RHEL

## Table of Contents
- [Overview](#overview)
- [Installation](#installation)
- [Basic Concepts](#basic-concepts)
- [Container Operations](#container-operations)
- [Image Management](#image-management)
- [Networking](#networking)
- [Volumes and Storage](#volumes-and-storage)
- [Pods](#pods)
- [Rootless Containers](#rootless-containers)
- [Security](#security)
- [Systemd Integration](#systemd-integration)
- [Registry Management](#registry-management)
- [Monitoring and Logs](#monitoring-and-logs)
- [Troubleshooting](#troubleshooting)
- [Migration from Docker](#migration-from-docker)

**Note**: See [Containerfile.md](Containerfile.md) for in-depth Containerfile guide and [PodmanCompose.md](PodmanCompose.md) for Podman Compose documentation.

---

## Overview

Podman (Pod Manager) is a daemonless container engine for developing, managing, and running OCI containers on Linux systems. It's the default container engine in RHEL 8, 9, and 10.

### Key Features
- **Daemonless**: No background daemon required (unlike Docker)
- **Rootless**: Can run containers as non-root user
- **Pod Support**: Native pod concept (like Kubernetes)
- **Docker Compatible**: Drop-in replacement for Docker CLI
- **Systemd Integration**: Native systemd service generation
- **Security**: Enhanced security with SELinux, seccomp, capabilities

### Podman vs Docker
| Feature | Podman | Docker |
|---------|--------|--------|
| Daemon | No | Yes |
| Root Required | No (rootless mode) | Yes (typically) |
| Pods | Native support | No (needs Compose/Swarm) |
| Systemd | Built-in integration | Third-party |
| OCI Compliant | Yes | Yes |
| CLI Compatibility | Docker-compatible | Standard |

---

## Installation

### RHEL 8/9/10

#### Install Podman
```bash
# Install podman and related tools
sudo dnf install -y podman podman-docker

# Verify installation
podman --version

# Check podman info
podman info
```

#### Install Additional Tools
```bash
# Container tools
sudo dnf install -y \
  buildah \
  skopeo \
  podman-compose \
  containernetworking-plugins \
  slirp4netns \
  fuse-overlayfs

# For systemd integration
sudo dnf install -y podman-systemd

# For remote management
sudo dnf install -y podman-remote

# Development tools
sudo dnf install -y podman-tui podman-plugins
```

#### Enable Podman Socket (Optional)
```bash
# For rootless
systemctl --user enable --now podman.socket

# For root
sudo systemctl enable --now podman.socket

# Verify socket
systemctl --user status podman.socket
```

### Configuration Files

```bash
# System-wide configuration
/etc/containers/storage.conf        # Storage configuration
/etc/containers/registries.conf     # Registry configuration
/etc/containers/policy.json         # Image signing policy
/etc/containers/containers.conf     # Container runtime config

# User-specific (rootless)
~/.config/containers/storage.conf
~/.config/containers/registries.conf
~/.config/containers/containers.conf
```

#### Configure Registries
```bash
# Edit registries configuration
sudo vi /etc/containers/registries.conf

# Add registries
[registries.search]
registries = ['registry.access.redhat.com', 'registry.redhat.io', 'docker.io', 'quay.io']

[registries.insecure]
registries = []

[registries.block]
registries = []
```

#### Configure Storage
```bash
# Check current storage
podman info --format '{{.Store.GraphDriverName}}'

# Edit storage configuration
sudo vi /etc/containers/storage.conf

# Overlay2 settings (default)
[storage]
driver = "overlay"
runroot = "/run/containers/storage"
graphroot = "/var/lib/containers/storage"

[storage.options]
additionalimagestores = []
```

---

## Basic Concepts

### Container Lifecycle States
- **Created**: Container created but not started
- **Running**: Container is executing
- **Paused**: Container execution suspended
- **Stopped**: Container stopped gracefully
- **Exited**: Container finished execution

### Image Layers
```bash
# View image layers
podman history <image>

# Inspect image details
podman inspect <image>

# Show image tree
podman image tree <image>
```

---

## Container Operations

### Running Containers

#### Basic Run
```bash
# Run container interactively
podman run -it rockylinux:9 bash

# Run container in background (detached)
podman run -d --name myapp nginx

# Run with automatic removal after exit
podman run --rm -it alpine sh

# Run with resource limits
podman run -d --name webapp \
  --memory="512m" \
  --cpus="1.5" \
  nginx

# Run with restart policy
podman run -d --name app \
  --restart=always \
  nginx
```

#### Port Mapping
```bash
# Map single port
podman run -d -p 8080:80 --name web nginx

# Map multiple ports
podman run -d \
  -p 8080:80 \
  -p 8443:443 \
  --name web nginx

# Map to specific interface
podman run -d -p 127.0.0.1:8080:80 --name web nginx

# Random host port
podman run -d -p 80 --name web nginx

# UDP port
podman run -d -p 53:53/udp --name dns nginx
```

#### Environment Variables
```bash
# Single environment variable
podman run -d -e DATABASE_HOST=db.example.com nginx

# Multiple variables
podman run -d \
  -e DB_HOST=localhost \
  -e DB_PORT=5432 \
  -e DB_NAME=myapp \
  nginx

# From file
podman run -d --env-file ./env.list nginx

# env.list format:
# DB_HOST=localhost
# DB_PORT=5432
```

#### Volume Mounts
```bash
# Bind mount (absolute path required)
podman run -d -v /host/path:/container/path:Z nginx

# Named volume
podman run -d -v mydata:/var/lib/data:Z nginx

# Read-only mount
podman run -d -v /host/path:/container/path:ro,Z nginx

# Multiple mounts
podman run -d \
  -v /data:/app/data:Z \
  -v /logs:/app/logs:Z \
  nginx
```

### Managing Containers

#### List Containers
```bash
# List running containers
podman ps

# List all containers (including stopped)
podman ps -a

# List with custom format
podman ps --format "{{.ID}}\t{{.Names}}\t{{.Status}}"

# List container IDs only
podman ps -q

# Filter containers
podman ps --filter "status=running"
podman ps --filter "name=web"
podman ps --filter "label=env=production"
```

#### Start/Stop/Restart
```bash
# Start container
podman start <container-name-or-id>

# Stop container (SIGTERM, 10 sec timeout)
podman stop <container-name-or-id>

# Stop with custom timeout
podman stop -t 30 <container-name-or-id>

# Restart container
podman restart <container-name-or-id>

# Pause container
podman pause <container-name-or-id>

# Unpause container
podman unpause <container-name-or-id>

# Kill container (SIGKILL)
podman kill <container-name-or-id>

# Kill with specific signal
podman kill -s SIGHUP <container-name-or-id>
```

#### Execute Commands
```bash
# Execute command in running container
podman exec <container> ls -la

# Interactive shell
podman exec -it <container> bash

# Execute as specific user
podman exec --user nginx <container> whoami

# Execute with environment variables
podman exec -e VAR=value <container> env

# Execute with working directory
podman exec -w /app <container> ls
```

#### Container Logs
```bash
# View logs
podman logs <container>

# Follow logs
podman logs -f <container>

# Show timestamps
podman logs -t <container>

# Show last N lines
podman logs --tail 100 <container>

# Show logs since timestamp
podman logs --since 2026-03-01T10:00:00 <container>

# Show logs for time range
podman logs --since 1h <container>
```

#### Inspect and Stats
```bash
# Inspect container
podman inspect <container>

# Get specific value
podman inspect --format '{{.NetworkSettings.IPAddress}}' <container>

# Container statistics
podman stats <container>

# All containers stats
podman stats --all

# One-time stats (no stream)
podman stats --no-stream <container>

# Top processes in container
podman top <container>

# Top with custom format
podman top <container> user pid ppid %cpu %mem vsz rss
```

#### Remove Containers
```bash
# Remove stopped container
podman rm <container>

# Force remove running container
podman rm -f <container>

# Remove with volumes
podman rm -v <container>

# Remove all stopped containers
podman container prune

# Remove all containers (DANGEROUS)
podman rm -f $(podman ps -aq)
```

### Container Lifecycle Management

#### Save and Export
```bash
# Export container filesystem
podman export <container> > container.tar
podman export <container> -o container.tar

# Import as image
podman import container.tar myimage:tag

# Save container as checkpoint
podman container checkpoint <container>

# Restore from checkpoint
podman container restore <container>
```

#### Commit Changes
```bash
# Create image from container
podman commit <container> mynewimage:v1

# Commit with message and author
podman commit \
  --message "Added custom config" \
  --author "admin@example.com" \
  <container> myimage:v2

# Commit with changes
podman commit \
  --change "ENV DEBUG=true" \
  --change "EXPOSE 8080" \
  <container> myimage:v3
```

---

## Image Management

### Pulling Images

```bash
# Pull image from default registry
podman pull nginx

# Pull from Docker Hub
podman pull rockylinux:9

# Pull specific tag
podman pull nginx:1.24-alpine

# Pull by digest
podman pull nginx@sha256:abc123...

# Pull all tags
podman pull --all-tags nginx

# Pull for specific platform
podman pull --platform linux/arm64 nginx
```

### Listing and Searching

```bash
# List local images
podman images

# List with digests
podman images --digests

# List image IDs only
podman images -q

# Filter images
podman images --filter "dangling=true"
podman images --filter "reference=nginx:*"
podman images --filter "before=nginx:latest"

# Search registries
podman search nginx

# Search with filters
podman search --filter=is-official=true nginx
podman search --limit 10 nginx

# Search specific registry
podman search registry.access.redhat.com/rhel
```

### Building Images

**Note**: For comprehensive Containerfile documentation, see [Containerfile.md](Containerfile.md)

#### Basic Build Commands
```bash
# Build from Containerfile
podman build -t myapp:v1 .

# Build with specific file
podman build -f Containerfile.prod -t myapp:prod .

# Build with build arguments
podman build \
  --build-arg VERSION=1.0 \
  --build-arg ENV=production \
  -t myapp:v1 .

# Build without cache
podman build --no-cache -t myapp:v1 .

# Build for specific platform
podman build --platform linux/amd64 -t myapp:v1 .

# Multi-stage build
podman build --target production -t myapp:prod .

# Build with secrets
podman build --secret id=mysecret,src=secret.txt -t myapp:v1 .
```

#### Simple Containerfile Example
```dockerfile
# Simple example - see Containerfile.md for comprehensive guide
FROM rockylinux:9
RUN dnf install -y httpd && dnf clean all
COPY index.html /var/www/html/
EXPOSE 8080
CMD ["/usr/sbin/httpd", "-D", "FOREGROUND"]
```

### Tagging Images

```bash
# Tag image
podman tag myapp:v1 myapp:latest

# Tag for registry
podman tag myapp:v1 registry.example.com/myapp:v1

# Multiple tags
podman tag myapp:v1 myapp:stable
podman tag myapp:v1 myapp:production
```

### Pushing Images

```bash
# Push to registry
podman push myapp:v1 docker://registry.example.com/myapp:v1

# Push with credentials
podman login registry.example.com
podman push myapp:v1 registry.example.com/myapp:v1

# Push all tags
podman push --all-tags myapp registry.example.com/myapp

# Push without compression
podman push --compress=false myapp:v1 registry.example.com/myapp:v1
```

### Removing Images

```bash
# Remove image
podman rmi myapp:v1

# Force remove
podman rmi -f myapp:v1

# Remove by ID
podman rmi abc123

# Remove dangling images
podman image prune

# Remove all unused images
podman image prune -a

# Remove with filter
podman image prune --filter "until=24h"
```

### Image Inspection

```bash
# Inspect image
podman inspect myapp:v1

# View image history
podman history myapp:v1

# View image layers as tree
podman image tree myapp:v1

# Get image size
podman images --format "{{.Repository}}:{{.Tag}} {{.Size}}"

# Export image
podman save myapp:v1 -o myapp.tar
podman save myapp:v1 | gzip > myapp.tar.gz

# Load image
podman load -i myapp.tar
```

---

## Networking

### Network Types
- **bridge**: Default network mode (CNI bridge)
- **host**: Use host network namespace
- **none**: No networking
- **container**: Share network with another container
- **slirp4netns**: User-mode networking (rootless default)

### Network Management

#### List Networks
```bash
# List networks
podman network ls

# Inspect network
podman network inspect bridge

# List with format
podman network ls --format "{{.Name}} {{.Driver}}"
```

#### Create Networks
```bash
# Create simple network
podman network create mynet

# Create with subnet
podman network create \
  --subnet 10.89.0.0/24 \
  --gateway 10.89.0.1 \
  mynet

# Create with multiple subnets
podman network create \
  --subnet 10.89.0.0/24 \
  --gateway 10.89.0.1 \
  --ip-range 10.89.0.0/28 \
  mynet

# Create with DNS
podman network create \
  --dns 8.8.8.8 \
  --dns 8.8.4.4 \
  mynet

# Create with options
podman network create \
  --opt com.docker.network.bridge.name=mybr0 \
  --label environment=production \
  mynet
```

#### Remove Networks
```bash
# Remove network
podman network rm mynet

# Force remove (disconnect containers first)
podman network rm -f mynet

# Remove unused networks
podman network prune
```

### Container Networking

```bash
# Use specific network
podman run -d --network mynet --name web nginx

# Use host network
podman run -d --network host --name web nginx

# Disable networking
podman run -d --network none --name web nginx

# Share network with container
podman run -d --network container:web --name app myapp

# Connect to multiple networks
podman run -d --network net1 --network net2 --name web nginx

# Set hostname
podman run -d --hostname webserver --name web nginx

# Add DNS server
podman run -d --dns 8.8.8.8 --name web nginx

# Add DNS search domain
podman run -d --dns-search example.com --name web nginx

# Add hosts entry
podman run -d --add-host db:10.0.0.5 --name web nginx

# Set MAC address
podman run -d --mac-address 02:42:ac:11:00:02 --name web nginx

# Static IP (requires custom network)
podman network create --subnet 10.89.0.0/24 mynet
podman run -d --network mynet --ip 10.89.0.10 --name web nginx
```

### Network Operations

```bash
# Connect running container to network
podman network connect mynet web

# Connect with IP
podman network connect --ip 10.89.0.10 mynet web

# Connect with alias
podman network connect --alias webapp mynet web

# Disconnect from network
podman network disconnect mynet web

# Inspect container networking
podman inspect -f '{{.NetworkSettings.Networks}}' web

# Get container IP
podman inspect -f '{{.NetworkSettings.IPAddress}}' web

# Test connectivity
podman exec web ping -c 3 app
```

### DNS and Service Discovery

```bash
# Containers on same network can resolve by name
podman network create mynet
podman run -d --network mynet --name db postgres
podman run -d --network mynet --name web nginx

# From web container
podman exec web ping db  # Works!

# Add network aliases
podman run -d --network mynet --network-alias database --name db postgres
podman exec web ping database  # Works!
```

---

## Volumes and Storage

### Volume Types
- **Named volumes**: Managed by Podman
- **Bind mounts**: Direct host directory mapping
- **tmpfs**: Temporary filesystem in memory

### Named Volumes

```bash
# Create volume
podman volume create mydata

# Create with options
podman volume create \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/path/to/export \
  nfsdata

# List volumes
podman volume ls

# Inspect volume
podman volume inspect mydata

# Get volume mount point
podman volume inspect -f '{{.Mountpoint}}' mydata

# Remove volume
podman volume rm mydata

# Remove unused volumes
podman volume prune

# Remove all volumes (DANGEROUS)
podman volume rm $(podman volume ls -q)
```

### Using Volumes

```bash
# Mount named volume
podman run -d -v mydata:/var/lib/data --name app nginx

# Mount with Z option (SELinux relabel)
podman run -d -v mydata:/var/lib/data:Z --name app nginx

# Mount read-only
podman run -d -v mydata:/var/lib/data:ro,Z --name app nginx

# Mount with specific options
podman run -d -v mydata:/data:Z,U --name app nginx
# U = change ownership to container user

# Multiple volumes
podman run -d \
  -v data1:/app/data:Z \
  -v data2:/app/cache:Z \
  --name app nginx
```

### Bind Mounts

```bash
# Simple bind mount
podman run -d -v /host/path:/container/path:Z --name app nginx

# Read-only bind mount
podman run -d -v /host/path:/container/path:ro,Z --name app nginx

# Bind mount current directory
podman run -d -v $(pwd):/app:Z --name app nginx

# Multiple bind mounts
podman run -d \
  -v /etc/config:/app/config:ro,Z \
  -v /var/log/app:/app/logs:Z \
  --name app nginx

# SELinux contexts
# :z - shared between containers
# :Z - private to container
podman run -d -v /data:/app/data:z --name app1 nginx
podman run -d -v /data:/app/data:z --name app2 nginx
```

### Tmpfs Mounts

```bash
# Mount tmpfs
podman run -d --tmpfs /tmp:rw,size=100m,mode=1777 --name app nginx

# Multiple tmpfs mounts
podman run -d \
  --tmpfs /tmp:rw,noexec,nosuid,size=1g \
  --tmpfs /run:rw,noexec,nosuid,size=512m \
  --name app nginx
```

### Volume Backup and Restore

```bash
# Backup volume to tar
podman run --rm \
  -v mydata:/data:Z \
  -v $(pwd):/backup:Z \
  alpine \
  tar czf /backup/mydata-backup.tar.gz -C /data .

# Restore volume from tar
podman run --rm \
  -v mydata:/data:Z \
  -v $(pwd):/backup:Z \
  alpine \
  tar xzf /backup/mydata-backup.tar.gz -C /data

# Copy between volumes
podman run --rm \
  -v source-vol:/source:Z \
  -v dest-vol:/dest:Z \
  alpine \
  sh -c "cp -a /source/. /dest/"
```

### Storage Drivers

```bash
# Check storage driver
podman info --format '{{.Store.GraphDriverName}}'

# Storage info
podman system df

# Detailed storage info
podman system df -v

# Clean up storage
podman system prune

# Aggressive cleanup
podman system prune -a --volumes
```

---

## Pods

### Understanding Pods
Pods are groups of containers that share network and optionally storage. Similar to Kubernetes pods.

### Pod Operations

#### Create Pods
```bash
# Create empty pod
podman pod create --name mypod

# Create with port mapping
podman pod create --name web-pod -p 8080:80

# Create with hostname
podman pod create --name app-pod --hostname appserver

# Create with network
podman pod create --name db-pod --network mynet

# Create with shared IPC
podman pod create --name app-pod --share ipc

# Create with infra container options
podman pod create --name app-pod \
  --infra-name app-infra \
  --infra-image rockylinux:9-minimal
```

#### List Pods
```bash
# List pods
podman pod ps

# List all pods (including stopped)
podman pod ps -a

# List with custom format
podman pod ps --format "{{.Name}} {{.Status}} {{.NumberOfContainers}}"

# Show pod IDs only
podman pod ps -q
```

#### Add Containers to Pod
```bash
# Add container to pod
podman run -d --pod mypod --name web nginx
podman run -d --pod mypod --name app myapp

# Containers in same pod share:
# - Network namespace (localhost communication)
# - IPC namespace
# - UTS namespace (hostname)

# Example: nginx + app in same pod
podman pod create --name webapp -p 8080:80
podman run -d --pod webapp --name nginx nginx
podman run -d --pod webapp --name app myapp:latest
```

#### Manage Pods
```bash
# Start pod (starts all containers)
podman pod start mypod

# Stop pod (stops all containers)
podman pod stop mypod

# Restart pod
podman pod restart mypod

# Pause pod
podman pod pause mypod

# Unpause pod
podman pod unpause mypod

# Kill pod
podman pod kill mypod

# Inspect pod
podman pod inspect mypod

# Get pod stats
podman pod stats mypod

# View pod logs
podman pod logs mypod

# Top processes in pod
podman pod top mypod
```

#### Remove Pods
```bash
# Remove pod (must stop first)
podman pod rm mypod

# Force remove running pod
podman pod rm -f mypod

# Remove pod and containers
podman pod rm -f mypod

# Remove all stopped pods
podman pod prune
```

### Pod YAML Generation

```bash
# Generate Kubernetes YAML from pod
podman generate kube mypod > mypod.yaml

# Generate with service
podman generate kube mypod --service > mypod.yaml

# Play Kubernetes YAML
podman play kube mypod.yaml

# Play with custom name
podman play kube --name custom-name mypod.yaml

# Remove resources created by play
podman play kube --down mypod.yaml
```

### Multi-Container Pod Example

```bash
# Create web application pod
podman pod create --name webapp -p 8080:80

# Add nginx reverse proxy
podman run -d \
  --pod webapp \
  --name nginx \
  -v ./nginx.conf:/etc/nginx/nginx.conf:ro,Z \
  nginx

# Add application
podman run -d \
  --pod webapp \
  --name app \
  -e DATABASE_URL=localhost:5432 \
  myapp:latest

# Add database
podman run -d \
  --pod webapp \
  --name db \
  -v dbdata:/var/lib/postgresql/data:Z \
  postgres:15

# Containers communicate via localhost
# app connects to db at localhost:5432
# nginx proxies to app at localhost:8000
```

---

## Rootless Containers

### Understanding Rootless Mode
Rootless containers run without root privileges, enhancing security. Available by default for regular users in RHEL 8/9/10.

### Setup Rootless

```bash
# Check user namespaces enabled
cat /proc/sys/user/max_user_namespaces  # Should be > 0

# Check subuid/subgid configuration
cat /etc/subuid
cat /etc/subgid

# Example output:
# username:100000:65536

# Enable lingering (keep user session after logout)
loginctl enable-linger $USER

# Start user socket
systemctl --user enable --now podman.socket

# Verify
systemctl --user status podman.socket
```

### Rootless Commands

```bash
# All podman commands work without sudo as regular user
podman run -d --name web nginx
podman ps
podman images

# User-specific locations
~/.local/share/containers/storage/  # Images and containers
~/.config/containers/               # Configuration

# Check rootless info
podman info | grep -i rootless
```

### Rootless Networking

```bash
# Default: slirp4netns (user-mode networking)
podman run -d -p 8080:80 --name web nginx

# Accessible on localhost:8080

# Use pasta for better performance (RHEL 9+)
podman run -d --network pasta -p 8080:80 --name web nginx

# Create rootless network
podman network create mynet

# Use network
podman run -d --network mynet --name web nginx
```

### Rootless Limitations

```bash
# Cannot bind to privileged ports (<1024) by default
# Workaround: Port forwarding
podman run -d -p 8080:80 nginx  # Access via 8080

# Or allow user to bind privileged ports (RHEL 9+)
sudo sysctl net.ipv4.ip_unprivileged_port_start=80

# Cannot use some volume drivers
# Workaround: Use bind mounts with appropriate permissions

# Limited resource control
# Workaround: Use systemd user services with resource limits
```

### Rootless vs Root

```bash
# Root containers (requires sudo)
sudo podman run -d --name root-web nginx
sudo podman ps

# Rootless containers (no sudo)
podman run -d --name user-web nginx
podman ps

# These are SEPARATE
# Root sees only root containers
# User sees only user containers

# List all (as root)
sudo podman ps -a
sudo podman --root ps -a  # Same

# List user containers from root (advanced)
sudo podman --runroot /run/user/1000/containers ps
```

---

## Security

### Security Features

#### SELinux Integration
```bash
# Check SELinux status
getenforce

# Run with SELinux enabled (default)
podman run -d --name web nginx

# Check container SELinux context
podman inspect --format '{{.ProcessLabel}}' web

# Disable SELinux for container (NOT RECOMMENDED)
podman run -d --security-opt label=disable --name web nginx

# Custom SELinux type
podman run -d --security-opt label=type:svirt_apache_t --name web nginx

# Volume SELinux contexts
# :z - shared label
# :Z - private label
podman run -d -v /data:/data:Z --name web nginx
```

#### Seccomp Profiles
```bash
# Default seccomp profile applied automatically

# Disable seccomp (NOT RECOMMENDED)
podman run -d --security-opt seccomp=unconfined --name web nginx

# Custom seccomp profile
podman run -d --security-opt seccomp=/path/to/profile.json --name web nginx

# Example profile.json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": ["read", "write", "open", "close"],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

#### Capabilities
```bash
# Drop all capabilities
podman run -d --cap-drop=ALL --name web nginx

# Add specific capabilities
podman run -d \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --name web nginx

# Drop specific capabilities
podman run -d --cap-drop=NET_RAW --name web nginx

# Run privileged (NOT RECOMMENDED)
podman run -d --privileged --name web nginx

# View container capabilities
podman exec web capsh --print
```

#### User Namespaces
```bash
# Remap to different user (rootless automatic)
podman run -d --userns=auto --name web nginx

# Run as specific user
podman run -d --user 1000:1000 --name web nginx

# Run as named user
podman run -d --user nginx --name web nginx

# Keep user namespace separate
podman run -d --userns=keep-id --name web nginx
```

#### Read-Only Containers
```bash
# Read-only root filesystem
podman run -d --read-only --name web nginx

# With tmpfs for writable directories
podman run -d \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=100m \
  --tmpfs /run:rw,noexec,nosuid,size=100m \
  --name web nginx
```

#### No New Privileges
```bash
# Prevent privilege escalation
podman run -d --security-opt=no-new-privileges --name web nginx
```

### Image Signing and Verification

```bash
# Sign image
podman push --sign-by user@example.com myimage:v1 registry.example.com/myimage:v1

# Configure policy
sudo vi /etc/containers/policy.json

{
  "default": [{"type": "reject"}],
  "transports": {
    "docker": {
      "registry.example.com": [
        {
          "type": "signedBy",
          "keyType": "GPGKeys",
          "keyPath": "/path/to/pubkey.gpg"
        }
      ]
    }
  }
}

# Pull with signature verification
podman pull registry.example.com/myimage:v1
```

### Scanning Images

```bash
# Install scanner
sudo dnf install -y clamav clamav-update

# Scan image
podman save myimage:v1 | clamscan -

# Or use Anchore/Trivy (install separately)
trivy image myimage:v1

# Scan for vulnerabilities
skopeo inspect --config docker://myimage:v1
```

---

## Systemd Integration

### Generate Systemd Units

```bash
# Generate unit for container
podman generate systemd --name myapp > ~/.config/systemd/user/container-myapp.service

# Generate with restart policy
podman generate systemd --name myapp --restart-policy=always > myapp.service

# Generate for pod
podman generate systemd --name mypod --files

# Generate with new flag (create container at start)
podman create --name myapp nginx
podman generate systemd --new --name myapp > myapp.service

# Install unit
mkdir -p ~/.config/systemd/user/
mv myapp.service ~/.config/systemd/user/

# Reload systemd
systemctl --user daemon-reload

# Enable and start
systemctl --user enable --now container-myapp.service
```

### Systemd Service Example

```bash
# Create container first
podman create \
  --name webapp \
  -p 8080:80 \
  -v webdata:/var/www/html:Z \
  --restart=always \
  nginx

# Generate service
podman generate systemd --new --name webapp | \
  tee ~/.config/systemd/user/container-webapp.service

# Enable linger
loginctl enable-linger $USER

# Reload and enable
systemctl --user daemon-reload
systemctl --user enable --now container-webapp.service

# Check status
systemctl --user status container-webapp.service

# View logs
journalctl --user -u container-webapp.service -f
```

### Quadlet (RHEL 9.2+)

Simplified systemd integration using .container files.

```bash
# Create container unit
mkdir -p ~/.config/containers/systemd/

cat > ~/.config/containers/systemd/webapp.container << 'EOF'
[Unit]
Description=Web Application
After=network-online.target

[Container]
Image=nginx:latest
PublishPort=8080:80
Volume=webdata:/var/www/html:Z
Environment=ENV=production

[Service]
Restart=always
TimeoutStartSec=900

[Install]
WantedBy=multi-user.target default.target
EOF

# Reload systemd
systemctl --user daemon-reload

# Start container
systemctl --user start webapp.service

# Enable on boot
systemctl --user enable webapp.service
```

### System Services (Root)

```bash
# Create as root
sudo podman create \
  --name system-web \
  -p 80:80 \
  nginx

# Generate system unit
sudo podman generate systemd --new --name system-web | \
  sudo tee /etc/systemd/system/container-system-web.service

# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable --now container-system-web.service
```

---

## Multi-Container Applications

**Note**: For comprehensive Podman Compose documentation, see [PodmanCompose.md](PodmanCompose.md)

Podman Compose allows you to define and run multi-container applications using docker-compose.yml files.

### Quick Start

```bash
# Install podman-compose
sudo dnf install -y podman-compose

# Start services from compose file
podman-compose up -d

# View logs
podman-compose logs -f

# Stop services
podman-compose down
```

### Simple Example

```yaml
version: '3'

services:
  web:
    image: nginx:latest
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html:Z

  app:
    image: myapp:latest
    environment:
      - DATABASE_URL=postgresql://db:5432/myapp
    depends_on:
      - db

  db:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=secret
    volumes:
      - dbdata:/var/lib/postgresql/data:Z

volumes:
  dbdata:
```

For complete compose file syntax, advanced features, and real-world examples, see [PodmanCompose.md](PodmanCompose.md).

---

## Registry Management

### Working with Registries

#### Login and Authentication
```bash
# Login to registry
podman login registry.example.com

# Login with credentials
podman login -u username -p password registry.example.com

# Login to Red Hat registry
podman login registry.redhat.io

# Login to Quay.io
podman login quay.io

# Login with token
echo $TOKEN | podman login --username oauth2accesstoken --password-stdin registry.example.com

# Logout
podman logout registry.example.com

# View credentials
cat ~/.config/containers/auth.json
```

#### Registry Configuration
```bash
# Edit registries configuration
vi ~/.config/containers/registries.conf

# Add registries
[registries.search]
registries = ['registry.access.redhat.com', 'docker.io', 'quay.io']

[registries.insecure]
registries = ['localhost:5000']

[[registry]]
location = "registry.example.com"
insecure = false
blocked = false

[[registry.mirror]]
location = "mirror.example.com"
```

### Local Registry

```bash
# Run local registry
podman run -d \
  -p 5000:5000 \
  --name registry \
  -v registry-data:/var/lib/registry:Z \
  --restart=always \
  registry:2

# Tag image for local registry
podman tag myapp:v1 localhost:5000/myapp:v1

# Push to local registry
podman push localhost:5000/myapp:v1

# Pull from local registry
podman pull localhost:5000/myapp:v1

# List images in local registry
curl -X GET http://localhost:5000/v2/_catalog
curl -X GET http://localhost:5000/v2/myapp/tags/list
```

### Image Transfer

```bash
# Copy between registries using skopeo
skopeo copy \
  docker://registry1.example.com/myapp:v1 \
  docker://registry2.example.com/myapp:v1

# Copy with authentication
skopeo copy \
  --src-creds user:pass \
  --dest-creds user:pass \
  docker://source/image:tag \
  docker://dest/image:tag

# Copy to local directory
skopeo copy docker://nginx:latest dir:/tmp/nginx

# Copy from tar
podman save myapp:v1 | skopeo copy docker-archive:/dev/stdin docker://registry.example.com/myapp:v1
```

---

## Monitoring and Logs

### Container Monitoring

```bash
# Real-time stats
podman stats

# Specific containers
podman stats web app db

# One-time stats
podman stats --no-stream

# Custom format
podman stats --format "{{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}"

# JSON output
podman stats --format json

# Resource usage
podman container inspect --format '{{.HostConfig.Memory}}' web
```

### System Monitoring

```bash
# System information
podman system info

# Disk usage
podman system df

# Detailed disk usage
podman system df -v

# Show events
podman events

# Filter events
podman events --filter container=web
podman events --filter event=start
podman events --since 2026-03-01

# JSON output
podman events --format json
```

### Logging

```bash
# View logs
podman logs web

# Follow logs
podman logs -f web

# Timestamps
podman logs -t web

# Last N lines
podman logs --tail 100 web

# Since timestamp
podman logs --since 2026-03-01T10:00:00 web
podman logs --since 1h web

# Until timestamp
podman logs --until 2026-03-01T23:59:59 web

# Multiple containers
podman logs web app db
```

### Log Drivers

```bash
# Default: json-file

# Use journald
podman run -d \
  --log-driver=journald \
  --name web \
  nginx

# View with journalctl
journalctl CONTAINER_NAME=web -f

# No logging
podman run -d --log-driver=none --name web nginx

# Configure log rotation
podman run -d \
  --log-driver=json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  --name web \
  nginx
```

### Health Checks

```bash
# Define health check in Containerfile
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost/ || exit 1

# Add health check at runtime
podman run -d \
  --health-cmd='curl -f http://localhost/ || exit 1' \
  --health-interval=30s \
  --health-timeout=3s \
  --health-retries=3 \
  --name web \
  nginx

# Check health status
podman inspect --format='{{.State.Health.Status}}' web

# View health check logs
podman inspect --format='{{json .State.Health}}' web | jq
```

---

## Troubleshooting

### Common Issues

#### Container Won't Start
```bash
# Check logs
podman logs container-name

# Inspect container
podman inspect container-name

# Check events
podman events --filter container=container-name

# Try running interactively
podman run -it --rm image:tag sh

# Check resource limits
podman inspect --format='{{.HostConfig.Memory}}' container-name
```

#### Network Issues
```bash
# Check container network settings
podman inspect --format='{{.NetworkSettings}}' container-name

# Get IP address
podman inspect --format='{{.NetworkSettings.IPAddress}}' container-name

# Test connectivity from host
ping $(podman inspect --format='{{.NetworkSettings.IPAddress}}' container-name)

# Test from inside container
podman exec container-name ping -c 3 8.8.8.8
podman exec container-name nslookup google.com

# Check firewall
sudo firewall-cmd --list-all

# Check network
podman network ls
podman network inspect bridge
```

#### Port Binding Issues
```bash
# Check port binding
podman port container-name

# List what's using port
sudo ss -tulpn | grep :8080

# Verify firewall allows port
sudo firewall-cmd --list-ports
sudo firewall-cmd --add-port=8080/tcp

# Try different port
podman run -d -p 8081:80 --name web nginx
```

#### Permission Issues
```bash
# Check SELinux denials
sudo ausearch -m avc -ts recent | grep podman

# Check file permissions
ls -lZ /path/to/file

# Relabel volume
podman unshare chown -R 0:0 /path/to/volume
podman run -d -v /path:/data:Z --name app myapp

# Check subuid/subgid (rootless)
cat /etc/subuid
cat /etc/subgid

# Recreate user mappings
podman system migrate
```

#### Storage Issues
```bash
# Check storage
podman system df

# Clean up
podman system prune -a

# Reset storage (DELETES EVERYTHING)
podman system reset

# Check storage driver
podman info --format '{{.Store.GraphDriverName}}'

# Storage location
podman info --format '{{.Store.GraphRoot}}'

# Fix storage issues
podman system migrate
```

#### Image Pull Issues
```bash
# Check registry configuration
cat ~/.config/containers/registries.conf

# Test registry connectivity
curl -v https://registry.example.com/v2/

# Pull with full path
podman pull docker.io/library/nginx:latest

# Check credentials
cat ~/.config/containers/auth.json

# Login again
podman login registry.example.com

# Try insecure (testing only)
podman pull --tls-verify=false localhost:5000/image:tag
```

### Debugging Containers

```bash
# Run container with shell
podman run -it --rm image:tag sh

# Override entrypoint
podman run -it --rm --entrypoint /bin/bash image:tag

# Attach to running container
podman attach container-name

# Debug with ephemeral container
podman exec -it container-name /bin/bash

# Copy files from container
podman cp container-name:/path/to/file ./local/path

# Copy files to container
podman cp ./local/file container-name:/path/to/

# Check process tree
podman top container-name

# Inspect full config
podman inspect container-name | less
```

### System Troubleshooting

```bash
# Check Podman version
podman version

# System information
podman system info

# Check for problems
podman system check

# Reset everything (NUCLEAR OPTION)
podman system reset

# View Podman logs (rootless)
journalctl --user -u podman -f

# View Podman logs (root)
sudo journalctl -u podman -f

# Check cgroup version
podman info --format '{{.Host.CgroupsVersion}}'

# Increase verbosity
podman --log-level debug run -it alpine sh
```

---

## Migration from Docker

### Docker Compatibility

```bash
# Install docker command alias
sudo dnf install -y podman-docker

# Creates docker symlink to podman
which docker  # -> /usr/bin/docker -> podman

# Most docker commands work
docker ps
docker run -d nginx
docker images
```

### Command Equivalents

| Docker Command | Podman Equivalent | Notes |
|---------------|-------------------|-------|
| `docker run` | `podman run` | Identical |
| `docker ps` | `podman ps` | Identical |
| `docker images` | `podman images` | Identical |
| `docker build` | `podman build` | Uses Containerfile or Dockerfile |
| `docker-compose` | `podman-compose` | Separate package |
| `docker exec` | `podman exec` | Identical |
| `docker logs` | `podman logs` | Identical |
| `docker pull/push` | `podman pull/push` | Identical |
| `docker network` | `podman network` | Similar |
| `docker volume` | `podman volume` | Identical |

### Migration Steps

```bash
# 1. Export Docker containers
docker export container-name > container.tar

# 2. Import to Podman
podman import container.tar image-name:tag

# Or save images
docker save image:tag > image.tar
podman load < image.tar

# 3. Convert docker-compose.yml
# Works as-is with podman-compose
podman-compose -f docker-compose.yml up -d

# 4. Migrate volumes
# Copy data from Docker volume to Podman volume
docker run -v docker-vol:/data alpine tar c -C /data . | \
  podman run -i -v podman-vol:/data alpine tar x -C /data

# 5. Update scripts
# Replace 'docker' with 'podman' or use alias
alias docker=podman
```

### Key Differences

```bash
# No daemon in Podman
# Docker: systemctl status docker
# Podman: No daemon to manage

# Rootless by default
# Docker: Usually requires sudo
# Podman: Works as regular user

# Pods are native
# Docker: Uses Compose/Swarm
# Podman: Built-in pod support

# Systemd integration
# Docker: Requires docker.service
# Podman: Native systemd units

# Socket activation
# Docker: Docker socket
# Podman: Optional socket per user
systemctl --user enable --now podman.socket
```

---

## Best Practices

### Security Best Practices

1. **Run rootless containers when possible**
```bash
# Default for regular users
podman run -d nginx
```

2. **Use specific image tags**
```bash
# BAD: Uses latest
podman run nginx

# GOOD: Specific version
podman run nginx:1.24.0-alpine
```

3. **Don't run as root inside containers**
```bash
# In Containerfile
USER 1001

# Or at runtime
podman run --user 1001:1001 nginx
```

4. **Use read-only root filesystem**
```bash
podman run -d --read-only --tmpfs /tmp nginx
```

5. **Drop unnecessary capabilities**
```bash
podman run -d --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx
```

6. **Scan images for vulnerabilities**
```bash
skopeo inspect --config docker://image:tag
trivy image image:tag
```

### Performance Best Practices

1. **Use overlay storage driver** (default in RHEL)
2. **Clean up regularly**
```bash
podman system prune -a
podman volume prune
```

3. **Use multi-stage builds**
4. **Minimize layers in images**
5. **Use .containerignore file**

### Operational Best Practices

1. **Use systemd for production services**
2. **Implement health checks**
3. **Set resource limits**
```bash
podman run -d \
  --memory="512m" \
  --cpus="1.0" \
  --pids-limit=100 \
  nginx
```

4. **Use named volumes instead of bind mounts**
5. **Tag images properly**
```bash
image:v1.0.0
image:v1.0
image:latest
```

6. **Document with labels**
```bash
LABEL version="1.0"
LABEL description="My application"
LABEL maintainer="admin@example.com"
```

### Maintenance Tasks

```bash
# Weekly: Clean up unused resources
podman system prune -a

# Monthly: Update base images
podman pull rockylinux:9
podman build -t myapp:latest .

# Check for updates
podman auto-update

# Review logs
journalctl --user -u container-*.service --since yesterday

# Backup volumes
for vol in $(podman volume ls -q); do
  podman run --rm -v $vol:/data -v $(pwd):/backup alpine \
    tar czf /backup/$vol-$(date +%Y%m%d).tar.gz -C /data .
done
```

---

## Quick Reference

### Essential Commands
```bash
# Containers
podman run -d --name app image:tag
podman ps
podman stop app
podman rm app
podman logs -f app
podman exec -it app bash

# Images
podman pull image:tag
podman images
podman build -t myapp:v1 .
podman rmi image:tag
podman push image:tag registry/image:tag

# Networks
podman network create mynet
podman network ls
podman run -d --network mynet --name app image

# Volumes
podman volume create mydata
podman volume ls
podman run -d -v mydata:/data:Z --name app image

# Pods
podman pod create --name mypod -p 8080:80
podman run -d --pod mypod --name web nginx
podman pod ps

# System
podman system info
podman system df
podman system prune -a
```

### Useful Aliases
```bash
alias p='podman'
alias pps='podman ps'
alias psa='podman ps -a'
alias pi='podman images'
alias prm='podman rm'
alias prmi='podman rmi'
alias plog='podman logs'
alias pexec='podman exec -it'
alias pbuild='podman build'
alias ppull='podman pull'
alias ppush='podman push'
```

### Configuration Locations
```bash
# Root
/etc/containers/storage.conf
/etc/containers/registries.conf
/etc/containers/policy.json
/var/lib/containers/storage/

# Rootless
~/.config/containers/storage.conf
~/.config/containers/registries.conf
~/.local/share/containers/storage/
```

### Getting Help
```bash
# General help
podman --help

# Command help
podman run --help
podman build --help

# Man pages
man podman
man podman-run
man podman-build

# Version info
podman version
podman info
```