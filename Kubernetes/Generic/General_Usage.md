# Kubernetes Administration Cheatsheet

## Table of Contents
- [Cluster Access & Context](#cluster-access--context)
- [Cluster Inspection](#cluster-inspection)
- [Namespace Management](#namespace-management)
- [Workload Management](#workload-management)
- [Service & Ingress](#service--ingress)
- [Storage Management](#storage-management)
- [Observability](#observability)
- [RBAC & Security](#rbac--security)
- [Backup & Recovery](#backup--recovery)
- [Troubleshooting](#troubleshooting)
- [Quick Reference](#quick-reference)
- [Best Practices](#best-practices)

---

## Cluster Access & Context

### Verify Access
```bash
# Show the current context
kubectl config current-context

# Show cluster API endpoints
kubectl cluster-info

# Test API access
kubectl get --raw /healthz

# View client and server versions
kubectl version
```

### Manage kubeconfig Contexts
```bash
# List contexts
kubectl config get-contexts

# Switch contexts
kubectl config use-context <context-name>

# Rename a context
kubectl config rename-context <old-name> <new-name>

# Show merged kubeconfig
kubectl config view

# Flatten multiple kubeconfig files into one
KUBECONFIG=~/.kube/config:~/.kube/other-config \
  kubectl config view --flatten > ~/.kube/merged-config
mv ~/.kube/merged-config ~/.kube/config
```

### Discover API Resources
```bash
# List resource types supported by the cluster
kubectl api-resources

# List API versions
kubectl api-versions

# Show field documentation for a resource
kubectl explain deployment
kubectl explain deployment.spec.template.spec.containers
```

> Cluster creation, control plane upgrades, and node provisioning are platform-specific tasks. Use `kubectl` for day-to-day Kubernetes administration and workload operations.

---

## Cluster Inspection

### Cluster and Node State
```bash
# List nodes
kubectl get nodes
kubectl get nodes -o wide

# Describe a node
kubectl describe node <node-name>

# Show allocatable resources on each node
kubectl get nodes -o custom-columns=NAME:.metadata.name,CPU:.status.allocatable.cpu,MEMORY:.status.allocatable.memory,PODS:.status.allocatable.pods

# Show node readiness quickly
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'
```

### Core System Components
```bash
# Inspect system workloads
kubectl get pods -n kube-system

# View all namespaced resources in kube-system
kubectl get all -n kube-system

# Show events sorted by most recent
kubectl get events -A --sort-by='.metadata.creationTimestamp'
```

### General Resource Discovery
```bash
# List common resources in all namespaces
kubectl get pods,svc,deploy,ds,sts -A

# Get everything in one namespace
kubectl get all -n <namespace>

# Filter by label
kubectl get pods -n <namespace> -l app=<app-name>

# Filter by field selector
kubectl get pods -A --field-selector status.phase=Running
```

---

## Namespace Management

### Namespaces
```bash
# List namespaces
kubectl get namespaces

# Create a namespace
kubectl create namespace <namespace-name>

# Show namespace labels
kubectl get namespace <namespace-name> --show-labels

# Delete a namespace
kubectl delete namespace <namespace-name>
```

### Labels and Annotations
```bash
# Label a namespace
kubectl label namespace <namespace-name> environment=prod

# Update an existing label
kubectl label namespace <namespace-name> environment=stage --overwrite

# Annotate a namespace
kubectl annotate namespace <namespace-name> owner=platform-team

# Remove a label
kubectl label namespace <namespace-name> environment-
```

### Resource Quotas and Limits
```bash
# Create a namespace quota
kubectl create quota <quota-name> \
  --hard=cpu=10,memory=20Gi,pods=50 \
  -n <namespace>

# View quotas
kubectl get resourcequotas -n <namespace>
kubectl describe resourcequota <quota-name> -n <namespace>

# View LimitRanges
kubectl get limitranges -n <namespace>
kubectl describe limitrange <name> -n <namespace>
```

---

## Workload Management

### Deployments
```bash
# List deployments
kubectl get deployments -n <namespace>

# Create a deployment
kubectl create deployment <name> --image=<image> -n <namespace>

# Scale a deployment
kubectl scale deployment/<name> --replicas=3 -n <namespace>

# Update an image
kubectl set image deployment/<name> <container>=<new-image> -n <namespace>

# Watch rollout status
kubectl rollout status deployment/<name> -n <namespace>

# Roll back
kubectl rollout undo deployment/<name> -n <namespace>

# View rollout history
kubectl rollout history deployment/<name> -n <namespace>
```

### Pods
```bash
# List pods
kubectl get pods -n <namespace>
kubectl get pods -A -o wide

# Describe a pod
kubectl describe pod <pod-name> -n <namespace>

# Show pod YAML
kubectl get pod <pod-name> -n <namespace> -o yaml

# Delete a pod
kubectl delete pod <pod-name> -n <namespace>

# Force delete a stuck pod
kubectl delete pod <pod-name> -n <namespace> --grace-period=0 --force
```

### Exec and File Copy
```bash
# Open a shell in a container
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh

# Run a command in a container
kubectl exec <pod-name> -n <namespace> -- <command>

# Copy files to a pod
kubectl cp <local-path> <namespace>/<pod-name>:<pod-path>

# Copy files from a pod
kubectl cp <namespace>/<pod-name>:<pod-path> <local-path>
```

### StatefulSets
```bash
# List StatefulSets
kubectl get statefulsets -n <namespace>

# Scale a StatefulSet
kubectl scale statefulset/<name> --replicas=3 -n <namespace>

# Edit a StatefulSet
kubectl edit statefulset/<name> -n <namespace>

# Delete while preserving pods
kubectl delete statefulset/<name> --cascade=orphan -n <namespace>
```

### DaemonSets
```bash
# List DaemonSets
kubectl get daemonsets -n <namespace>

# Show DaemonSet rollout status
kubectl rollout status daemonset/<name> -n <namespace>

# Update DaemonSet image
kubectl set image daemonset/<name> <container>=<image> -n <namespace>
```

### Jobs and CronJobs
```bash
# List Jobs and CronJobs
kubectl get jobs,cronjobs -n <namespace>

# Create a Job from a CronJob
kubectl create job <job-name> --from=cronjob/<cronjob-name> -n <namespace>

# Suspend a CronJob
kubectl patch cronjob/<name> -n <namespace> -p '{"spec":{"suspend":true}}'

# Resume a CronJob
kubectl patch cronjob/<name> -n <namespace> -p '{"spec":{"suspend":false}}'

# View Job logs
kubectl logs job/<job-name> -n <namespace>
```

### Horizontal Pod Autoscaling
```bash
# List HPAs
kubectl get hpa -n <namespace>

# Create an HPA
kubectl autoscale deployment <name> --cpu-percent=70 --min=2 --max=10 -n <namespace>

# Describe an HPA
kubectl describe hpa <name> -n <namespace>
```

---

## Service & Ingress

### Services
```bash
# List services
kubectl get svc -n <namespace>
kubectl get svc -A

# Expose a deployment internally
kubectl expose deployment <name> --port=80 --target-port=8080 -n <namespace>

# Expose as NodePort
kubectl expose deployment <name> --type=NodePort --port=80 -n <namespace>

# Expose as LoadBalancer
kubectl expose deployment <name> --type=LoadBalancer --port=80 -n <namespace>

# Describe a service
kubectl describe svc <service-name> -n <namespace>

# View endpoints
kubectl get endpoints <service-name> -n <namespace>
```

### Ingress
```bash
# List ingresses
kubectl get ingress -n <namespace>
kubectl get ingress -A

# Describe an ingress
kubectl describe ingress <ingress-name> -n <namespace>

# Apply an ingress manifest
kubectl apply -f ingress.yaml

# Delete an ingress
kubectl delete ingress <ingress-name> -n <namespace>
```

### Port Forwarding
```bash
# Forward local port to a pod
kubectl port-forward pod/<pod-name> 8080:80 -n <namespace>

# Forward local port to a service
kubectl port-forward svc/<service-name> 8080:80 -n <namespace>
```

---

## Storage Management

### Persistent Volumes and Claims
```bash
# List PVs and PVCs
kubectl get pv
kubectl get pvc -A

# Describe a PV or PVC
kubectl describe pv <pv-name>
kubectl describe pvc <pvc-name> -n <namespace>

# Apply a PVC manifest
kubectl apply -f pvc.yaml

# Delete a PVC
kubectl delete pvc <pvc-name> -n <namespace>

# Resize a PVC if the storage class supports expansion
kubectl patch pvc <pvc-name> -n <namespace> \
  -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
```

### Storage Classes
```bash
# List storage classes
kubectl get storageclass
kubectl get sc

# Describe a storage class
kubectl describe storageclass <storage-class-name>

# Mark a storage class as default
kubectl patch storageclass <storage-class-name> \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### Volume Snapshots
```bash
# List snapshot resources
kubectl get volumesnapshot -n <namespace>
kubectl get volumesnapshotclass

# Create a snapshot
kubectl apply -f volumesnapshot.yaml

# Restore by applying a PVC that references a snapshot as dataSource
kubectl apply -f pvc-from-snapshot.yaml
```

---

## Observability

### Logs
```bash
# Pod logs
kubectl logs <pod-name> -n <namespace>
kubectl logs -f <pod-name> -n <namespace>
kubectl logs <pod-name> -c <container-name> -n <namespace>
kubectl logs <pod-name> --previous -n <namespace>

# Logs from all containers in a pod
kubectl logs <pod-name> --all-containers -n <namespace>

# Tail and time-based filters
kubectl logs <pod-name> --tail=100 -n <namespace>
kubectl logs <pod-name> --since=1h -n <namespace>
```

### Metrics
```bash
# Node and pod metrics (requires metrics-server)
kubectl top nodes
kubectl top pods -A
kubectl top pod <pod-name> --containers -n <namespace>

# Sort hot pods
kubectl top pods -A --sort-by=cpu
kubectl top pods -A --sort-by=memory
```

### Events
```bash
# List events in one namespace
kubectl get events -n <namespace>

# List events across the cluster, sorted by time
kubectl get events -A --sort-by='.metadata.creationTimestamp'

# Watch warning events only
kubectl get events -A --field-selector type=Warning --watch

# Show events for a specific object
kubectl get events -n <namespace> --field-selector involvedObject.name=<pod-name>
```

### Resource Usage and Status
```bash
# Show pod readiness
kubectl get pods -n <namespace> \
  -o custom-columns=NAME:.metadata.name,READY:.status.containerStatuses[*].ready,PHASE:.status.phase,RESTARTS:.status.containerStatuses[*].restartCount

# Watch live changes
kubectl get pods -n <namespace> -w
```

---

## RBAC & Security

### Service Accounts
```bash
# List service accounts
kubectl get serviceaccounts -n <namespace>

# Create a service account
kubectl create serviceaccount <name> -n <namespace>

# Create a token for a service account
kubectl create token <service-account> -n <namespace>

# Describe a service account
kubectl describe serviceaccount <name> -n <namespace>
```

### Roles and Bindings
```bash
# List roles and bindings
kubectl get roles,rolebindings -n <namespace>
kubectl get clusterroles,clusterrolebindings

# Create a namespace-scoped role
kubectl create role <role-name> \
  --verb=get,list,watch \
  --resource=pods \
  -n <namespace>

# Bind a role to a user
kubectl create rolebinding <binding-name> \
  --role=<role-name> \
  --user=<username> \
  -n <namespace>

# Bind a cluster role to a service account
kubectl create clusterrolebinding <binding-name> \
  --clusterrole=view \
  --serviceaccount=<namespace>:<service-account>
```

### Permission Checks
```bash
# Check your own permissions
kubectl auth can-i create deployments -n <namespace>

# Check as another user
kubectl auth can-i list secrets --as=<user> -n <namespace>
```

### Network Policies
```bash
# List network policies
kubectl get networkpolicies -n <namespace>

# Describe a policy
kubectl describe networkpolicy <policy-name> -n <namespace>

# Apply a policy manifest
kubectl apply -f networkpolicy.yaml
```

### Pod Security Admission
```bash
# Enforce a Pod Security level on a namespace
kubectl label namespace <namespace> \
  pod-security.kubernetes.io/enforce=restricted --overwrite

# Audit at a different level
kubectl label namespace <namespace> \
  pod-security.kubernetes.io/audit=baseline --overwrite

# Warn at a different level
kubectl label namespace <namespace> \
  pod-security.kubernetes.io/warn=baseline --overwrite
```

### Secrets and ConfigMaps
```bash
# List secrets and ConfigMaps
kubectl get secrets,configmaps -n <namespace>

# Create a generic secret
kubectl create secret generic <secret-name> \
  --from-literal=key=value \
  -n <namespace>

# Create a TLS secret
kubectl create secret tls <secret-name> \
  --cert=<path/to/cert> \
  --key=<path/to/key> \
  -n <namespace>

# Create a ConfigMap from a file
kubectl create configmap <configmap-name> \
  --from-file=<path/to/file> \
  -n <namespace>

# Read a secret value
kubectl get secret <secret-name> -n <namespace> \
  -o jsonpath='{.data.key}' | base64 -d
```

---

## Backup & Recovery

### Export Resource Definitions
```bash
# Export all common resources in a namespace
kubectl get all,configmap,secret,serviceaccount,role,rolebinding,pvc,ingress \
  -n <namespace> -o yaml > namespace-backup.yaml

# Export one resource
kubectl get deployment <name> -n <namespace> -o yaml > deployment.yaml
```

### Save Last-Applied Configuration
```bash
# Show the last applied config stored by kubectl
kubectl apply view-last-applied deployment/<name> -n <namespace>
```

### Restore Resources
```bash
# Re-apply exported manifests
kubectl apply -f namespace-backup.yaml

# Diff before applying
kubectl diff -f namespace-backup.yaml
```

### Notes
- `kubectl` is useful for exporting and restoring resource manifests.
- Full control plane and `etcd` backup procedures depend on how the cluster was installed and operated.
- For production disaster recovery, document platform-specific snapshot and restore steps separately.

---

## Troubleshooting

### Cluster Health
```bash
# General cluster status
kubectl cluster-info
kubectl get nodes
kubectl get pods -n kube-system

# API health endpoints
kubectl get --raw /livez
kubectl get --raw /readyz
```

### Node Issues
```bash
# Inspect a node
kubectl describe node <node-name>

# Cordon a node
kubectl cordon <node-name>

# Drain a node
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Return node to service
kubectl uncordon <node-name>
```

### Pod Troubleshooting
```bash
# Show status and scheduling details
kubectl get pod <pod-name> -n <namespace> -o wide
kubectl describe pod <pod-name> -n <namespace>

# Previous container logs after a crash
kubectl logs <pod-name> -c <container-name> --previous -n <namespace>

# Attach an ephemeral debug container
kubectl debug <pod-name> -n <namespace> --image=busybox --target=<container-name>
```

### Networking Checks
```bash
# Start a temporary debug pod
kubectl run net-debug --rm -it --restart=Never --image=nicolaka/netshoot -- /bin/bash

# Test cluster DNS
kubectl run dns-test --rm -it --restart=Never --image=busybox -- nslookup kubernetes.default

# Check Services and Endpoints
kubectl get svc,endpoints -n <namespace>

# Inspect Ingress objects
kubectl get ingress -A
kubectl describe ingress <ingress-name> -n <namespace>
```

### Storage Checks
```bash
# Inspect PVC binding state
kubectl get pvc -n <namespace>
kubectl describe pvc <pvc-name> -n <namespace>

# Inspect CSI resources
kubectl get csidrivers
kubectl get volumeattachments
```

### Performance and Pressure
```bash
# Check hot nodes and pods
kubectl top nodes
kubectl top pods -A --sort-by=memory
kubectl top pods -A --sort-by=cpu

# Review node conditions
kubectl describe nodes | grep -A 5 "Conditions"
```

### Useful Debug Output
```bash
# Full YAML for offline inspection
kubectl get pod <pod-name> -n <namespace> -o yaml

# JSONPath example
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.containerStatuses[*].state}'
```

---

## Quick Reference

### Common Short Names
```bash
kubectl get po      # pods
kubectl get svc     # services
kubectl get ns      # namespaces
kubectl get no      # nodes
kubectl get deploy  # deployments
kubectl get rs      # replicasets
kubectl get sts     # statefulsets
kubectl get ds      # daemonsets
kubectl get cm      # configmaps
kubectl get ing     # ingresses
kubectl get pv      # persistent volumes
kubectl get pvc     # persistent volume claims
kubectl get sc      # storage classes
kubectl get sa      # service accounts
```

### Output Formats
```bash
kubectl get pods -o wide
kubectl get pods -o yaml
kubectl get pods -o json
kubectl get pods --show-labels
kubectl get pods -l app=web
kubectl get pods --field-selector status.phase=Running
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
```

### Useful Aliases
```bash
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgn='kubectl get nodes'
alias kga='kubectl get all'
alias kdp='kubectl describe pod'
alias kl='kubectl logs'
alias kex='kubectl exec -it'
alias kaf='kubectl apply -f'
alias kdf='kubectl delete -f'
alias kctx='kubectl config use-context'
alias kns='kubectl config set-context --current --namespace'
```

---

## Best Practices

1. Use namespaces to separate teams, applications, and environments.
2. Label resources consistently so filtering and automation stay simple.
3. Set requests and limits for workloads.
4. Apply `readinessProbe`, `livenessProbe`, and `startupProbe` where appropriate.
5. Prefer declarative manifests with `kubectl apply` for repeatable changes.
6. Use RBAC with least privilege.
7. Enforce Pod Security Admission labels on namespaces.
8. Review events first when debugging scheduling or startup failures.
9. Keep backup and restore manifests tested and documented.
10. Watch rollout status during changes and use rollback when needed.
