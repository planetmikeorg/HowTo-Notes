# Kubernetes Managed by Rancher - Administrator Cheatsheet

## Table of Contents
- [Rancher CLI & Authentication](#rancher-cli--authentication)
- [Cluster Management](#cluster-management)
- [Project & Namespace Management](#project--namespace-management)
- [Workload Management](#workload-management)
- [Service & Ingress](#service--ingress)
- [Storage Management](#storage-management)
- [Monitoring & Logging](#monitoring--logging)
- [RBAC & Security](#rbac--security)
- [Backup & Disaster Recovery](#backup--disaster-recovery)
- [Troubleshooting](#troubleshooting)
- [Rancher-Specific Features](#rancher-specific-features)

---

## Rancher CLI & Authentication

### Install Rancher CLI
```bash
# Download and install Rancher CLI
wget https://github.com/rancher/cli/releases/download/v2.8.0/rancher-linux-amd64-v2.8.0.tar.gz
tar -xvf rancher-linux-amd64-v2.8.0.tar.gz
sudo mv rancher-v2.8.0/rancher /usr/local/bin/

# Verify installation
rancher --version
```

### Login to Rancher
```bash
# Login with API token
rancher login https://rancher.example.com --token <bearer-token>

# Login interactively
rancher login https://rancher.example.com

# List contexts
rancher context ls

# Switch context to specific cluster
rancher context switch <cluster-name>
```

### Generate API Token
```bash
# Via Rancher UI:
# User Icon → Account & API Keys → Create API Key

# View current token
rancher token
```

### Configure kubectl for Rancher Cluster
```bash
# Download kubeconfig from Rancher UI
# Cluster → Kubeconfig File → Download

# Or use Rancher CLI
rancher cluster kubeconfig <cluster-name> > ~/.kube/rancher-config

# Use the config
export KUBECONFIG=~/.kube/rancher-config

# Or merge with existing config
KUBECONFIG=~/.kube/config:~/.kube/rancher-config kubectl config view --flatten > ~/.kube/merged-config
mv ~/.kube/merged-config ~/.kube/config
```

---

## Cluster Management

### List Clusters
```bash
# Using Rancher CLI
rancher cluster ls

# Get cluster details
rancher cluster

# Using kubectl (on Rancher management cluster)
kubectl get clusters.management.cattle.io -n <cluster-id>
```

### Create New Cluster
```bash
# Create RKE2 cluster via Rancher CLI
rancher cluster create <cluster-name> --rke2

# Import existing cluster
rancher cluster import <cluster-name>

# Create from template
rancher cluster create <cluster-name> --cluster-template <template-name>
```

### Upgrade Cluster
```bash
# List available Kubernetes versions
rancher settings ls | grep kubernetes-version

# Upgrade cluster
rancher cluster upgrade <cluster-name> --kubernetes-version v1.28.5

# Check upgrade status
rancher cluster ls
kubectl get nodes -o wide
```

### Cluster Configuration
```bash
# Edit cluster configuration
rancher cluster update <cluster-name>

# Scale node pools
rancher node-pool scale <node-pool-name> --quantity 5

# Update cluster YAML
kubectl edit cluster.management.cattle.io <cluster-name> -n fleet-default
```

### Node Management
```bash
# List nodes in cluster
rancher nodes ls

# Cordon node (mark unschedulable)
kubectl cordon <node-name>

# Drain node (evict all pods)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Uncordon node
kubectl uncordon <node-name>

# Delete node from cluster
rancher nodes delete <node-name>

# Add node labels via Rancher
kubectl label nodes <node-name> <key>=<value>
```

---

## Project & Namespace Management

### Projects (Rancher-Specific)
```bash
# List projects
rancher projects ls

# Create project
rancher projects create <project-name>

# Switch to project
rancher context switch <project-name>

# Set resource quotas on project
rancher project set-quota <project-name> --limit-cpu 10 --limit-memory 20Gi

# Move namespace to project
kubectl label namespace <namespace> field.cattle.io/projectId=<project-id>
```

### Namespaces
```bash
# List namespaces
kubectl get namespaces
rancher namespaces ls

# Create namespace in project
rancher namespace create <namespace-name> --project <project-name>

# Create namespace with kubectl
kubectl create namespace <namespace-name>

# Add namespace to Rancher project
kubectl label namespace <namespace-name> field.cattle.io/projectId=<cluster-id>:<project-name>

# Delete namespace
kubectl delete namespace <namespace-name>
```

### Resource Quotas
```bash
# Create resource quota
kubectl create quota <quota-name> \
  --hard=cpu=10,memory=20Gi,pods=20 \
  -n <namespace>

# View quotas
kubectl get resourcequotas -n <namespace>
kubectl describe resourcequota <quota-name> -n <namespace>

# Project-level quotas (via Rancher UI or API)
rancher projects set-quota <project-name> \
  --limit-cpu 50 \
  --limit-memory 100Gi \
  --limit-pods 100
```

---

## Workload Management

### Deployments
```bash
# List deployments
kubectl get deployments -n <namespace>
rancher kubectl get deployments -n <namespace>

# Create deployment
kubectl create deployment <name> --image=<image> -n <namespace>

# Scale deployment
kubectl scale deployment <name> --replicas=5 -n <namespace>

# Update image
kubectl set image deployment/<name> <container>=<new-image> -n <namespace>

# Rollout status
kubectl rollout status deployment/<name> -n <namespace>

# Rollback deployment
kubectl rollout undo deployment/<name> -n <namespace>

# View rollout history
kubectl rollout history deployment/<name> -n <namespace>
```

### Pods
```bash
# List pods
kubectl get pods -n <namespace>
kubectl get pods --all-namespaces -o wide

# Get pod details
kubectl describe pod <pod-name> -n <namespace>

# View pod logs
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -c <container-name> -n <namespace>
kubectl logs -f <pod-name> -n <namespace>  # Follow logs

# Execute commands in pod
kubectl exec -it <pod-name> -n <namespace> -- /bin/bash
kubectl exec <pod-name> -n <namespace> -- <command>

# Copy files to/from pod
kubectl cp <local-path> <pod-name>:<pod-path> -n <namespace>
kubectl cp <pod-name>:<pod-path> <local-path> -n <namespace>

# Delete pod
kubectl delete pod <pod-name> -n <namespace>

# Force delete stuck pod
kubectl delete pod <pod-name> -n <namespace> --grace-period=0 --force
```

### StatefulSets
```bash
# List StatefulSets
kubectl get statefulsets -n <namespace>

# Scale StatefulSet
kubectl scale statefulset <name> --replicas=3 -n <namespace>

# Delete StatefulSet (keep pods)
kubectl delete statefulset <name> --cascade=orphan -n <namespace>

# Update StatefulSet
kubectl edit statefulset <name> -n <namespace>
```

### DaemonSets
```bash
# List DaemonSets
kubectl get daemonsets -n <namespace>

# Create DaemonSet
kubectl apply -f daemonset.yaml

# Update DaemonSet
kubectl set image daemonset/<name> <container>=<image> -n <namespace>

# View DaemonSet status
kubectl rollout status daemonset/<name> -n <namespace>
```

### Jobs & CronJobs
```bash
# List jobs
kubectl get jobs -n <namespace>

# Create job from CronJob
kubectl create job <job-name> --from=cronjob/<cronjob-name> -n <namespace>

# List CronJobs
kubectl get cronjobs -n <namespace>

# Suspend CronJob
kubectl patch cronjob/<name> -p '{"spec":{"suspend":true}}' -n <namespace>

# Resume CronJob
kubectl patch cronjob/<name> -p '{"spec":{"suspend":false}}' -n <namespace>

# View job logs
kubectl logs job/<job-name> -n <namespace>
```

---

## Service & Ingress

### Services
```bash
# List services
kubectl get services -n <namespace>
kubectl get svc --all-namespaces

# Create ClusterIP service
kubectl expose deployment <name> --port=80 --target-port=8080 -n <namespace>

# Create NodePort service
kubectl expose deployment <name> --type=NodePort --port=80 -n <namespace>

# Create LoadBalancer service
kubectl expose deployment <name> --type=LoadBalancer --port=80 -n <namespace>

# Get service details
kubectl describe service <service-name> -n <namespace>

# Get service endpoints
kubectl get endpoints <service-name> -n <namespace>

# Delete service
kubectl delete service <service-name> -n <namespace>
```

### Ingress
```bash
# List ingresses
kubectl get ingress -n <namespace>
kubectl get ingress --all-namespaces

# Describe ingress
kubectl describe ingress <ingress-name> -n <namespace>

# Create ingress
kubectl apply -f ingress.yaml

# Get ingress with backends
kubectl get ingress <ingress-name> -n <namespace> -o yaml

# Delete ingress
kubectl delete ingress <ingress-name> -n <namespace>
```

### Rancher Load Balancers
```bash
# Via Rancher UI:
# Cluster → Service Discovery → Load Balancers

# View via kubectl
kubectl get services -n <namespace> -l cattle.io/creator=norman

# Update load balancer annotations
kubectl annotate service <service-name> \
  external-dns.alpha.kubernetes.io/hostname=example.com \
  -n <namespace>
```

---

## Storage Management

### Persistent Volumes (PV)
```bash
# List persistent volumes
kubectl get pv

# Describe PV
kubectl describe pv <pv-name>

# Delete PV
kubectl delete pv <pv-name>

# Get PV with specific storage class
kubectl get pv -l storageclass=<storage-class-name>
```

### Persistent Volume Claims (PVC)
```bash
# List PVCs
kubectl get pvc -n <namespace>
kubectl get pvc --all-namespaces

# Describe PVC
kubectl describe pvc <pvc-name> -n <namespace>

# Create PVC
kubectl apply -f pvc.yaml

# Delete PVC
kubectl delete pvc <pvc-name> -n <namespace>

# Resize PVC (if storage class supports it)
kubectl patch pvc <pvc-name> -n <namespace> \
  -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
```

### Storage Classes
```bash
# List storage classes
kubectl get storageclass
kubectl get sc

# Describe storage class
kubectl describe storageclass <sc-name>

# Set default storage class
kubectl patch storageclass <sc-name> \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Remove default from storage class
kubectl patch storageclass <sc-name> \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

### Volume Snapshots
```bash
# List volume snapshots
kubectl get volumesnapshot -n <namespace>

# Create snapshot
kubectl apply -f volumesnapshot.yaml

# List snapshot classes
kubectl get volumesnapshotclass

# Restore from snapshot (create PVC from snapshot)
kubectl apply -f pvc-from-snapshot.yaml
```

---

## Monitoring & Logging

### Rancher Monitoring (Prometheus/Grafana)
```bash
# Install monitoring (via Rancher UI or CLI)
# Cluster → Apps → Charts → Monitoring

# Access Grafana
# Cluster → Monitoring → Grafana

# Access Prometheus
# Cluster → Monitoring → Prometheus

# Query Prometheus via kubectl port-forward
kubectl port-forward -n cattle-monitoring-system \
  svc/rancher-monitoring-prometheus 9090:9090

# Access AlertManager
kubectl port-forward -n cattle-monitoring-system \
  svc/rancher-monitoring-alertmanager 9093:9093
```

### View Logs
```bash
# Pod logs
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -c <container> -n <namespace>
kubectl logs -f <pod-name> -n <namespace>  # Stream logs
kubectl logs --since=1h <pod-name> -n <namespace>
kubectl logs --tail=100 <pod-name> -n <namespace>

# Previous container logs (if crashed)
kubectl logs <pod-name> --previous -n <namespace>

# All containers in pod
kubectl logs <pod-name> --all-containers -n <namespace>

# Rancher logging (if installed)
# Cluster → Logging → View Logs in Kibana/Grafana Loki
```

### Rancher Logging
```bash
# Install Rancher logging stack
# Cluster → Apps → Charts → Logging

# View cluster flows
kubectl get clusterflows.logging.banzaicloud.io

# View cluster outputs
kubectl get clusteroutputs.logging.banzaicloud.io

# View logging operator logs
kubectl logs -n cattle-logging-system -l app.kubernetes.io/name=logging-operator
```

### Metrics
```bash
# Node metrics
kubectl top nodes

# Pod metrics
kubectl top pods -n <namespace>
kubectl top pods --all-namespaces

# Container metrics
kubectl top pod <pod-name> --containers -n <namespace>

# Sort by CPU
kubectl top pods -n <namespace> --sort-by=cpu

# Sort by memory
kubectl top pods -n <namespace> --sort-by=memory
```

### Events
```bash
# View events
kubectl get events -n <namespace>
kubectl get events --all-namespaces --sort-by='.lastTimestamp'

# Watch events
kubectl get events -n <namespace> --watch

# Filter events by type
kubectl get events -n <namespace> --field-selector type=Warning

# Events for specific resource
kubectl get events -n <namespace> --field-selector involvedObject.name=<pod-name>
```

---

## RBAC & Security

### Service Accounts
```bash
# List service accounts
kubectl get serviceaccounts -n <namespace>

# Create service account
kubectl create serviceaccount <name> -n <namespace>

# Get service account token
kubectl create token <service-account> -n <namespace>

# Describe service account
kubectl describe serviceaccount <name> -n <namespace>

# Delete service account
kubectl delete serviceaccount <name> -n <namespace>
```

### Roles & RoleBindings
```bash
# List roles
kubectl get roles -n <namespace>

# List cluster roles
kubectl get clusterroles

# Create role
kubectl create role <role-name> \
  --verb=get,list,watch \
  --resource=pods \
  -n <namespace>

# Create role binding
kubectl create rolebinding <binding-name> \
  --role=<role-name> \
  --user=<username> \
  -n <namespace>

# Bind to service account
kubectl create rolebinding <binding-name> \
  --role=<role-name> \
  --serviceaccount=<namespace>:<service-account> \
  -n <namespace>

# List role bindings
kubectl get rolebindings -n <namespace>

# List cluster role bindings
kubectl get clusterrolebindings

# Check permissions
kubectl auth can-i <verb> <resource> -n <namespace>
kubectl auth can-i create pods -n <namespace>

# Check permissions for user
kubectl auth can-i list pods --as=<user> -n <namespace>
```

### Rancher User Management
```bash
# List users (via Rancher CLI)
rancher users ls

# Create user via API
# Use Rancher UI: Global → Users → Add User

# Assign cluster role to user
kubectl create clusterrolebinding <binding-name> \
  --clusterrole=<cluster-role> \
  --user=<rancher-user-id>

# Rancher project member roles
# Via Rancher UI: Cluster → Projects → <project> → Members → Add
```

### Network Policies
```bash
# List network policies
kubectl get networkpolicies -n <namespace>

# Describe network policy
kubectl describe networkpolicy <policy-name> -n <namespace>

# Create network policy
kubectl apply -f networkpolicy.yaml

# Delete network policy
kubectl delete networkpolicy <policy-name> -n <namespace>
```

### Pod Security
```bash
# Pod Security Standards (PSS)
kubectl label namespace <namespace> \
  pod-security.kubernetes.io/enforce=restricted

# Pod Security Admission levels: privileged, baseline, restricted

# View current PSS labels
kubectl get namespace <namespace> -o yaml | grep pod-security

# Rancher Pod Security Policies (deprecated in K8s 1.25+)
kubectl get podsecuritypolicies
```

### Secrets & ConfigMaps
```bash
# List secrets
kubectl get secrets -n <namespace>

# Create secret from literal
kubectl create secret generic <secret-name> \
  --from-literal=key=value \
  -n <namespace>

# Create secret from file
kubectl create secret generic <secret-name> \
  --from-file=<path/to/file> \
  -n <namespace>

# Create TLS secret
kubectl create secret tls <secret-name> \
  --cert=<path/to/cert> \
  --key=<path/to/key> \
  -n <namespace>

# View secret (base64 encoded)
kubectl get secret <secret-name> -n <namespace> -o yaml

# Decode secret
kubectl get secret <secret-name> -n <namespace> -o jsonpath='{.data.key}' | base64 -d

# ConfigMaps
kubectl get configmaps -n <namespace>
kubectl create configmap <name> --from-file=<path> -n <namespace>
kubectl describe configmap <name> -n <namespace>
```

---

## Backup & Disaster Recovery

### Rancher Backup Operator
```bash
# Install Rancher Backup operator
# Apps → Charts → Rancher Backup

# Create backup
kubectl apply -f - <<EOF
apiVersion: resources.cattle.io/v1
kind: Backup
metadata:
  name: rancher-backup
spec:
  storageLocation:
    s3:
      bucketName: my-backup-bucket
      endpoint: s3.amazonaws.com
      region: us-west-2
      credentialSecretName: s3-creds
      folder: rancher-backups
  retentionCount: 10
EOF

# List backups
kubectl get backups

# Create restore
kubectl apply -f - <<EOF
apiVersion: resources.cattle.io/v1
kind: Restore
metadata:
  name: restore-rancher
spec:
  backupFilename: rancher-backup-*.tar.gz
  storageLocation:
    s3:
      bucketName: my-backup-bucket
      endpoint: s3.amazonaws.com
      region: us-west-2
      credentialSecretName: s3-creds
      folder: rancher-backups
EOF

# Check restore status
kubectl get restores
kubectl describe restore restore-rancher
```

### etcd Snapshots (RKE/RKE2)
```bash
# Create on-demand etcd snapshot (via Rancher UI)
# Cluster → Snapshots → Take Snapshot

# Via kubectl
kubectl exec -n kube-system etcd-<node-name> -- \
  etcdctl snapshot save /tmp/snapshot.db

# List snapshots
# Cluster → Snapshots

# Restore from snapshot (via Rancher UI)
# Cluster → Snapshots → <snapshot> → Restore

# Configure automatic snapshots
# Cluster → Edit Config → Advanced Options → etcd
```

### Export Resources
```bash
# Export all resources in namespace
kubectl get all -n <namespace> -o yaml > namespace-backup.yaml

# Export specific resource types
kubectl get deployments,services,configmaps,secrets -n <namespace> -o yaml > resources.yaml

# Backup entire cluster (using velero)
velero backup create <backup-name> --include-namespaces <namespace>

# Restore from velero backup
velero restore create --from-backup <backup-name>
```

---

## Troubleshooting

### Cluster Health
```bash
# Check cluster status
rancher cluster ls
kubectl get nodes
kubectl cluster-info

# Check component status
kubectl get componentstatuses
kubectl get pods -n kube-system

# Check Rancher agent health
kubectl get pods -n cattle-system
kubectl logs -n cattle-system -l app=cattle-cluster-agent

# Rancher server logs
kubectl logs -n cattle-system -l app=rancher -f
```

### Node Issues
```bash
# Check node status
kubectl get nodes -o wide
kubectl describe node <node-name>

# Check node conditions
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'

# Check kubelet logs on node
ssh <node> journalctl -u kubelet -f

# Restart kubelet
ssh <node> systemctl restart kubelet

# Check Rancher agent on node
ssh <node> docker ps | grep rancher
ssh <node> docker logs <agent-container-id>
```

### Pod Troubleshooting
```bash
# Check pod status
kubectl get pod <pod-name> -n <namespace> -o wide

# Describe pod (shows events)
kubectl describe pod <pod-name> -n <namespace>

# Check pod logs
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -c <container> -n <namespace> --previous

# Debug with ephemeral container (K8s 1.23+)
kubectl debug <pod-name> -n <namespace> --image=busybox --target=<container>

# Check resource usage
kubectl top pod <pod-name> -n <namespace>

# Common issues:
# - ImagePullBackOff: Check image name, registry credentials
# - CrashLoopBackOff: Check logs, liveness probe
# - Pending: Check resources, node affinity, taints
# - OOMKilled: Increase memory limits
```

### Network Troubleshooting
```bash
# Test pod connectivity
kubectl run test-pod --image=nicolaka/netshoot --rm -it -- /bin/bash

# DNS resolution test
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default

# Test service connectivity
kubectl run -it --rm debug --image=busybox --restart=Never -- wget -O- <service-name>.<namespace>:80

# Check CNI plugin
kubectl get pods -n kube-system | grep -E 'calico|flannel|canal|weave|cilium'
kubectl logs -n kube-system <cni-pod>

# Check kube-proxy
kubectl get pods -n kube-system | grep kube-proxy
kubectl logs -n kube-system <kube-proxy-pod>

# Check network policies
kubectl get networkpolicies --all-namespaces
```

### Storage Troubleshooting
```bash
# Check PV/PVC status
kubectl get pv,pvc --all-namespaces

# Check storage class
kubectl get storageclass
kubectl describe storageclass <sc-name>

# Check CSI driver
kubectl get csidrivers
kubectl get csinodes

# Check volume attachments
kubectl get volumeattachments

# CSI driver logs
kubectl logs -n kube-system -l app=<csi-driver-name>
```

### Performance Issues
```bash
# Check resource usage
kubectl top nodes
kubectl top pods --all-namespaces

# Check node pressure
kubectl describe nodes | grep -A 5 "Conditions"

# Identify resource-heavy pods
kubectl top pods --all-namespaces --sort-by=memory
kubectl top pods --all-namespaces --sort-by=cpu

# Check API server performance
kubectl get --raw /metrics | grep apiserver_request_duration

# Check etcd performance
kubectl get --raw /metrics | grep etcd
```

### Rancher-Specific Troubleshooting
```bash
# Check Rancher server health
kubectl get pods -n cattle-system

# Rancher server logs
kubectl logs -n cattle-system deploy/rancher

# Check cluster agent
kubectl logs -n cattle-system -l app=cattle-cluster-agent -f

# Check fleet agent (GitOps)
kubectl logs -n cattle-fleet-system -l app=fleet-agent

# Reset cluster agent
kubectl delete pod -n cattle-system -l app=cattle-cluster-agent

# Re-register cluster with Rancher
# Cluster → Edit → Registration → Copy command
```

---

## Rancher-Specific Features

### Catalogs & Apps
```bash
# List available charts
rancher catalog list

# Install app from catalog
rancher app install <chart-name> <app-name> \
  --namespace <namespace> \
  --version <version>

# List installed apps
rancher apps ls
kubectl get apps -n cattle-system

# Upgrade app
rancher app upgrade <app-name> <version>

# Delete app
rancher app delete <app-name>

# Via kubectl (Helm releases)
kubectl get helmcharts -n <namespace>
kubectl get helmchartconfigs -n <namespace>
```

### Fleet (GitOps)
```bash
# View Fleet clusters
kubectl get clusters.fleet.cattle.io -n fleet-default

# View GitRepos
kubectl get gitrepos -n fleet-default

# Create GitRepo
kubectl apply -f - <<EOF
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: my-app
  namespace: fleet-default
spec:
  repo: https://github.com/org/repo
  branch: main
  paths:
  - /manifests
  targets:
  - clusterSelector:
      matchLabels:
        env: production
EOF

# View bundle deployments
kubectl get bundles -n fleet-default
kubectl get bundledeployments --all-namespaces

# Fleet agent logs
kubectl logs -n cattle-fleet-system -l app=fleet-agent
```

### Multi-Cluster Management
```bash
# List all managed clusters
rancher cluster ls

# Switch between clusters
rancher context switch <cluster-name>

# Execute command on multiple clusters
# Via Rancher UI: Global → kubectl Shell → Select Multiple Clusters

# Get resources across clusters (via Rancher API)
for cluster in $(rancher cluster ls --format '{{.Cluster.Name}}'); do
  echo "=== $cluster ==="
  rancher context switch $cluster
  kubectl get nodes
done
```

### Continuous Delivery
```bash
# View CD pipelines (via Rancher UI)
# Cluster → Tools → Continuous Delivery

# Create pipeline
# Via Rancher UI or kubectl apply pipeline YAML

# Trigger pipeline
kubectl apply -f pipeline-execution.yaml
```

### CIS Scans (Security Compliance)
```bash
# Install CIS Benchmark
# Cluster → Tools → CIS Benchmark

# Run CIS scan
kubectl apply -f - <<EOF
apiVersion: cis.cattle.io/v1
kind: ClusterScan
metadata:
  name: cis-scan
spec:
  scanProfileName: cis-1.6-profile
EOF

# View scan results
kubectl get clusterscans
kubectl describe clusterscan cis-scan

# View detailed report (via Rancher UI)
# Cluster → CIS Benchmark → View Report
```

### Istio Service Mesh
```bash
# Install Istio via Rancher
# Cluster → Apps → Charts → Istio

# Enable Istio for namespace
kubectl label namespace <namespace> istio-injection=enabled

# View Istio components
kubectl get pods -n istio-system

# View virtual services
kubectl get virtualservices --all-namespaces

# View destination rules
kubectl get destinationrules --all-namespaces

# Access Kiali (Istio dashboard)
kubectl port-forward -n istio-system svc/kiali 20001:20001
```

---

## Quick Reference

### Common Resource Shortcuts
```bash
kubectl get po        # pods
kubectl get svc       # services
kubectl get ns        # namespaces
kubectl get no        # nodes
kubectl get deploy    # deployments
kubectl get rs        # replicasets
kubectl get sts       # statefulsets
kubectl get ds        # daemonsets
kubectl get cm        # configmaps
kubectl get ing       # ingress
kubectl get pv        # persistent volumes
kubectl get pvc       # persistent volume claims
kubectl get sc        # storage classes
```

### Output Formats
```bash
kubectl get pods -o wide                    # Additional columns
kubectl get pods -o yaml                    # YAML output
kubectl get pods -o json                    # JSON output
kubectl get pods -o jsonpath='{.items[*].metadata.name}'  # JSONPath
kubectl get pods --show-labels              # Show labels
kubectl get pods -l app=web                 # Filter by label
kubectl get pods --field-selector status.phase=Running  # Field selector
```

### Useful Aliases
```bash
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgn='kubectl get nodes'
alias kdp='kubectl describe pod'
alias kl='kubectl logs'
alias kex='kubectl exec -it'
alias kaf='kubectl apply -f'
alias kdf='kubectl delete -f'
alias kgns='kubectl get namespaces'
alias kctx='kubectl config use-context'
alias rlogin='rancher login'
alias rcluster='rancher cluster ls'
```

### Rancher URLs
```bash
# Rancher Server API
https://<rancher-url>/v3

# Cluster API  
https://<rancher-url>/k8s/clusters/<cluster-id>

# Prometheus
https://<rancher-url>/k8s/clusters/<cluster-id>/api/v1/namespaces/cattle-monitoring-system/services/http:rancher-monitoring-prometheus:9090/proxy

# Grafana
https://<rancher-url>/k8s/clusters/<cluster-id>/api/v1/namespaces/cattle-monitoring-system/services/http:rancher-monitoring-grafana:80/proxy

# AlertManager
https://<rancher-url>/k8s/clusters/<cluster-id>/api/v1/namespaces/cattle-monitoring-system/services/http:rancher-monitoring-alertmanager:9093/proxy
```

---

## Best Practices

1. **Always use namespaces** to organize workloads and apply resource quotas
2. **Use Rancher projects** for multi-tenant environments with RBAC
3. **Enable monitoring** (Prometheus/Grafana) for observability
4. **Configure backup** strategy using Rancher Backup operator
5. **Use Fleet** for GitOps-based deployments across multiple clusters
6. **Implement network policies** for micro-segmentation
7. **Use resource requests/limits** to prevent resource contention
8. **Enable logging** to centralized location (ELK, Loki, Splunk)
9. **Regular etcd snapshots** for disaster recovery
10. **Use secrets management** (Sealed Secrets, External Secrets Operator)
11. **Implement Pod Security Standards** for workload security
12. **Use HPA/VPA** for automatic scaling
13. **Label everything** for better organization and filtering
14. **Document custom configurations** and maintain runbooks