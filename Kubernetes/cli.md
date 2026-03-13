# Kubernetes CLI Guide

A comprehensive guide to using command-line tools for Kubernetes: kubectl, Helm, and Rancher CLI.

---

## Table of Contents

1. [kubectl](#kubectl)
   - [Installation and Configuration](#installation-and-configuration)
   - [Context and Cluster Management](#context-and-cluster-management)
   - [Resource Management](#resource-management)
   - [Working with Pods](#working-with-pods)
   - [Deployments and Scaling](#deployments-and-scaling)
   - [Services and Networking](#services-and-networking)
   - [ConfigMaps and Secrets](#configmaps-and-secrets)
   - [Operators and CRDs](#operators-and-crds)
   - [Troubleshooting and Debugging](#troubleshooting-and-debugging)
   - [Advanced kubectl](#advanced-kubectl)

2. [Helm](#helm)
   - [Installation](#helm-installation)
   - [Repository Management](#repository-management)
   - [Chart Management](#chart-management)
   - [Release Management](#release-management)
   - [Chart Development](#chart-development)
   - [Helm Plugins](#helm-plugins)

3. [Rancher CLI](#rancher-cli)
   - [Installation and Setup](#rancher-installation-and-setup)
   - [Cluster Management](#rancher-cluster-management)
   - [Project and Namespace Operations](#project-and-namespace-operations)
   - [Application Management](#rancher-application-management)

---

# kubectl

kubectl is the official Kubernetes command-line tool for interacting with Kubernetes clusters.

## Installation and Configuration

### Installing kubectl

**Linux:**
```bash
# Download latest stable release
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Make executable and move to PATH
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Verify installation
kubectl version --client
```

**macOS:**
```bash
# Using Homebrew
brew install kubectl

# Or using curl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Verify
kubectl version --client
```

**Windows (PowerShell):**
```powershell
# Download
curl.exe -LO "https://dl.k8s.io/release/v1.28.0/bin/windows/amd64/kubectl.exe"

# Add to PATH and verify
kubectl version --client
```

### Initial Configuration

**Set up kubeconfig:**
```bash
# Default location: ~/.kube/config
# View current config
kubectl config view

# Set KUBECONFIG environment variable (if using custom location)
export KUBECONFIG=/path/to/kubeconfig

# Multiple configs
export KUBECONFIG=~/.kube/config:~/.kube/config-dev:~/.kube/config-prod
```

**Enable shell completion:**
```bash
# Bash
echo 'source <(kubectl completion bash)' >>~/.bashrc
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc

# Zsh
echo 'source <(kubectl completion zsh)' >>~/.zshrc
echo 'alias k=kubectl' >>~/.zshrc
echo 'complete -F __start_kubectl k' >>~/.zshrc

# Reload shell
source ~/.bashrc  # or source ~/.zshrc
```

## Context and Cluster Management

Contexts define which cluster, user, and namespace kubectl uses.

### Viewing and Switching Contexts

```bash
# List all contexts
kubectl config get-contexts

# View current context
kubectl config current-context

# Switch to a different context
kubectl config use-context production-cluster

# Get cluster information
kubectl cluster-info

# View cluster details
kubectl cluster-info dump
```

### Managing Contexts

```bash
# Create a new context
kubectl config set-context dev-context \
  --cluster=dev-cluster \
  --user=dev-user \
  --namespace=development

# Set default namespace for current context
kubectl config set-context --current --namespace=production

# Rename context
kubectl config rename-context old-name new-name

# Delete context
kubectl config delete-context old-context

# Get specific context details
kubectl config get-contexts production-cluster
```

### Managing Clusters and Users

```bash
# Add a cluster
kubectl config set-cluster my-cluster \
  --server=https://k8s.example.com:6443 \
  --certificate-authority=/path/to/ca.crt \
  --embed-certs=true

# Add user credentials
kubectl config set-credentials my-user \
  --client-certificate=/path/to/client.crt \
  --client-key=/path/to/client.key \
  --embed-certs=true

# Using token authentication
kubectl config set-credentials my-user --token=bearer_token_here

# View users
kubectl config view --minify --output jsonpath='{.users[*].name}'
```

## Resource Management

### Listing Resources

```bash
# List all resource types
kubectl api-resources

# Get pods in current namespace
kubectl get pods

# Get pods in specific namespace
kubectl get pods -n kube-system

# Get pods in all namespaces
kubectl get pods --all-namespaces
# or
kubectl get pods -A

# Get multiple resource types
kubectl get pods,services,deployments

# Get all resources in namespace
kubectl get all

# Wide output (more columns)
kubectl get pods -o wide

# Custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName

# Sort by field
kubectl get pods --sort-by=.metadata.creationTimestamp

# Get resources with labels
kubectl get pods --show-labels
kubectl get pods -l app=nginx
kubectl get pods -l 'environment in (production, staging)'
kubectl get pods -l 'app,!legacy'
```

### Describing Resources

```bash
# Describe pod (shows events, conditions, and detailed info)
kubectl describe pod nginx-pod

# Describe with specific namespace
kubectl describe pod nginx-pod -n production

# Describe multiple resources
kubectl describe pods -l app=nginx

# Describe deployment
kubectl describe deployment web-app

# Describe node
kubectl describe node worker-node-01

# Describe service
kubectl describe service frontend
```

### Creating Resources

```bash
# Create from file
kubectl create -f deployment.yaml

# Create from multiple files
kubectl create -f ./configs/

# Create from URL
kubectl create -f https://example.com/deployment.yaml

# Create from stdin
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
EOF

# Create namespace
kubectl create namespace development

# Create deployment imperatively
kubectl create deployment nginx --image=nginx:latest --replicas=3

# Create service
kubectl create service clusterip my-service --tcp=80:8080

# Create secret from literal
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=secret123

# Create secret from file
kubectl create secret generic ssl-cert \
  --from-file=tls.crt=./cert.crt \
  --from-file=tls.key=./cert.key

# Create configmap from literal
kubectl create configmap app-config \
  --from-literal=database_host=postgres.example.com \
  --from-literal=database_port=5432

# Create configmap from file
kubectl create configmap nginx-config --from-file=nginx.conf
```

### Applying Resources (Declarative)

```bash
# Apply configuration (creates or updates)
kubectl apply -f deployment.yaml

# Apply all files in directory
kubectl apply -f ./configs/

# Apply recursively
kubectl apply -f ./configs/ --recursive

# Apply with dry-run (see what would change)
kubectl apply -f deployment.yaml --dry-run=client -o yaml

# Server-side dry run
kubectl apply -f deployment.yaml --dry-run=server

# Show differences before applying
kubectl diff -f deployment.yaml

# Apply and record the command in annotation
kubectl apply -f deployment.yaml --record

# Prune resources (delete resources not in config)
kubectl apply -f configs/ --prune -l app=myapp
```

### Editing Resources

```bash
# Edit resource in default editor
kubectl edit deployment nginx

# Edit with specific editor
KUBE_EDITOR="nano" kubectl edit service frontend

# Edit multiple resources
kubectl edit deployment/nginx deployment/apache

# Patch resource (JSON)
kubectl patch deployment nginx -p '{"spec":{"replicas":5}}'

# Patch resource (strategic merge)
kubectl patch deployment nginx --patch '
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
'

# Patch using file
kubectl patch deployment nginx --patch-file patch.yaml

# Set image (shorthand for patching image)
kubectl set image deployment/nginx nginx=nginx:1.21

# Set resources
kubectl set resources deployment nginx --limits=cpu=200m,memory=512Mi --requests=cpu=100m,memory=256Mi

# Set environment variable
kubectl set env deployment/nginx DATABASE_URL=postgres://db.example.com

# Set service account
kubectl set serviceaccount deployment nginx sa-nginx
```

### Deleting Resources

```bash
# Delete resource by name
kubectl delete pod nginx-pod

# Delete from file
kubectl delete -f deployment.yaml

# Delete multiple resources
kubectl delete pod nginx-pod1 nginx-pod2

# Delete all pods with label
kubectl delete pods -l app=nginx

# Delete all pods in namespace
kubectl delete pods --all -n development

# Delete namespace (deletes all resources in it)
kubectl delete namespace development

# Force delete (skip graceful shutdown)
kubectl delete pod nginx-pod --grace-period=0 --force

# Delete with wait
kubectl delete pod nginx-pod --wait=true

# Delete and wait for completion
kubectl wait --for=delete pod/nginx-pod --timeout=60s
```

### Resource Output Formats

```bash
# YAML output
kubectl get pod nginx -o yaml

# JSON output
kubectl get pod nginx -o json

# JSONPath (extract specific fields)
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pods -o jsonpath='{.items[*].status.podIP}'
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'

# Wide output
kubectl get pods -o wide

# Name only
kubectl get pods -o name

# Custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,IP:.status.podIP

# Go template
kubectl get pods -o go-template='{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}'

# Go template file
kubectl get pods -o go-template-file=template.txt
```

## Working with Pods

### Basic Pod Operations

```bash
# Create pod from image
kubectl run nginx --image=nginx:latest

# Create pod with specific port
kubectl run nginx --image=nginx --port=80

# Create pod with environment variables
kubectl run myapp --image=myapp:v1 --env="ENV=production" --env="DEBUG=false"

# Create pod with resource limits
kubectl run nginx --image=nginx --limits=cpu=200m,memory=512Mi --requests=cpu=100m,memory=256Mi

# Create pod with labels
kubectl run nginx --image=nginx --labels=app=web,tier=frontend

# Create pod with command override
kubectl run busybox --image=busybox --command -- sh -c "echo Hello World"

# Create pod in specific namespace
kubectl run nginx --image=nginx -n production

# Create pod with restart policy
kubectl run test-pod --image=busybox --restart=Never -- echo "test"

# Dry run (generate YAML without creating)
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
```

### Pod Information

```bash
# Get pod status
kubectl get pod nginx

# Get pod details
kubectl describe pod nginx

# Get pod logs
kubectl logs nginx

# Follow logs (tail -f)
kubectl logs -f nginx

# Get logs from previous container instance
kubectl logs nginx --previous

# Get logs with timestamps
kubectl logs nginx --timestamps

# Get logs since specific time
kubectl logs nginx --since=1h
kubectl logs nginx --since-time=2026-03-03T10:00:00Z

# Get last N lines
kubectl logs nginx --tail=50

# Get logs from specific container in pod
kubectl logs nginx -c sidecar-container

# Get logs from all containers in pod
kubectl logs nginx --all-containers

# Stream logs from multiple pods
kubectl logs -l app=nginx --all-containers=true -f
```

### Executing Commands in Pods

```bash
# Execute command in pod
kubectl exec nginx -- ls -la /

# Interactive shell
kubectl exec -it nginx -- /bin/bash
kubectl exec -it nginx -- /bin/sh

# Execute in specific container
kubectl exec -it nginx -c sidecar -- /bin/bash

# Execute with namespace
kubectl exec -it nginx -n production -- /bin/bash

# Run command without TTY
kubectl exec nginx -- curl http://localhost

# Execute complex commands
kubectl exec nginx -- sh -c "ps aux | grep nginx"

# Copy files to pod
kubectl cp /local/file.txt nginx:/tmp/file.txt

# Copy files from pod
kubectl cp nginx:/var/log/app.log /local/app.log

# Copy from specific container
kubectl cp nginx:/config/app.conf /local/app.conf -c sidecar

# Copy directory
kubectl cp /local/config/ nginx:/app/config/
```

### Pod Port Forwarding

```bash
# Forward local port to pod port
kubectl port-forward pod/nginx 8080:80

# Forward to specific address
kubectl port-forward pod/nginx 8080:80 --address='0.0.0.0'

# Forward multiple ports
kubectl port-forward pod/nginx 8080:80 8443:443

# Forward to service
kubectl port-forward service/nginx 8080:80

# Forward to deployment
kubectl port-forward deployment/nginx 8080:80

# Forward in background
kubectl port-forward pod/nginx 8080:80 &
```

### Pod Debugging

```bash
# Get pod events
kubectl get events --field-selector involvedObject.name=nginx

# Get pod resource usage
kubectl top pod nginx

# Get all pod resource usage
kubectl top pods -A

# Debug pod (creates ephemeral debug container)
kubectl debug nginx -it --image=busybox

# Debug with specific image
kubectl debug nginx -it --image=nicolaka/netshoot

# Debug node by creating pod on node
kubectl debug node/worker-01 -it --image=busybox

# Create copy of pod for debugging
kubectl debug nginx --copy-to=nginx-debug

# Attach to running pod
kubectl attach nginx -it

# Check pod readiness/liveness
kubectl get pod nginx -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'
```

## Deployments and Scaling

### Creating Deployments

```bash
# Create deployment
kubectl create deployment nginx --image=nginx:latest

# Create deployment with replicas
kubectl create deployment nginx --image=nginx:latest --replicas=3

# Create deployment with port
kubectl create deployment nginx --image=nginx:latest --port=80

# Create from YAML
kubectl apply -f deployment.yaml

# Example deployment YAML
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
EOF
```

### Managing Deployments

```bash
# Get deployments
kubectl get deployments

# Get deployment status
kubectl rollout status deployment/nginx

# Get deployment history
kubectl rollout history deployment/nginx

# Get specific revision details
kubectl rollout history deployment/nginx --revision=2

# Scale deployment
kubectl scale deployment nginx --replicas=5

# Autoscale deployment
kubectl autoscale deployment nginx --min=2 --max=10 --cpu-percent=80

# Update deployment image
kubectl set image deployment/nginx nginx=nginx:1.22

# Edit deployment
kubectl edit deployment nginx

# Pause deployment (stops rollout)
kubectl rollout pause deployment/nginx

# Resume deployment
kubectl rollout resume deployment/nginx

# Restart deployment (rolling restart)
kubectl rollout restart deployment/nginx

# Undo rollout (rollback to previous revision)
kubectl rollout undo deployment/nginx

# Rollback to specific revision
kubectl rollout undo deployment/nginx --to-revision=2
```

### Deployment Strategies

```bash
# Rolling update (default)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 3
      maxUnavailable: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
EOF

# Recreate strategy (downtime)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
EOF
```

### StatefulSets

```bash
# Create StatefulSet
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
EOF

# Get StatefulSets
kubectl get statefulsets

# Scale StatefulSet
kubectl scale statefulset mysql --replicas=5

# Delete StatefulSet (keep pods)
kubectl delete statefulset mysql --cascade=orphan

# Update StatefulSet
kubectl set image statefulset/mysql mysql=mysql:8.1
```

### DaemonSets

```bash
# Create DaemonSet
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluentd:latest
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
EOF

# Get DaemonSets
kubectl get daemonsets

# Update DaemonSet image
kubectl set image daemonset/fluentd fluentd=fluentd:v1.15
```

### Jobs and CronJobs

```bash
# Create Job
kubectl create job hello --image=busybox -- echo "Hello World"

# Create Job from file
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
  completions: 3
  parallelism: 2
EOF

# Get Jobs
kubectl get jobs

# Get Job logs
kubectl logs job/pi

# Delete completed jobs
kubectl delete job --field-selector status.successful=1

# Create CronJob
kubectl create cronjob hello --image=busybox --schedule="*/5 * * * *" -- echo "Hello World"

# Create CronJob from file
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup-tool:latest
            command: ["/usr/local/bin/backup.sh"]
          restartPolicy: OnFailure
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
EOF

# Get CronJobs
kubectl get cronjobs

# Suspend CronJob
kubectl patch cronjob backup -p '{"spec":{"suspend":true}}'

# Create Job from CronJob (manual trigger)
kubectl create job --from=cronjob/backup manual-backup-001
```

## Services and Networking

### Creating Services

```bash
# Create ClusterIP service (default)
kubectl create service clusterip nginx --tcp=80:80

# Expose deployment as service
kubectl expose deployment nginx --port=80 --target-port=8080

# Create NodePort service
kubectl create service nodeport nginx --tcp=80:80 --node-port=30080

# Create LoadBalancer service
kubectl create service loadbalancer nginx --tcp=80:80

# Create service from YAML
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
    name: http
EOF

# Headless service (for StatefulSets)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
  - port: 3306
    name: mysql
EOF

# ExternalName service
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.example.com
EOF
```

### Managing Services

```bash
# Get services
kubectl get services
kubectl get svc

# Describe service
kubectl describe service nginx

# Get service endpoints
kubectl get endpoints nginx

# Get service details
kubectl get service nginx -o wide

# Edit service
kubectl edit service nginx

# Delete service
kubectl delete service nginx

# Test service connectivity
kubectl run tmp-shell --rm -i --tty --image nicolaka/netshoot -- /bin/bash
# Inside pod:
# curl http://nginx.default.svc.cluster.local
```

### Ingress

```bash
# Create Ingress
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
EOF

# Ingress with TLS
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
EOF

# Multiple paths
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-path
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
EOF

# Get Ingress
kubectl get ingress

# Describe Ingress
kubectl describe ingress app-ingress

# Get Ingress with addresses
kubectl get ingress -o wide
```

### NetworkPolicies

```bash
# Deny all ingress traffic
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF

# Allow specific pods
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
EOF

# Allow from namespace
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-monitoring
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
EOF

# Egress policy
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
EOF

# Get NetworkPolicies
kubectl get networkpolicies
kubectl get netpol
```

## ConfigMaps and Secrets

### ConfigMaps

```bash
# Create ConfigMap from literal values
kubectl create configmap app-config \
  --from-literal=database_host=postgres.example.com \
  --from-literal=database_port=5432 \
  --from-literal=log_level=info

# Create ConfigMap from file
kubectl create configmap nginx-config --from-file=nginx.conf

# Create ConfigMap from directory
kubectl create configmap app-configs --from-file=./config/

# Create ConfigMap from env file
echo "DATABASE_HOST=postgres.example.com" > config.env
echo "DATABASE_PORT=5432" >> config.env
kubectl create configmap env-config --from-env-file=config.env

# Create ConfigMap from YAML
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_host: "postgres.example.com"
  database_port: "5432"
  app.properties: |
    server.port=8080
    server.host=0.0.0.0
    logging.level=INFO
EOF

# Get ConfigMaps
kubectl get configmaps
kubectl get cm

# View ConfigMap data
kubectl get configmap app-config -o yaml

# Describe ConfigMap
kubectl describe configmap app-config

# Edit ConfigMap
kubectl edit configmap app-config

# Delete ConfigMap
kubectl delete configmap app-config
```

### Using ConfigMaps in Pods

```bash
# Mount as volume
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: config
      mountPath: /etc/config
  volumes:
  - name: config
    configMap:
      name: app-config
EOF

# Use as environment variables
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    envFrom:
    - configMapRef:
        name: app-config
EOF

# Use specific keys as env vars
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: DATABASE_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database_host
    - name: DATABASE_PORT
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database_port
EOF
```

### Secrets

```bash
# Create generic secret from literals
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=secretpass123

# Create secret from files
kubectl create secret generic tls-secret \
  --from-file=tls.crt=./cert.crt \
  --from-file=tls.key=./cert.key

# Create TLS secret
kubectl create secret tls tls-secret \
  --cert=./cert.crt \
  --key=./cert.key

# Create Docker registry secret
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass \
  --docker-email=user@example.com

# Create secret from YAML (base64 encoded)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: YWRtaW4=
  password: c2VjcmV0cGFzczEyMw==
EOF

# Create secret from YAML (plain text)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  username: admin
  password: secretpass123
EOF

# Get secrets
kubectl get secrets

# Describe secret (doesn't show values)
kubectl describe secret db-secret

# View secret data (base64 encoded)
kubectl get secret db-secret -o yaml

# Decode secret value
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 --decode

# Edit secret
kubectl edit secret db-secret

# Delete secret
kubectl delete secret db-secret
```

### Using Secrets in Pods

```bash
# Mount as volume
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: secrets
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secrets
    secret:
      secretName: db-secret
EOF

# Use as environment variables
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
EOF

# Use Docker registry secret for pulling images
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: private-app
spec:
  containers:
  - name: app
    image: registry.example.com/myapp:latest
  imagePullSecrets:
  - name: regcred
EOF
```

## Operators and CRDs

Operators extend Kubernetes functionality using Custom Resource Definitions (CRDs).

### Understanding CRDs

```bash
# List all CRDs
kubectl get crds

# Get CRD details
kubectl get crd <crd-name> -o yaml

# Describe CRD
kubectl describe crd <crd-name>

# List API resources (includes CRDs)
kubectl api-resources

# Get API versions
kubectl api-versions
```

### Creating Custom Resources

```bash
# Example: Creating a CRD
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com
spec:
  group: example.com
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              size:
                type: string
              replicas:
                type: integer
                minimum: 1
                maximum: 10
  scope: Namespaced
  names:
    plural: databases
    singular: database
    kind: Database
    shortNames:
    - db
EOF

# Create a custom resource instance
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: Database
metadata:
  name: my-database
spec:
  size: "10Gi"
  replicas: 3
EOF

# Get custom resources
kubectl get databases
kubectl get db

# Describe custom resource
kubectl describe database my-database

# Delete custom resource
kubectl delete database my-database
```

### Common Operator Examples

#### Prometheus Operator

```bash
# Install Prometheus Operator (using manifests)
kubectl apply -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/bundle.yaml

# Create ServiceMonitor (custom resource)
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: example-app
  labels:
    team: frontend
spec:
  selector:
    matchLabels:
      app: example-app
  endpoints:
  - port: web
    interval: 30s
EOF

# Create Prometheus instance
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
spec:
  serviceAccountName: prometheus
  serviceMonitorSelector:
    matchLabels:
      team: frontend
  resources:
    requests:
      memory: 400Mi
  replicas: 2
EOF

# Get Prometheus custom resources
kubectl get prometheus
kubectl get servicemonitor
kubectl get prometheusrule

# Create PrometheusRule (alerting rules)
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: example-alerts
spec:
  groups:
  - name: example
    interval: 30s
    rules:
    - alert: HighErrorRate
      expr: rate(http_requests_total{status="500"}[5m]) > 0.05
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "High error rate detected"
EOF
```

#### Cert-Manager Operator

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Create ClusterIssuer (Let's Encrypt)
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF

# Create Certificate resource
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com
spec:
  secretName: example-com-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - example.com
  - www.example.com
EOF

# Get cert-manager resources
kubectl get clusterissuer
kubectl get certificate
kubectl get certificaterequest
kubectl get order
kubectl get challenge

# Check certificate status
kubectl describe certificate example-com

# View certificate details
kubectl get secret example-com-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout
```

#### Istio Operator

```bash
# Install Istio operator
kubectl apply -f https://istio.io/latest/manifests/operator.yaml

# Create IstioOperator resource
cat <<EOF | kubectl apply -f -
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio-control-plane
  namespace: istio-system
spec:
  profile: default
  components:
    pilot:
      k8s:
        resources:
          requests:
            cpu: 100m
            memory: 512Mi
    ingressGateways:
    - name: istio-ingressgateway
      enabled: true
      k8s:
        service:
          type: LoadBalancer
EOF

# Create VirtualService
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp.example.com
  gateways:
  - myapp-gateway
  http:
  - match:
    - uri:
        prefix: "/api"
    route:
    - destination:
        host: api-service
        port:
          number: 8080
  - route:
    - destination:
        host: web-service
        port:
          number: 80
EOF

# Create DestinationRule
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp-service
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
EOF

# Get Istio resources
kubectl get virtualservice
kubectl get destinationrule
kubectl get gateway
kubectl get serviceentry

# Analyze Istio configuration
istioctl analyze
```

#### Postgres Operator (Zalando)

```bash
# Install Postgres Operator
kubectl apply -k github.com/zalando/postgres-operator/manifests

# Create PostgreSQL cluster
cat <<EOF | kubectl apply -f -
apiVersion: "acid.zalan.do/v1"
kind: postgresql
metadata:
  name: demo-cluster
spec:
  teamId: "demo"
  volume:
    size: 10Gi
  numberOfInstances: 3
  users:
    app_user: []
  databases:
    app_db: app_user
  postgresql:
    version: "15"
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi
EOF

# Get PostgreSQL clusters
kubectl get postgresql

# Get cluster pods
kubectl get pods -l cluster-name=demo-cluster

# Get database credentials
kubectl get secret app-user.demo-cluster.credentials.postgresql.acid.zalan.do -o yaml
```

### Working with Operators

```bash
# Install Operator Lifecycle Manager (OLM)
curl -L https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.26.0/install.sh -o install.sh
chmod +x install.sh
./install.sh v0.26.0

# List available operators
kubectl get packagemanifests -n olm

# Install operator via OLM
cat <<EOF | kubectl apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: my-operator
  namespace: operators
spec:
  channel: stable
  name: my-operator
  source: operatorhubio-catalog
  sourceNamespace: olm
EOF

# Get installed operators
kubectl get csv -A

# Get operator subscriptions
kubectl get subscriptions -A

# Check operator logs
kubectl logs -n operators deployment/my-operator-controller-manager
```

## Troubleshooting and Debugging

### Cluster Diagnostics

```bash
# Check cluster health
kubectl get componentstatuses
kubectl get cs

# Check node status
kubectl get nodes
kubectl describe node <node-name>

# Check node resource usage
kubectl top nodes

# Get cluster events
kubectl get events --all-namespaces --sort-by='.lastTimestamp'

# Get events for specific resource
kubectl get events --field-selector involvedObject.name=nginx-pod

# Check API server logs (if access to control plane)
kubectl logs -n kube-system kube-apiserver-<node-name>

# Check controller manager logs
kubectl logs -n kube-system kube-controller-manager-<node-name>

# Check scheduler logs
kubectl logs -n kube-system kube-scheduler-<node-name>

# Check kubelet logs (on node)
journalctl -u kubelet -f

# Drain node (for maintenance)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Cordon node (prevent new pods)
kubectl cordon <node-name>

# Uncordon node (allow scheduling)
kubectl uncordon <node-name>

# Taint node
kubectl taint nodes <node-name> key=value:NoSchedule

# Remove taint
kubectl taint nodes <node-name> key:NoSchedule-
```

### Pod Troubleshooting

```bash
# Check pod status
kubectl get pod nginx -o wide

# Describe pod (shows events and conditions)
kubectl describe pod nginx

# Get pod logs
kubectl logs nginx
kubectl logs nginx --previous  # Previous container instance
kubectl logs nginx -c sidecar  # Specific container

# Stream logs
kubectl logs -f nginx --all-containers

# Check pod events
kubectl get events --field-selector involvedObject.name=nginx

# Check pod resource usage
kubectl top pod nginx

# Execute commands in pod
kubectl exec nginx -- ps aux
kubectl exec nginx -- netstat -tulpn
kubectl exec -it nginx -- /bin/bash

# Debug with ephemeral container
kubectl debug nginx -it --image=nicolaka/netshoot -- bash

# Copy logs from pod
kubectl cp nginx:/var/log/app.log ./app.log

# Check if pod can resolve DNS
kubectl exec nginx -- nslookup kubernetes.default

# Test connectivity
kubectl exec nginx -- curl http://service-name

# Check pod security context
kubectl get pod nginx -o jsonpath='{.spec.securityContext}'

# Check container security context
kubectl get pod nginx -o jsonpath='{.spec.containers[0].securityContext}'

# View pod definition as created
kubectl get pod nginx -o yaml

# Check pod scheduling
kubectl get pod nginx -o jsonpath='{.spec.nodeName}'
kubectl get pod nginx -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")]}'
```

### Network Troubleshooting

```bash
# Create debug pod
kubectl run tmp-shell --rm -i --tty --image nicolaka/netshoot -- /bin/bash

# Inside debug pod:
# Test DNS
nslookup kubernetes.default
nslookup myservice.mynamespace.svc.cluster.local

# Test service connectivity
curl http://myservice:80
curl http://myservice.mynamespace.svc.cluster.local:80

# Test pod IP directly
curl http://10.244.1.5:8080

# Check routes
ip route

# Check network interfaces
ip addr

# Trace route
traceroute myservice

# Test DNS from specific namespace
nslookup myservice

# Check NetworkPolicies
kubectl get networkpolicies

# Describe NetworkPolicy
kubectl describe networkpolicy allow-frontend

# Test egress connectivity
curl https://www.google.com

# Check service endpoints
kubectl get endpoints myservice

# Describe service
kubectl describe service myservice

# Port forward for testing
kubectl port-forward service/myservice 8080:80

# Check Ingress
kubectl get ingress
kubectl describe ingress my-ingress

# Check Ingress controller logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```

### Storage Troubleshooting

```bash
# Get PersistentVolumes
kubectl get pv

# Get PersistentVolumeClaims
kubectl get pvc

# Describe PVC
kubectl describe pvc my-pvc

# Check PVC events
kubectl get events --field-selector involvedObject.name=my-pvc

# Get StorageClasses
kubectl get storageclass
kubectl get sc

# Describe StorageClass
kubectl describe sc standard

# Check volume mount in pod
kubectl exec nginx -- df -h
kubectl exec nginx -- mount | grep /data

# Get PV associated with PVC
kubectl get pvc my-pvc -o jsonpath='{.spec.volumeName}'

# Check PV status
kubectl get pv <pv-name> -o jsonpath='{.status.phase}'

# Manually delete PV (if stuck)
kubectl patch pv <pv-name> -p '{"metadata":{"finalizers":null}}'
```

### Resource Issues

```bash
# Check resource quotas
kubectl get resourcequota
kubectl describe resourcequota my-quota

# Check LimitRanges
kubectl get limitrange
kubectl describe limitrange limits

# View pod resource requests and limits
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].resources}{"\n"}{end}'

# Top consumers by CPU
kubectl top pods --all-namespaces --sort-by=cpu

# Top consumers by memory
kubectl top pods --all-namespaces --sort-by=memory

# Check if pod is OOMKilled
kubectl get pod nginx -o jsonpath='{.status.containerStatuses[0].lastState}'

# Check pod evictions
kubectl get events --field-selector reason=Evicted

# Check node capacity
kubectl describe node <node-name> | grep -A 5 "Allocated resources"
```

## Advanced kubectl

### Labels and Selectors

```bash
# Add label
kubectl label pod nginx env=production

# Add multiple labels
kubectl label pod nginx tier=frontend app=web

# Update label (overwrite)
kubectl label pod nginx env=staging --overwrite

# Remove label
kubectl label pod nginx env-

# Get resources with label
kubectl get pods -l env=production
kubectl get pods -l env=production,tier=frontend

# Label selector with set operations
kubectl get pods -l 'env in (production, staging)'
kubectl get pods -l 'env notin (development)'
kubectl get pods -l 'tier,env'  # Has both labels
kubectl get pods -l '!legacy'   # Doesn't have legacy label

# Label all pods in namespace
kubectl label pods --all env=production

# Label nodes
kubectl label node worker-01 disktype=ssd
kubectl label node worker-01 gpu=true

# Show labels
kubectl get pods --show-labels
kubectl get nodes --show-labels
```

### Annotations

```bash
# Add annotation
kubectl annotate pod nginx description="Web server pod"

# Add multiple annotations
kubectl annotate pod nginx \
  owner="team-a" \
  created-by="kubectl"

# Update annotation
kubectl annotate pod nginx description="Updated web server" --overwrite

# Remove annotation
kubectl annotate pod nginx description-

# View annotations
kubectl get pod nginx -o jsonpath='{.metadata.annotations}'

# Annotate from file
kubectl annotate -f pod.yaml owner="team-a"
```

### Field Selectors

```bash
# Get running pods
kubectl get pods --field-selector status.phase=Running

# Get pods on specific node
kubectl get pods --field-selector spec.nodeName=worker-01

# Get failed pods
kubectl get pods --field-selector status.phase=Failed

# Multiple field selectors
kubectl get pods --field-selector status.phase=Running,spec.nodeName=worker-01

# Get services with specific type
kubectl get services --field-selector spec.type=LoadBalancer

# Get events for specific object
kubectl get events --field-selector involvedObject.name=nginx

# Combine label and field selectors
kubectl get pods -l app=nginx --field-selector status.phase=Running
```

### JSONPath and Output

```bash
# Get pod names
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Get pod IPs
kubectl get pods -o jsonpath='{.items[*].status.podIP}'

# Get pod names and IPs
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'

# Get node names and roles
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels.node-role\.kubernetes\.io/.*}{"\n"}{end}'

# Get all container images
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}' | tr ' ' '\n' | sort | uniq

# Get resource limits
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].resources.limits.memory}{"\n"}{end}'

# Get PVC and associated PV
kubectl get pvc -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.volumeName}{"\n"}{end}'

# Get service external IPs
kubectl get services -o jsonpath='{.items[?(@.spec.type=="LoadBalancer")].status.loadBalancer.ingress[*].ip}'

# Complex JSONPath with filtering
kubectl get pods -o jsonpath='{.items[?(@.status.phase=="Running")].metadata.name}'
```

### Resource Management with Kustomize

```bash
# Apply kustomization
kubectl apply -k ./overlays/production/

# View kustomization output without applying
kubectl kustomize ./overlays/production/

# Diff kustomization
kubectl diff -k ./overlays/production/

# Delete resources from kustomization
kubectl delete -k ./overlays/production/

# Example kustomization.yaml
cat <<EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml

namespace: production

commonLabels:
  app: myapp
  env: production

replicas:
- name: myapp
  count: 5

images:
- name: myapp
  newTag: v2.0.0
EOF
```

### Watch and Wait

```bash
# Watch resources (update in real-time)
kubectl get pods --watch
kubectl get pods -w

# Watch all resources
kubectl get all -w

# Wait for condition
kubectl wait --for=condition=Ready pod/nginx --timeout=60s

# Wait for deletion
kubectl wait --for=delete pod/nginx --timeout=60s

# Wait for deployment rollout
kubectl wait --for=condition=Available deployment/nginx --timeout=300s

# Wait for job completion
kubectl wait --for=condition=complete job/pi --timeout=600s
```

### Batch Operations

```bash
# Delete all pods with label
kubectl delete pods -l app=nginx

# Delete all evicted pods
kubectl get pods --all-namespaces --field-selector status.phase=Failed -o json | \
  kubectl delete -f -

# Restart all pods in deployment
kubectl rollout restart deployment --all

# Scale all deployments
kubectl scale deployment --all --replicas=3

# Update image for multiple deployments
kubectl set image deployment/app1 deployment/app2 app=myapp:v2

# Delete resources by type
kubectl delete pods,services,deployments -l app=myapp

# Force delete stuck resources
kubectl delete pod nginx --grace-period=0 --force

# Batch create from directory
kubectl apply -f ./manifests/ --recursive
```

### Plugin Management

```bash
# List plugins
kubectl plugin list

# Install krew (plugin manager)
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)

# Add to PATH
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"

# Install useful plugins
kubectl krew install ctx        # Switch contexts
kubectl krew install ns         # Switch namespaces
kubectl krew install tree       # Show resource hierarchy
kubectl krew install neat       # Clean up YAML output
kubectl krew install tail       # Tail logs from multiple pods
kubectl krew install view-secret # View decoded secrets

# Use plugins
kubectl ctx                     # List contexts
kubectl ctx production          # Switch context
kubectl ns development          # Switch namespace
kubectl tree deployment nginx   # Show deployment hierarchy
kubectl neat get pod nginx -o yaml  # Clean YAML
kubectl tail -l app=nginx       # Tail all nginx pods
kubectl view-secret db-secret   # View secret values
```

### Performance and Optimization

```bash
# Use short names
kubectl get po    # pods
kubectl get svc   # services
kubectl get deploy # deployments
kubectl get rs    # replicasets
kubectl get ns    # namespaces
kubectl get no    # nodes
kubectl get pv    # persistentvolumes
kubectl get pvc   # persistentvolumeclaims
kubectl get cm    # configmaps

# Aliases
alias k=kubectl
alias kg='kubectl get'
alias kd='kubectl describe'
alias kdel='kubectl delete'
alias kaf='kubectl apply -f'
alias kex='kubectl exec -it'
alias klog='kubectl logs -f'

# Reduce output
kubectl get pods --no-headers
kubectl get pods -o name

# Quick delete (skip grace period)
kubectl delete pod nginx --now

# Use server-side apply (faster)
kubectl apply --server-side -f deployment.yaml

# Cache discovery (speed up commands)
kubectl --cache-dir=/tmp/kubectl-cache get pods
```

---

# Helm

Helm is the package manager for Kubernetes, simplifying the deployment and management of applications.

## Helm Installation

### Installing Helm 3

**Linux/macOS:**
```bash
# Using installation script
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Using Homebrew (macOS)
brew install helm

# Manual installation
wget https://get.helm.sh/helm-v3.13.0-linux-amd64.tar.gz
tar -zxvf helm-v3.13.0-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm

# Verify installation
helm version
```

**Windows:**
```powershell
# Using Chocolatey
choco install kubernetes-helm

# Using Scoop
scoop install helm

# Verify
helm version
```

### Helm Configuration

```bash
# Initialize Helm configuration
helm repo add stable https://charts.helm.sh/stable

# Set up shell completion
# Bash
helm completion bash > /etc/bash_completion.d/helm

# Zsh
helm completion zsh > "${fpath[1]}/_helm"

# Configure Helm cache
export HELM_CACHE_HOME=/tmp/helm/cache
export HELM_CONFIG_HOME=/tmp/helm/config
export HELM_DATA_HOME=/tmp/helm/data

# View Helm environment
helm env
```

## Repository Management

### Adding Repositories

```bash
# Add Helm repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add nginx-stable https://helm.nginx.com/stable
helm repo add jetstack https://charts.jetstack.io
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add elastic https://helm.elastic.co
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest

# List repositories
helm repo list

# Update repositories
helm repo update

# Remove repository
helm repo remove bitnami

# Search for charts in repository
helm search repo nginx
helm search repo bitnami/

# Search Helm Hub (public charts)
helm search hub wordpress

# Get repository information
helm repo list -o yaml
```

## Chart Management

### Searching and Inspecting Charts

```bash
# Search for charts
helm search repo mysql
helm search repo database

# Search with version
helm search repo mysql --version 9.3.4

# Search all versions
helm search repo mysql --versions

# Show chart information
helm show chart bitnami/mysql

# Show chart values (default configuration)
helm show values bitnami/mysql

# Show chart README
helm show readme bitnami/mysql

# Show all chart information
helm show all bitnami/mysql

# Inspect specific version
helm show values bitnami/mysql --version 9.3.4
```

### Downloading Charts

```bash
# Download chart
helm pull bitnami/mysql

# Download and extract
helm pull bitnami/mysql --untar

# Download specific version
helm pull bitnami/mysql --version 9.3.4

# Download to specific directory
helm pull bitnami/mysql --untar --untardir ./charts

# Verify chart integrity
helm pull bitnami/mysql --verify
```

## Release Management

### Installing Charts

```bash
# Install chart
helm install my-release bitnami/mysql

# Install with custom values
helm install my-release bitnami/mysql \
  --set auth.rootPassword=secretpass \
  --set auth.database=mydb \
  --set primary.persistence.size=20Gi

# Install from values file
helm install my-release bitnami/mysql -f values.yaml

# Install in specific namespace
helm install my-release bitnami/mysql -n production --create-namespace

# Install specific version
helm install my-release bitnami/mysql --version 9.3.4

# Install with custom release name
helm install wordpress bitnami/wordpress

# Dry run (test installation)
helm install my-release bitnami/mysql --dry-run --debug

# Install from local chart
helm install my-release ./mychart/

# Install from URL
helm install my-release https://example.com/charts/myapp-1.0.0.tgz

# Install with timeout
helm install my-release bitnami/mysql --timeout 10m

# Install and wait for ready
helm install my-release bitnami/mysql --wait

# Generate name automatically
helm install bitnami/mysql --generate-name
```

### Example: Installing with Custom Values

```bash
# Create values.yaml
cat <<EOF > values.yaml
auth:
  rootPassword: "MySecretPassword123"
  database: "myapp_db"
  username: "myapp_user"
  password: "MyAppPassword456"

primary:
  persistence:
    enabled: true
    size: 50Gi
    storageClass: "fast-ssd"
  resources:
    requests:
      memory: "512Mi"
      cpu: "500m"
    limits:
      memory: "1Gi"
      cpu: "1000m"

secondary:
  replicaCount: 2
  persistence:
    enabled: true
    size: 50Gi
  resources:
    requests:
      memory: "512Mi"
      cpu: "500m"

metrics:
  enabled: true
  serviceMonitor:
    enabled: true
EOF

# Install with values file
helm install mysql bitnami/mysql -f values.yaml -n database --create-namespace
```

### Listing Releases

```bash
# List releases in current namespace
helm list

# List releases in all namespaces
helm list --all-namespaces
helm list -A

# List releases in specific namespace
helm list -n production

# List all releases (including failed/uninstalled)
helm list --all

# Filter by status
helm list --deployed
helm list --failed
helm list --pending
helm list --superseded

# Output as JSON
helm list -o json

# Output as YAML
helm list -o yaml

# Show additional columns
helm list --max 50
helm list --date
```

### Getting Release Information

```bash
# Get release status
helm status my-release

# Get release status in specific namespace
helm status my-release -n production

# Get release values
helm get values my-release

# Get all user-supplied values
helm get values my-release --all

# Get release manifest
helm get manifest my-release

# Get release hooks
helm get hooks my-release

# Get release notes
helm get notes my-release

# Get everything about release
helm get all my-release
```

### Upgrading Releases

```bash
# Upgrade release
helm upgrade my-release bitnami/mysql

# Upgrade with new values
helm upgrade my-release bitnami/mysql --set auth.rootPassword=newpass

# Upgrade with values file
helm upgrade my-release bitnami/mysql -f values.yaml

# Upgrade to specific version
helm upgrade my-release bitnami/mysql --version 9.4.0

# Upgrade and install if not exists
helm upgrade --install my-release bitnami/mysql

# Upgrade with wait
helm upgrade my-release bitnami/mysql --wait

# Upgrade with timeout
helm upgrade my-release bitnami/mysql --timeout 10m

# Force upgrade (force resource updates)
helm upgrade my-release bitnami/mysql --force

# Reuse existing values
helm upgrade my-release bitnami/mysql --reuse-values

# Reset values to chart defaults
helm upgrade my-release bitnami/mysql --reset-values

# Dry run upgrade
helm upgrade my-release bitnami/mysql --dry-run --debug

# Atomic upgrade (rollback on failure)
helm upgrade my-release bitnami/mysql --atomic

# Cleanup old resources
helm upgrade my-release bitnami/mysql --cleanup-on-fail
```

### Rolling Back Releases

```bash
# Rollback to previous revision
helm rollback my-release

# Rollback to specific revision
helm rollback my-release 2

# Rollback in specific namespace
helm rollback my-release 2 -n production

# Dry run rollback
helm rollback my-release --dry-run

# Wait for rollback completion
helm rollback my-release --wait

# Rollback with timeout
helm rollback my-release --timeout 10m

# Force rollback
helm rollback my-release --force

# Cleanup on fail
helm rollback my-release --cleanup-on-fail
```

### Viewing Release History

```bash
# View release history
helm history my-release

# View history in specific namespace
helm history my-release -n production

# Show maximum revisions
helm history my-release --max 50

# Output as JSON
helm history my-release -o json
```

### Uninstalling Releases

```bash
# Uninstall release
helm uninstall my-release

# Uninstall in specific namespace
helm uninstall my-release -n production

# Keep history (allows rollback)
helm uninstall my-release --keep-history

# Dry run uninstall
helm uninstall my-release --dry-run

# Uninstall with timeout
helm uninstall my-release --timeout 10m

# Uninstall and wait
helm uninstall my-release --wait
```

## Chart Development

### Creating a Chart

```bash
# Create new chart
helm create mychart

# Chart structure created:
# mychart/
#   Chart.yaml          # Chart metadata
#   values.yaml         # Default values
#   charts/             # Dependent charts
#   templates/          # Template files
#     deployment.yaml
#     service.yaml
#     ingress.yaml
#     _helpers.tpl      # Template helpers
#     NOTES.txt         # Post-install notes
#   .helmignore         # Files to ignore

# View created structure
tree mychart/
```

### Chart.yaml Example

```yaml
apiVersion: v2
name: mychart
description: A Helm chart for my application
type: application
version: 1.0.0
appVersion: "1.0.0"
keywords:
  - web
  - application
home: https://example.com
sources:
  - https://github.com/example/mychart
maintainers:
  - name: John Doe
    email: john@example.com
dependencies:
  - name: postgresql
    version: 12.x.x
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
  - name: redis
    version: 17.x.x
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

### values.yaml Example

```yaml
replicaCount: 3

image:
  repository: myapp
  pullPolicy: IfNotPresent
  tag: "1.0.0"

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: app-tls
      hosts:
        - app.example.com

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

postgresql:
  enabled: true
  auth:
    database: myapp
    username: myapp
  primary:
    persistence:
      size: 10Gi

redis:
  enabled: true
  auth:
    enabled: true
  master:
    persistence:
      size: 8Gi
```

### Template Example (deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: {{ .Values.service.targetPort }}
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /health
            port: http
        readinessProbe:
          httpGet:
            path: /ready
            port: http
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
        env:
        - name: DATABASE_HOST
          value: {{ include "mychart.fullname" . }}-postgresql
        - name: REDIS_HOST
          value: {{ include "mychart.fullname" . }}-redis-master
```

### Testing and Validating Charts

```bash
# Lint chart (check for errors)
helm lint mychart/

# Lint with values
helm lint mychart/ -f values.yaml

# Test chart templates
helm template my-release mychart/

# Test with values
helm template my-release mychart/ -f values.yaml

# Test specific template
helm template my-release mychart/ -s templates/deployment.yaml

# Dry run install
helm install my-release mychart/ --dry-run --debug

# Validate against Kubernetes API
helm install my-release mychart/ --dry-run --debug --validate

# Test with different values
helm template my-release mychart/ --set replicaCount=5 --debug
```

### Packaging Charts

```bash
# Package chart
helm package mychart/

# Package with specific version
helm package mychart/ --version 1.0.1

# Package with app version
helm package mychart/ --app-version 2.0.0

# Package and sign
helm package mychart/ --sign --key mykey --keyring ~/.gnupg/secring.gpg

# Package to specific directory
helm package mychart/ --destination ./packages/

# Update chart dependencies
helm dependency update mychart/

# Build chart dependencies
helm dependency build mychart/

# List chart dependencies
helm dependency list mychart/
```

### Publishing Charts

```bash
# Create chart repository index
helm repo index ./charts/

# Update repository index
helm repo index ./charts/ --merge index.yaml

# Add custom URL
helm repo index ./charts/ --url https://charts.example.com

# Serve charts locally (testing)
helm serve --repo-path ./charts/

# Upload to ChartMuseum
curl --data-binary "@mychart-1.0.0.tgz" https://charts.example.com/api/charts

# Upload to Artifactory
curl -u user:password -T mychart-1.0.0.tgz \
  "https://artifactory.example.com/artifactory/helm-local/mychart-1.0.0.tgz"
```

### Chart Dependencies

```bash
# Add dependency to Chart.yaml
cat <<EOF >> Chart.yaml
dependencies:
  - name: postgresql
    version: "12.1.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
    tags:
      - database
EOF

# Update dependencies
helm dependency update mychart/

# Build dependencies
helm dependency build mychart/

# List dependencies
helm dependency list mychart/

# Override dependency values
cat <<EOF > values.yaml
postgresql:
  enabled: true
  auth:
    username: myuser
    password: mypass
    database: mydb
EOF
```

### Chart Hooks

```yaml
# Example pre-install hook (job template)
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "mychart.fullname" . }}-db-migration
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  template:
    spec:
      containers:
      - name: db-migration
        image: "{{ .Values.migration.image }}"
        command: ["./migrate.sh"]
      restartPolicy: Never
```

**Available hooks:**
- `pre-install`: Before resources are installed
- `post-install`: After all resources are installed
- `pre-delete`: Before resources are deleted
- `post-delete`: After all resources are deleted
- `pre-upgrade`: Before resources are upgraded
- `post-upgrade`: After all resources are upgraded
- `pre-rollback`: Before resources are rolled back
- `post-rollback`: After all resources are rolled back
- `test`: When `helm test` is invoked

## Helm Plugins

### Installing Plugins

```bash
# Install plugin
helm plugin install https://github.com/databus23/helm-diff

# List installed plugins
helm plugin list

# Update plugin
helm plugin update diff

# Uninstall plugin
helm plugin uninstall diff
```

### Useful Plugins

```bash
# helm-diff: Show differences
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade my-release bitnami/mysql -f values.yaml

# helm-secrets: Manage secrets
helm plugin install https://github.com/jkroepke/helm-secrets
helm secrets view secrets.yaml

# helm-s3: S3 repository
helm plugin install https://github.com/hypnoglow/helm-s3
helm repo add my-s3-repo s3://my-bucket/charts

# helm-git: Git-based repositories
helm plugin install https://github.com/aslafy-z/helm-git
helm install my-release git+https://github.com/example/charts@charts/mychart

# helm-unittest: Unit testing
helm plugin install https://github.com/helm-unittest/helm-unittest
helm unittest mychart/

# helm-push: Push to ChartMuseum
helm plugin install https://github.com/chartmuseum/helm-push
helm cm-push mychart/ chartmuseum
```

---

# Rancher CLI

The Rancher CLI (rancher) is a command-line tool for interacting with Rancher Server.

## Rancher Installation and Setup

### Installing Rancher CLI

**Linux/macOS:**
```bash
# Download latest release
wget https://github.com/rancher/cli/releases/download/v2.7.0/rancher-linux-amd64-v2.7.0.tar.gz

# Extract
tar -xzf rancher-linux-amd64-v2.7.0.tar.gz

# Move to PATH
sudo mv rancher-v2.7.0/rancher /usr/local/bin/

# Verify installation
rancher --version
```

**Windows:**
```powershell
# Download from GitHub releases
# https://github.com/rancher/cli/releases

# Extract and add to PATH
# Verify
rancher --version
```

### Authentication

```bash
# Login to Rancher server
rancher login https://rancher.example.com --token <api-token>

# Login with token from environment
export RANCHER_URL=https://rancher.example.com
export RANCHER_TOKEN=<api-token>
rancher login

# Verify login
rancher settings

# View current context
rancher context current

# Switch context (cluster)
rancher context switch

# List contexts
rancher context list
```

### Creating API Token

1. Login to Rancher UI
2. Navigate to User Avatar → Account & API Keys
3. Click "Create API Key"
4. Provide description and scope
5. Save token (shown only once)

## Rancher Cluster Management

### Listing Clusters

```bash
# List all clusters
rancher clusters ls

# List with more details
rancher clusters ls --format json

# Get specific cluster
rancher clusters get <cluster-id>

# Describe cluster
rancher inspect <cluster-id>
```

### Creating Clusters

```bash
# Create custom cluster
rancher clusters create my-cluster \
  --rke-config cluster.yml

# Create cluster from template
rancher clusters create my-cluster \
  --cluster-template-id <template-id> \
  --cluster-template-revision-id <revision-id>

# Example RKE config (cluster.yml)
cat <<EOF > cluster.yml
nodes:
  - address: 192.168.1.10
    user: ubuntu
    role: [controlplane, etcd, worker]
  - address: 192.168.1.11
    user: ubuntu
    role: [worker]

kubernetes_version: "v1.28.0-rancher1-1"

network:
  plugin: canal

services:
  etcd:
    backup_config:
      enabled: true
      interval_hours: 6
      retention: 6
EOF
```

### Managing Clusters

```bash
# Edit cluster
rancher clusters edit <cluster-id>

# Delete cluster
rancher clusters delete <cluster-id>

# Get kubeconfig for cluster
rancher clusters kubeconfig <cluster-id> > ~/.kube/config-cluster

# Run kubectl commands on cluster
rancher kubectl --context <cluster-id> get nodes

# Update cluster
rancher clusters update <cluster-id> --rke-config cluster.yml
```

### Cluster Operations

```bash
# Create etcd snapshot
rancher clusters etcd-snapshot <cluster-id> --name snapshot-001

# Restore from snapshot
rancher clusters etcd-restore <cluster-id> --snapshot-name snapshot-001

# Rotate certificates
rancher clusters rotate-certs <cluster-id>

# Enable monitoring
rancher clusters enable-monitoring <cluster-id>

# Disable monitoring
rancher clusters disable-monitoring <cluster-id>
```

## Project and Namespace Operations

### Managing Projects

```bash
# List projects
rancher projects ls

# Create project
rancher projects create my-project \
  --cluster <cluster-id> \
  --description "My Project"

# Get project details
rancher projects get <project-id>

# Move namespace to project
rancher namespaces move <namespace> --project <project-id>

# Set project resource quotas
rancher projects set-quota <project-id> \
  --limit-cpu 10000m \
  --limit-memory 20Gi \
  --limit-pods 100
```

### Managing Namespaces

```bash
# List namespaces
rancher namespaces ls

# Create namespace
rancher namespaces create my-namespace --project <project-id>

# Delete namespace
rancher namespaces delete <namespace>

# Get namespace details
rancher namespaces get <namespace>
```

### Working with Context

```bash
# Switch to cluster context
rancher context switch <cluster-name>

# Set project context
rancher context set --project <project-id>

# View current context
rancher context current

# Use kubectl with Rancher context
rancher kubectl get pods
rancher kubectl -n production get deployments
```

## Rancher Application Management

### Catalog Management

```bash
# List catalogs
rancher catalog ls

# Add catalog
rancher catalog add my-catalog \
  --url https://charts.example.com \
  --branch main

# Refresh catalog
rancher catalog refresh my-catalog

# Delete catalog
rancher catalog delete my-catalog
```

### App Management

```bash
# List apps
rancher apps ls

# Install app
rancher apps install my-release mycatalog/myapp \
  --version 1.0.0 \
  --namespace production \
  --set replicas=3

# Install with values file
rancher apps install my-release mycatalog/myapp \
  --values values.yaml \
  --namespace production

# Upgrade app
rancher apps upgrade my-release \
  --version 1.1.0 \
  --set replicas=5

# Rollback app
rancher apps rollback my-release --revision 1

# Delete app
rancher apps delete my-release
```

### Multi-Cluster Apps

```bash
# List multi-cluster apps
rancher multiclusterapps ls

# Create multi-cluster app
rancher multiclusterapps create my-global-app \
  --template mycatalog/myapp \
  --version 1.0.0 \
  --targets cluster1,cluster2 \
  --answers answers.yaml

# Update multi-cluster app
rancher multiclusterapps upgrade my-global-app \
  --version 1.1.0

# Delete multi-cluster app
rancher multiclusterapps delete my-global-app
```

### Node Management

```bash
# List nodes
rancher nodes ls

# Describe node
rancher nodes get <node-id>

# Cordon node
rancher nodes cordon <node-id>

# Drain node
rancher nodes drain <node-id> \
  --ignore-daemonsets \
  --delete-emptydir-data

# Uncordon node
rancher nodes uncordon <node-id>

# Delete node
rancher nodes delete <node-id>
```

### User and RBAC Management

```bash
# List users
rancher users ls

# Create user
rancher users create --username john.doe --password pass123

# List global roles
rancher globalroles ls

# Assign global role
rancher globalrolebindings create \
  --user <user-id> \
  --global-role admin

# List cluster roles
rancher clusterroles ls

# Assign cluster role
rancher clusterrolebindings create \
  --user <user-id> \
  --cluster <cluster-id> \
  --cluster-role cluster-owner

# List project roles
rancher projectroles ls

# Assign project role
rancher projectrolebindings create \
  --user <user-id> \
  --project <project-id> \
  --project-role project-member
```

### Settings and Configuration

```bash
# List settings
rancher settings ls

# Get setting value
rancher settings get server-url

# Set setting value
rancher settings set server-url https://rancher.example.com

# View Rancher version
rancher settings get server-version
```

### Fleet GitOps

```bash
# List Git repos
rancher gitrepos ls

# Create Git repo
rancher gitrepos create my-repo \
  --repo-url https://github.com/myorg/fleet-config \
  --branch main \
  --paths clusters/production

# Update Git repo
rancher gitrepos update my-repo \
  --branch develop

# Force sync
rancher gitrepos force-update my-repo

# Delete Git repo
rancher gitrepos delete my-repo
```

### Monitoring and Alerting

```bash
# List alert rules
rancher alertrules ls --cluster <cluster-id>

# Create alert rule
rancher alertrules create high-cpu-alert \
  --cluster <cluster-id> \
  --expression 'cpu_usage > 80' \
  --severity critical

# List notifiers
rancher notifiers ls

# Create Slack notifier
rancher notifiers create slack-notifier \
  --type slack \
  --slack-url https://hooks.slack.com/services/XXX

# Test notifier
rancher notifiers test <notifier-id>
```

### Backup and Restore

```bash
# Create backup
rancher backup create backup-$(date +%Y%m%d) \
  --cluster <cluster-id>

# List backups
rancher backup ls

# Restore from backup
rancher backup restore <backup-id> \
  --cluster <cluster-id>

# Delete backup
rancher backup delete <backup-id>
```

### Useful Rancher CLI Tips

```bash
# Use JSON output for scripting
rancher clusters ls --format json | jq '.[]|{name:.name,state:.state}'

# Filter results
rancher projects ls | grep -i production

# Combine with kubectl
CLUSTER_ID=$(rancher clusters ls --format json | jq -r '.[]|select(.name=="production")|.id')
rancher kubectl --context ${CLUSTER_ID} get nodes

# Batch operations
rancher projects ls --format json | jq -r '.[].id' | while read project; do
  rancher projects get $project
done

# Export cluster config
rancher clusters kubeconfig my-cluster --output json > cluster-config.json

# Quiet mode (less verbose)
rancher --quiet clusters ls

# Debug mode
rancher --debug clusters ls
```

---

## Additional Resources

### kubectl Resources

- [Official kubectl Documentation](https://kubernetes.io/docs/reference/kubectl/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [kubectl Book](https://kubectl.docs.kubernetes.io/)
- [Kubernetes API Reference](https://kubernetes.io/docs/reference/kubernetes-api/)

### Helm Resources

- [Official Helm Documentation](https://helm.sh/docs/)
- [Helm Hub](https://artifacthub.io/)
- [Helm Chart Best Practices](https://helm.sh/docs/chart_best_practices/)
- [Chart Template Guide](https://helm.sh/docs/chart_template_guide/)

### Rancher CLI Resources

- [Rancher CLI Documentation](https://ranchermanager.docs.rancher.com/reference-guides/cli-with-rancher)
- [Rancher CLI GitHub](https://github.com/rancher/cli)
- [Rancher API](https://ranchermanager.docs.rancher.com/reference-guides/about-the-api)

### Community Tools

- **k9s**: Terminal UI for Kubernetes
- **kubectx/kubens**: Fast context/namespace switching
- **stern**: Multi-pod log tailing
- **kustomize**: Kubernetes native configuration management
- **skaffold**: Development workflow automation
- **telepresence**: Local development with remote cluster

---

## Quick Reference

### Common kubectl Commands

```bash
# Cluster
kubectl cluster-info
kubectl get nodes

# Resources
kubectl get all
kubectl get pods -A
kubectl describe pod <name>
kubectl logs -f <pod>
kubectl exec -it <pod> -- /bin/bash

# Apply/Create/Delete
kubectl apply -f file.yaml
kubectl create deployment nginx --image=nginx
kubectl delete pod <name>

# Scale/Update
kubectl scale deployment nginx --replicas=3
kubectl set image deployment/nginx nginx=nginx:1.21
kubectl rollout status deployment/nginx
kubectl rollout undo deployment/nginx

# Debug
kubectl debug <pod> -it --image=busybox
kubectl top nodes
kubectl top pods
```

### Common Helm Commands

```bash
# Repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo nginx

# Install/Upgrade/Uninstall
helm install my-release bitnami/nginx
helm upgrade my-release bitnami/nginx
helm uninstall my-release

# Info
helm list
helm status my-release
helm get values my-release
helm history my-release

# Rollback
helm rollback my-release 1

# Development
helm create mychart
helm lint mychart/
helm template my-release mychart/
helm package mychart/
```

### Common Rancher CLI Commands

```bash
# Login
rancher login https://rancher.example.com --token <token>

# Clusters
rancher clusters ls
rancher context switch <cluster>
rancher kubectl get nodes

# Projects/Namespaces
rancher projects ls
rancher namespaces create <name> --project <id>

# Apps
rancher apps ls
rancher apps install <release> <catalog/chart>
rancher apps upgrade <release>

# Nodes
rancher nodes ls
rancher nodes drain <node>

# Backup
rancher backup create <name>
rancher backup restore <backup-id>
```
