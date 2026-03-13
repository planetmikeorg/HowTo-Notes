# Rancher CLI Guide

A practical guide for working with Rancher from the command line.

## Table of Contents
- [Overview](#overview)
- [Install the CLI](#install-the-cli)
- [Authenticate](#authenticate)
- [Contexts and Configuration](#contexts-and-configuration)
- [Cluster Operations](#cluster-operations)
- [Projects and Namespaces](#projects-and-namespaces)
- [Applications and Catalogs](#applications-and-catalogs)
- [Nodes and Workloads](#nodes-and-workloads)
- [Users, Roles, and Settings](#users-roles-and-settings)
- [Fleet and GitOps](#fleet-and-gitops)
- [Scripting Tips](#scripting-tips)
- [Troubleshooting](#troubleshooting)
- [References](#references)

---

## Overview

The `rancher` CLI is used to interact with Rancher Manager without relying on the web UI. Common tasks include:

- authenticating to a Rancher server
- switching contexts between managed clusters and projects
- retrieving kubeconfig files
- managing projects, namespaces, apps, and nodes
- automating Rancher operations in scripts and pipelines

> Rancher CLI behavior can vary slightly by Rancher version. Check `rancher --help` and the server version when a command behaves differently than expected.

---

## Install the CLI

### macOS
```bash
# Homebrew is the simplest option when available
brew install rancher-cli

# Verify installation
rancher --version
```

### Linux
```bash
# Download a release tarball from GitHub
wget https://github.com/rancher/cli/releases/download/<version>/rancher-linux-amd64-<version>.tar.gz

# Extract it
tar -xzf rancher-linux-amd64-<version>.tar.gz

# Move the binary into your PATH
sudo mv rancher-<version>/rancher /usr/local/bin/

# Verify installation
rancher --version
```

### Windows
```powershell
# Download the matching release from GitHub
# https://github.com/rancher/cli/releases

# Extract rancher.exe and place it in a directory on PATH
rancher --version
```

### Upgrade
```bash
# If installed with Homebrew
brew upgrade rancher-cli

# Or replace the binary with a newer release tarball
rancher --version
```

---

## Authenticate

### Log In with an API Token
```bash
rancher login https://rancher.example.com --token <api-token>
```

### Log In with Environment Variables
```bash
export RANCHER_URL=https://rancher.example.com
export RANCHER_TOKEN=<api-token>

rancher login
```

### Verify Authentication
```bash
# View available settings from the server
rancher settings ls

# Show current context
rancher context current
```

### Create a Token
1. Sign in to Rancher.
2. Open your user menu.
3. Go to Account and API Keys.
4. Create a new API key.
5. Save the token securely. It is normally shown only once.

---

## Contexts and Configuration

### List and Switch Contexts
```bash
# List available contexts
rancher context ls

# Switch to a cluster or project context
rancher context switch <context-name>

# Show the active context
rancher context current
```

### Use Rancher-Managed kubectl
```bash
# Run kubectl through the Rancher CLI
rancher kubectl get nodes
rancher kubectl -n <namespace> get pods
```

### Export kubeconfig
```bash
# Download kubeconfig for a managed cluster
rancher clusters kubeconfig <cluster-id> > ~/.kube/rancher-<cluster-id>.yaml

# Use it temporarily
KUBECONFIG=~/.kube/rancher-<cluster-id>.yaml kubectl get nodes
```

### Useful Global Flags
```bash
# Show help
rancher --help

# Reduce output
rancher --quiet clusters ls

# Increase verbosity for troubleshooting
rancher --debug clusters ls
```

---

## Cluster Operations

### List and Inspect Clusters
```bash
# List clusters
rancher clusters ls

# JSON output for automation
rancher clusters ls --format json

# Get one cluster
rancher clusters get <cluster-id>

# Inspect raw details
rancher inspect <cluster-id>
```

### Create or Update a Cluster
```bash
# Example: create a cluster using an RKE config file
rancher clusters create <cluster-name> --rke-config cluster.yml

# Update cluster config
rancher clusters update <cluster-id> --rke-config cluster.yml

# Edit interactively
rancher clusters edit <cluster-id>
```

### Delete a Cluster
```bash
rancher clusters delete <cluster-id>
```

### Example `cluster.yml`
```yaml
nodes:
  - address: 192.168.1.10
    user: ubuntu
    role: [controlplane, etcd, worker]
  - address: 192.168.1.11
    user: ubuntu
    role: [worker]

kubernetes_version: "v1.29.0-rancher1-1"

network:
  plugin: canal
```

### Cluster Maintenance
```bash
# Create an etcd snapshot
rancher clusters etcd-snapshot <cluster-id> --name snapshot-001

# Restore from snapshot
rancher clusters etcd-restore <cluster-id> --snapshot-name snapshot-001

# Rotate certificates
rancher clusters rotate-certs <cluster-id>
```

---

## Projects and Namespaces

### Projects
```bash
# List projects
rancher projects ls

# Create a project
rancher projects create <project-name> \
  --cluster <cluster-id> \
  --description "Application project"

# Get project details
rancher projects get <project-id>
```

### Project Quotas
```bash
rancher projects set-quota <project-id> \
  --limit-cpu 10000m \
  --limit-memory 20Gi \
  --limit-pods 100
```

### Namespaces
```bash
# List namespaces
rancher namespaces ls

# Create a namespace in a project
rancher namespaces create <namespace-name> --project <project-id>

# Get namespace details
rancher namespaces get <namespace-name>

# Delete a namespace
rancher namespaces delete <namespace-name>
```

### Move a Namespace
```bash
rancher namespaces move <namespace-name> --project <project-id>
```

---

## Applications and Catalogs

### Catalog Operations
```bash
# List catalogs
rancher catalog ls

# Add a catalog
rancher catalog add <name> \
  --url https://charts.example.com \
  --branch main

# Refresh a catalog
rancher catalog refresh <name>

# Delete a catalog
rancher catalog delete <name>
```

### Application Operations
```bash
# List apps
rancher apps ls

# Install an app
rancher apps install <release-name> <catalog>/<chart> \
  --namespace <namespace> \
  --version <version>

# Install with values
rancher apps install <release-name> <catalog>/<chart> \
  --namespace <namespace> \
  --values values.yaml

# Upgrade an app
rancher apps upgrade <release-name> --version <version>

# Roll back an app
rancher apps rollback <release-name> --revision 1

# Delete an app
rancher apps delete <release-name>
```

---

## Nodes and Workloads

### Nodes
```bash
# List nodes
rancher nodes ls

# Get node details
rancher nodes get <node-id>

# Cordon a node
rancher nodes cordon <node-id>

# Drain a node
rancher nodes drain <node-id> \
  --ignore-daemonsets \
  --delete-emptydir-data

# Uncordon a node
rancher nodes uncordon <node-id>

# Delete a node
rancher nodes delete <node-id>
```

### Combined Rancher and kubectl Flow
```bash
# Switch Rancher context
rancher context switch <cluster-or-project>

# Use Rancher-backed kubectl commands
rancher kubectl get deploy -A
rancher kubectl describe pod <pod-name> -n <namespace>
rancher kubectl logs <pod-name> -n <namespace>
```

---

## Users, Roles, and Settings

### Users and Role Bindings
```bash
# List users
rancher users ls

# Create a user
rancher users create --username <username> --password <password>

# List global roles
rancher globalroles ls

# Bind a global role
rancher globalrolebindings create \
  --user <user-id> \
  --global-role admin

# List cluster roles
rancher clusterroles ls

# Bind a cluster role
rancher clusterrolebindings create \
  --user <user-id> \
  --cluster <cluster-id> \
  --cluster-role cluster-owner

# List project roles
rancher projectroles ls

# Bind a project role
rancher projectrolebindings create \
  --user <user-id> \
  --project <project-id> \
  --project-role project-member
```

### Settings
```bash
# List settings
rancher settings ls

# Get a setting
rancher settings get server-url

# Set a setting
rancher settings set server-url https://rancher.example.com
```

---

## Fleet and GitOps

### GitRepo Operations
```bash
# List GitRepos
rancher gitrepos ls

# Create a GitRepo
rancher gitrepos create <name> \
  --repo-url https://github.com/example/fleet-repo \
  --branch main \
  --paths clusters/production

# Update a GitRepo
rancher gitrepos update <name> --branch develop

# Force a sync
rancher gitrepos force-update <name>

# Delete a GitRepo
rancher gitrepos delete <name>
```

---

## Scripting Tips

### JSON Output with `jq`
```bash
# Show cluster names and states
rancher clusters ls --format json | jq '.[] | {name: .name, state: .state}'

# Capture a cluster ID by name
CLUSTER_ID=$(rancher clusters ls --format json | jq -r '.[] | select(.name=="production") | .id')

echo "$CLUSTER_ID"
```

### Loop Through Resources
```bash
rancher projects ls --format json | jq -r '.[].id' | while read -r project; do
  rancher projects get "$project"
done
```

### Use in CI/CD
```bash
export RANCHER_URL=https://rancher.example.com
export RANCHER_TOKEN=<api-token>

rancher login
rancher clusters ls --format json
```

Security notes:
- prefer short-lived or scoped tokens
- avoid echoing tokens into logs
- store secrets in the CI platform secret store

---

## Troubleshooting

### Basic Checks
```bash
# Confirm the CLI version
rancher --version

# Verify connectivity and auth
rancher settings ls

# Check active context
rancher context current
```

### Common Problems

**Login fails**
- verify the Rancher server URL
- verify the API token is still valid
- confirm your account has the needed permissions

**Context switch fails**
- run `rancher context ls` and confirm the target exists
- log in again if the local CLI cache is stale

**Commands differ from examples**
- compare your Rancher version with the CLI release
- run `rancher <command> --help` for the exact flags available

**kubectl via Rancher is not working**
- verify the selected Rancher context
- retrieve a kubeconfig directly with `rancher clusters kubeconfig`
- test access with plain `kubectl`

---

## References

- [Rancher CLI documentation](https://ranchermanager.docs.rancher.com/reference-guides/cli-with-rancher)
- [Rancher CLI releases](https://github.com/rancher/cli/releases)
- [Rancher CLI source](https://github.com/rancher/cli)
- [Rancher API reference](https://ranchermanager.docs.rancher.com/reference-guides/about-the-api)
