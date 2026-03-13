
# Kubernetes Overview

## What is Kubernetes?

Kubernetes (K8s) is an open-source container orchestration platform that automates the deployment, scaling, and management of containerized applications across clusters of hosts. Originally developed by Google and donated to the Cloud Native Computing Foundation (CNCF), Kubernetes has become the de facto standard for container orchestration.

### Key Characteristics

**Container Orchestration**: Kubernetes manages the lifecycle of containers, from scheduling and deployment to scaling and self-healing. It abstracts away the underlying infrastructure, allowing you to deploy applications consistently across various environments (on-premises, cloud, hybrid).

**Declarative Configuration**: Rather than issuing imperative commands, you declare the desired state of your application using YAML or JSON manifests. Kubernetes continuously works to maintain that state, automatically handling failures and adjustments.

**Self-Healing**: Kubernetes monitors the health of containers and nodes. If a container crashes or a node fails, Kubernetes automatically reschedules workloads to healthy nodes, ensuring application availability.

**Horizontal Scaling**: Applications can scale out (add more instances) or scale in (remove instances) based on demand, either manually or automatically using metrics like CPU, memory, or custom application metrics.

**Service Discovery and Load Balancing**: Kubernetes provides built-in mechanisms for services to discover each other and distributes traffic across multiple pod instances, ensuring even load distribution.

**Storage Orchestration**: Kubernetes can automatically mount storage systems of your choice, whether local storage, public cloud providers (AWS EBS, Azure Disk, GCP Persistent Disk), or network storage systems (NFS, iSCSI, Ceph, GlusterFS).

**Automated Rollouts and Rollbacks**: When deploying new versions, Kubernetes gradually rolls out changes while monitoring application health. If issues are detected, it can automatically roll back to the previous stable version.

**Secret and Configuration Management**: Kubernetes separates configuration from application code, allowing you to store and manage sensitive information (passwords, OAuth tokens, SSH keys) and configuration data without rebuilding container images.

## Core Components

### Control Plane Components

The control plane makes global decisions about the cluster (scheduling, detecting and responding to cluster events) and maintains the desired state of the cluster.

#### kube-apiserver
**Purpose**: The API server is the front-end for the Kubernetes control plane and the central management entity.

**Functionality**:
- Exposes the Kubernetes API via RESTful interface (HTTP/HTTPS)
- Validates and processes REST operations (GET, POST, PUT, DELETE, PATCH)
- Serves as the gateway for all administrative tasks
- Authenticates and authorizes requests using authentication plugins
- Persists cluster state to etcd
- Horizontally scalable for high availability

**Communication**: All components (kubectl, controllers, schedulers, kubelets) interact with the cluster exclusively through the API server. No component directly modifies etcd.

**Example Operations**:
- Creating/updating/deleting resources (pods, services, deployments)
- Watching for resource changes (used by controllers)
- Admission control webhooks for policy enforcement
- Custom Resource Definitions (CRDs) extension

#### etcd
**Purpose**: A distributed, consistent, and highly-available key-value store that serves as Kubernetes' backing store for all cluster data.

**Functionality**:
- Stores entire cluster state (configuration data, metadata, status)
- Provides strong consistency using the Raft consensus algorithm
- Supports watch operations for real-time change notifications
- Maintains cluster state history and versioning
- Implements leader election and distributed locking

**Architecture**:
- Typically deployed in clusters of 3, 5, or 7 nodes for fault tolerance
- Requires odd numbers for quorum-based consensus
- Only the API server communicates directly with etcd
- Critical component requiring regular backups

**Data Stored**:
- Pod definitions and their current states
- Secrets and ConfigMaps
- Node information and resource availability
- Service endpoints and routing rules
- RBAC policies and role bindings
- Custom resources and their states

**Performance Considerations**:
- SSD storage recommended for optimal performance
- Network latency between etcd nodes affects performance
- Large clusters may require dedicated etcd nodes
- Regular compaction to prevent database bloat

#### kube-scheduler
**Purpose**: Watches for newly created pods with no assigned node and selects an optimal node for them to run on.

**Scheduling Process**:
1. **Filtering**: Eliminates nodes that don't meet pod requirements
   - Resource availability (CPU, memory, GPU)
   - Node selectors and labels
   - Affinity and anti-affinity rules
   - Taints and tolerations
   - Volume availability

2. **Scoring**: Ranks remaining nodes based on priorities
   - Resource balance across nodes
   - Pod spreading for high availability
   - Node affinity preferences
   - Image locality (reduces pull time)
   - Custom priority functions

3. **Binding**: Assigns pod to highest-scored node

**Scheduling Policies**:
- **Node Affinity**: Attract pods to specific nodes
- **Pod Affinity**: Co-locate related pods
- **Pod Anti-Affinity**: Spread pods across different nodes/zones
- **Taints and Tolerations**: Repel pods from nodes unless they tolerate specific conditions
- **Priority Classes**: Schedule critical pods before lower-priority ones
- **Custom Schedulers**: Deploy additional schedulers for specialized workloads

**Advanced Features**:
- **Preemption**: Evicts lower-priority pods to make room for higher-priority ones
- **Multi-scheduler Support**: Run multiple schedulers simultaneously
- **Scheduler Extender**: External webhooks for custom scheduling logic
- **Scheduling Framework**: Plugin architecture for customization

#### kube-controller-manager
**Purpose**: Runs a collection of controllers that regulate the state of the cluster and drive it toward the desired state.

**Core Controllers**:

**Node Controller**:
- Monitors node health via heartbeats
- Marks nodes as NotReady after timeout
- Evicts pods from unhealthy nodes
- Manages node lifecycle (registration, updates)

**Replication Controller**:
- Ensures correct number of pod replicas are running
- Creates/deletes pods to match desired count
- Handles pod failures and node outages
- Works with ReplicaSets, Deployments, StatefulSets

**Endpoints Controller**:
- Populates Endpoints objects (links Services to Pods)
- Monitors Services and their matching Pods
- Updates endpoint lists as pods come and go
- Enables service discovery mechanism

**Service Account & Token Controllers**:
- Creates default ServiceAccounts for new namespaces
- Generates API access tokens for ServiceAccounts
- Injects tokens into pods as mounted secrets
- Manages token lifecycle and rotation

**Additional Controllers**:
- **Job Controller**: Manages Job and CronJob execution
- **Deployment Controller**: Handles rolling updates and rollbacks
- **StatefulSet Controller**: Manages stateful applications with stable identities
- **DaemonSet Controller**: Ensures pods run on all/specific nodes
- **Namespace Controller**: Handles namespace deletion and cleanup
- **PersistentVolume Controller**: Manages PV and PVC binding
- **ResourceQuota Controller**: Enforces resource limits per namespace

**Control Loop Pattern**:
Each controller runs an infinite loop:
1. Watch for changes to specific resources
2. Compare current state with desired state
3. Take action to reconcile differences
4. Update resource status
5. Repeat

#### cloud-controller-manager
**Purpose**: Embeds cloud-specific control logic, allowing Kubernetes to interact with the underlying cloud provider's API.

**Responsibilities**:
- **Node Controller**: Checks if deleted VMs have been removed from cloud
- **Route Controller**: Sets up routes in cloud network infrastructure
- **Service Controller**: Creates, updates, and deletes cloud load balancers
- **Volume Controller**: Creates, attaches, and mounts cloud volumes

**Cloud Provider Support**:
- AWS (ELB, EBS, Route53)
- Azure (Load Balancer, Managed Disks, DNS)
- GCP (Cloud Load Balancing, Persistent Disks, Cloud DNS)
- OpenStack, VMware vSphere, Alibaba Cloud

**Benefits**:
- Separates cloud-specific code from core Kubernetes
- Enables faster cloud provider iteration
- Allows cloud providers to maintain their own integration
- Supports out-of-tree cloud provider implementations

### Node Components

Node components run on every node, maintaining running pods and providing the Kubernetes runtime environment.

#### kubelet
**Purpose**: The primary node agent that ensures containers are running in pods according to PodSpecs.

**Responsibilities**:
- **Pod Management**:
  - Watches for PodSpecs assigned to its node
  - Ensures pod containers are running and healthy
  - Reports pod and node status to API server
  - Handles pod lifecycle (creation, updates, deletion)

- **Container Management**:
  - Pulls container images from registries
  - Starts and stops containers via container runtime
  - Monitors container resource usage
  - Collects container logs

- **Health Monitoring**:
  - Executes liveness probes (restart unhealthy containers)
  - Executes readiness probes (route traffic only to ready pods)
  - Executes startup probes (delay other probes during initialization)
  - Reports health status back to control plane

- **Volume Management**:
  - Mounts volumes into containers
  - Manages volume lifecycle (attach, mount, unmount, detach)
  - Handles various volume types (emptyDir, hostPath, PV, CSI)

- **Resource Management**:
  - Enforces resource limits (CPU, memory)
  - Implements Quality of Service (QoS) classes
  - Reports node capacity and allocatable resources

**Communication**:
- Registers node with API server
- Sends periodic heartbeats (node status updates)
- Watches API server for pod assignments
- Posts pod status updates

**Configuration**:
- Static pod support (reads manifests from local directory)
- TLS certificate management for secure communication
- Feature gates for enabling/disabling features
- Eviction policies for resource pressure

#### kube-proxy
**Purpose**: Maintains network rules on nodes to enable service communication and load balancing.

**Functionality**:
- **Service Discovery**: Watches API server for Service and Endpoints changes
- **Load Balancing**: Distributes traffic across pod replicas
- **Network Rules**: Implements packet forwarding rules
- **Connection Handling**: Manages TCP/UDP stream forwarding

**Implementation Modes**:

**iptables Mode** (default):
- Uses Linux iptables rules for packet filtering and NAT
- Random load balancing across backends
- Lower overhead, better performance
- No connection tracking or retry on failure
- Can handle tens of thousands of services

**IPVS Mode** (IP Virtual Server):
- Uses Linux IPVS kernel module
- More load balancing algorithms (round-robin, least connection, weighted)
- Better performance for large clusters (100k+ services)
- More efficient connection tracking
- Requires IPVS kernel modules

**userspace Mode** (legacy):
- kube-proxy acts as actual proxy
- Higher latency and lower performance
- Deprecated, not recommended

**Kernelspace Mode** (Windows):
- Uses Windows Host Networking Service (HNS)
- Creates policies for Windows networking stack

**Service Types Handled**:
- ClusterIP: Internal cluster traffic
- NodePort: External traffic on specific port
- LoadBalancer: Cloud provider load balancer integration
- ExternalName: DNS CNAME record mapping

#### container runtime
**Purpose**: Software responsible for running containers on the node.

**Container Runtime Interface (CRI)**:
Kubernetes uses CRI, a plugin interface that enables kubelet to use different container runtimes without recompilation.

**Supported Runtimes**:

**containerd**:
- Graduated from CNCF, industry standard
- Lightweight daemon that manages container lifecycle
- Used by Docker (Docker Engine uses containerd internally)
- Direct integration with Kubernetes via CRI plugin
- Smaller footprint than Docker
- Better performance and security isolation

**CRI-O**:
- Specifically designed for Kubernetes
- Lightweight alternative to containerd
- OCI-compliant runtime
- Minimal dependencies
- Supports multiple OCI runtimes (runc, Kata Containers)

**Docker Engine** (deprecated):
- Previously most common runtime
- Kubernetes removed dockershim in v1.24
- Still usable via cri-dockerd adapter
- More features than needed for Kubernetes
- Larger resource footprint

**Additional Runtimes**:
- **Kata Containers**: VM-based containers for enhanced security
- **gVisor**: Sandboxed runtime with application kernel
- **Firecracker**: microVM runtime for serverless workloads

**Runtime Responsibilities**:
- Image management (pull, store, delete)
- Container lifecycle (create, start, stop, remove)
- Container networking setup
- Resource isolation (cgroups, namespaces)
- Storage management


## Core Services and Features

### Workload Resources

Workload resources manage the lifecycle and scaling of containerized applications running in Kubernetes.

#### Pods
**Definition**: The smallest and simplest Kubernetes object, representing a single instance of a running process in your cluster.

**Characteristics**:
- Encapsulates one or more containers (typically one)
- Containers in a pod share network namespace (IP address, ports)
- Share storage volumes
- Scheduled together on the same node
- Ephemeral by nature (not designed to live forever)

**Pod Lifecycle**:
1. **Pending**: Pod accepted but containers not yet created
2. **Running**: Pod bound to node, all containers created, at least one running
3. **Succeeded**: All containers terminated successfully
4. **Failed**: All containers terminated, at least one failed
5. **Unknown**: Pod state cannot be determined

**Container Types**:
- **Init Containers**: Run before app containers, perform initialization tasks
- **App Containers**: Main application containers
- **Sidecar Containers**: Helper containers (logging, monitoring, proxies)
- **Ephemeral Containers**: Debugging containers injected into running pods

**Use Cases**:
- Single container per pod (most common)
- Multi-container patterns: sidecar, adapter, ambassador
- Batch jobs and one-off tasks
- Debugging and troubleshooting

**Limitations**:
- No self-healing (use higher-level controllers)
- No scaling capabilities
- No rolling updates
- Tied to single node

#### Deployments
**Definition**: Manages stateless applications, providing declarative updates for Pods and ReplicaSets.

**Functionality**:
- **Declarative Updates**: Define desired state, Kubernetes makes it happen
- **Rolling Updates**: Gradually replace old pods with new ones
- **Rollbacks**: Revert to previous versions if issues occur
- **Scaling**: Adjust number of replicas up or down
- **Self-Healing**: Replaces failed pods automatically

**Update Strategies**:

**RollingUpdate** (default):
- Gradually replaces old pods with new ones
- Configurable: `maxSurge` (extra pods during update), `maxUnavailable` (pods down during update)
- Zero-downtime deployments
- Allows canary and phased rollouts

**Recreate**:
- Terminates all old pods before creating new ones
- Causes downtime but simpler
- Useful when running multiple versions simultaneously isn't possible

**Rollback Capability**:
- Maintains revision history (default: last 10 revisions)
- Quick rollback to any previous revision
- View rollout history and status
- Pause and resume rollouts for canary testing

**Use Cases**:
- Web applications and APIs
- Microservices
- Stateless workloads
- Applications requiring rolling updates
- Multi-replica applications for high availability

**Best Practices**:
- Use health probes (readiness, liveness)
- Set resource requests and limits
- Use pod disruption budgets
- Implement progressive delivery (canary, blue-green)
- Version container images (avoid `latest` tag)

#### StatefulSets
**Definition**: Manages stateful applications requiring stable network identities and persistent storage.

**Characteristics**:
- **Stable Network Identity**: Each pod gets persistent hostname (pod-0, pod-1, pod-2)
- **Ordered Deployment**: Pods created/deleted in sequential order
- **Ordered Scaling**: Scale up/down sequentially
- **Persistent Storage**: Each pod can have unique PersistentVolumeClaim
- **Stable Pod Identity**: Pod name and hostname remain constant across rescheduling

**Ordering Guarantees**:
- **Deployment**: Pods deployed in order (0, 1, 2, ...)
- **Scaling Up**: New pods created only after previous pod is Running and Ready
- **Scaling Down**: Pods terminated in reverse order (n, n-1, n-2, ...)
- **Updates**: Rolling updates respect ordering

**Pod Management Policies**:
- **OrderedReady** (default): Strict ordering, wait for each pod
- **Parallel**: Deploy/delete all pods simultaneously (faster, less safe)

**Use Cases**:
- Databases (MySQL, PostgreSQL, MongoDB)
- Distributed systems (ZooKeeper, etcd, Kafka)
- Applications requiring stable storage
- Clustered applications needing stable network IDs
- Master-slave architectures
- Quorum-based systems

**Headless Service Requirement**:
- StatefulSets require a headless service (ClusterIP: None)
- Provides stable DNS entries for each pod
- Format: `pod-name.service-name.namespace.svc.cluster.local`

**Persistent Storage**:
- Supports `volumeClaimTemplates`
- Each pod gets unique PersistentVolumeClaim
- Storage persists across pod rescheduling
- Must manually delete PVCs when cleaning up

**Challenges**:
- More complex than Deployments
- Requires careful planning for updates
- Storage management complexity
- Backup and disaster recovery considerations

#### DaemonSets
**Definition**: Ensures that all (or specific) nodes run a copy of a pod, typically for node-level operations.

**Functionality**:
- **Automatic Pod Placement**: Pod scheduled on every node automatically
- **Node Affinity Support**: Run on subset of nodes using node selectors
- **Automatic Scaling**: As nodes are added/removed, pods follow
- **Update Strategy**: Supports rolling updates

**Update Strategies**:
- **RollingUpdate**: Gradually update pods node by node
- **OnDelete**: Manual updates, new pods created only when old ones deleted

**Use Cases**:
- **Log Collection**: Fluentd, Filebeat, Logstash agents
- **Monitoring**: Node exporters (Prometheus), Datadog agents
- **Network**: CNI plugins, kube-proxy alternatives
- **Storage**: Ceph, GlusterFS daemons
- **Security**: Intrusion detection, vulnerability scanning
- **Cluster Management**: Node problem detectors

**Scheduling**:
- Bypasses default scheduler for immediate placement
- Respects node taints (with tolerations)
- Can run on control plane nodes if tolerated
- Honors resource requests and limits

**Node Selection**:
```yaml
nodeSelector:
  disktype: ssd
  
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: node-type
          operator: In
          values: [compute]
```

**Best Practices**:
- Set resource limits to prevent node resource exhaustion
- Use priorityClassName for critical DaemonSets
- Implement health checks
- Consider update impact on cluster operations

#### Jobs
**Definition**: Creates one or more pods and ensures a specified number successfully terminate.

**Characteristics**:
- **Run to Completion**: Pods expected to exit successfully
- **Retries**: Automatic pod restart on failure
- **Parallelism**: Run multiple pods simultaneously
- **Completion Tracking**: Monitors successful completions
- **Cleanup**: Optional automatic deletion after completion

**Job Patterns**:

**Non-Parallel Jobs**:
- Single pod (default)
- Completes when pod succeeds
- Use case: One-time tasks

**Parallel Jobs with Fixed Completion Count**:
- Multiple pods, specific success count
- `.spec.completions`: number of successful completions needed
- `.spec.parallelism`: number of pods running simultaneously
- Use case: Batch processing with known size

**Parallel Jobs with Work Queue**:
- Multiple pods, no completion count
- Pods coordinate via external queue
- Job completes when at least one pod succeeds and all pods terminate
- Use case: Distributed processing

**Failure Handling**:
- **backoffLimit**: Maximum retries before marking job as failed (default: 6)
- **activeDeadlineSeconds**: Maximum time for job execution
- **restartPolicy**: OnFailure (restart container) or Never (create new pod)

**Use Cases**:
- Data processing and ETL pipelines
- Batch computations
- Database migrations
- Backup operations
- One-time administrative tasks
- Report generation

#### CronJobs
**Definition**: Creates Jobs on a repeating schedule, similar to cron in Unix/Linux.

**Characteristics**:
- **Schedule Format**: Cron expression (minute hour day month weekday)
- **Timezone Support**: Specify timezone for schedule
- **Job Management**: Creates and manages Job objects
- **History Limits**: Retains configurable number of completed/failed jobs
- **Concurrency Policy**: Controls overlapping job executions

**Schedule Format**:
```
# ┌───────────── minute (0 - 59)
# │ ┌───────────── hour (0 - 23)
# │ │ ┌───────────── day of month (1 - 31)
# │ │ │ ┌───────────── month (1 - 12)
# │ │ │ │ ┌───────────── day of week (0 - 6) (Sunday to Saturday)
# │ │ │ │ │
# * * * * *
```

**Examples**:
- `*/5 * * * *`: Every 5 minutes
- `0 */2 * * *`: Every 2 hours
- `0 9 * * 1-5`: 9 AM weekdays
- `0 0 * * 0`: Midnight every Sunday

**Concurrency Policies**:
- **Allow**: Multiple jobs can run simultaneously (default)
- **Forbid**: Skip new job if previous still running
- **Replace**: Cancel current job and start new one

**Configuration Options**:
- **startingDeadlineSeconds**: Deadline for starting job if missed
- **successfulJobsHistoryLimit**: Number of successful jobs to keep (default: 3)
- **failedJobsHistoryLimit**: Number of failed jobs to keep (default: 1)
- **suspend**: Pause scheduling without deleting resource

**Use Cases**:
- Scheduled backups
- Report generation
- Data synchronization
- Cache warming
- Log rotation
- Certificate renewal
- Periodic health checks
- Automated cleanup tasks

**Best Practices**:
- Set reasonable `activeDeadlineSeconds` to prevent stuck jobs
- Use `Forbid` concurrency policy for critical jobs
- Monitor job execution and failures
- Set appropriate history limits
- Use idempotent operations
- Handle timezone changes carefully

#### ReplicaSets
**Definition**: Maintains a stable set of replica pods running at any given time.

**Functionality**:
- Ensures specified number of pod replicas are running
- Creates pods if count is too low
- Deletes pods if count is too high
- Uses label selectors to identify pods to manage

**Relationship with Deployments**:
- Deployments create and manage ReplicaSets automatically
- Direct ReplicaSet use rarely needed (use Deployments instead)
- Each Deployment revision creates a new ReplicaSet
- Old ReplicaSets retained for rollback capability

**Label Selectors**:
- **Equality-based**: `key=value`, `key!=value`
- **Set-based**: `key in (value1, value2)`, `key notin (value1, value2)`

**Use Cases**:
- Usually managed by Deployments
- Direct use only when custom update orchestration needed
- Advanced scenarios requiring manual ReplicaSet control

**Limitations**:
- No rolling update strategy (use Deployments)
- No rollback capability
- Manual management required

### Networking

Kubernetes networking provides connectivity and service discovery for pods, enabling complex microservice architectures.

#### Services
**Definition**: An abstract way to expose an application running on a set of pods as a network service.

**Purpose**:
- Provide stable IP address and DNS name for dynamic pod IPs
- Load balance traffic across multiple pod replicas
- Enable service discovery within cluster
- Support external access to applications

**Service Types**:

**ClusterIP** (default):
- **Scope**: Internal cluster communication only
- **IP Address**: Virtual IP (VIP) accessible only within cluster
- **Use Case**: Internal microservice communication, databases
- **DNS**: Accessible via `service-name.namespace.svc.cluster.local`
- **Example**: Backend API services, internal databases

**NodePort**:
- **Scope**: External access via node IP and static port
- **Port Range**: 30000-32767 (configurable)
- **Mechanism**: Creates ClusterIP + opens port on every node
- **Use Case**: Development, small deployments, non-production
- **Access**: `<NodeIP>:<NodePort>`
- **Limitations**: One service per port, requires node firewall management

**LoadBalancer**:
- **Scope**: External access via cloud provider's load balancer
- **Mechanism**: Creates NodePort + provisions external LB
- **Cloud Integration**: AWS ELB/ALB/NLB, Azure Load Balancer, GCP Load Balancer
- **Use Case**: Production external services
- **Features**: Health checks, SSL termination, DDoS protection
- **Cost**: Each service provisions a cloud load balancer (can be expensive)

**ExternalName**:
- **Scope**: Maps service to external DNS name
- **Mechanism**: Returns CNAME record for external service
- **Use Case**: Database hosted outside cluster, external APIs
- **No Proxying**: DNS-only, no load balancing or proxying
- **Example**: Migrating services gradually to/from cluster

**Headless Services** (ClusterIP: None):
- **Purpose**: Direct pod-to-pod communication without load balancing
- **DNS**: Returns pod IPs instead of service IP
- **Use Case**: StatefulSets, custom load balancing, peer discovery
- **Format**: `pod-name.service-name.namespace.svc.cluster.local`

**Service Discovery**:
- **DNS**: CoreDNS provides service name resolution
- **Environment Variables**: Injected into pods at startup (legacy)
- **Service Mesh**: Advanced discovery with Istio, Linkerd

**Session Affinity**:
- **ClientIP**: Routes requests from same client to same pod
- **Use Case**: Stateful sessions, WebSockets
- **Configuration**: `service.spec.sessionAffinity: ClientIP`

**Endpoint Slices**:
- Scalable way to track network endpoints
- Replaces Endpoints API for large clusters
- Supports IPv4, IPv6, FQDN endpoints
- Better performance with thousands of pods

#### Ingress
**Definition**: API object that manages external access to services via HTTP/HTTPS, providing routing, SSL termination, and name-based virtual hosting.

**Capabilities**:
- **Host-based Routing**: Route traffic based on hostname
- **Path-based Routing**: Route traffic based on URL path
- **SSL/TLS Termination**: Handle HTTPS, offload encryption from backend
- **Load Balancing**: Distribute traffic across service backends
- **Authentication**: Basic auth, OAuth, custom auth
- **Rate Limiting**: Control request rates
- **Rewrites and Redirects**: Modify requests and responses

**Ingress Controllers**:
Ingress resource requires an Ingress Controller to function:

**NGINX Ingress Controller**:
- Most popular, well-maintained by Kubernetes community
- High performance, extensive features
- ConfigMaps for advanced configuration
- SSL passthrough, TCP/UDP support
- WebSocket support

**Traefik**:
- Modern cloud-native ingress controller
- Automatic service discovery
- Let's Encrypt integration
- Dashboard UI, middleware support
- Kubernetes-native CRDs

**HAProxy Ingress**:
- Enterprise-grade load balancing
- Advanced routing algorithms
- Blue-green deployments
- Circuit breakers, rate limiting

**Cloud-Specific**:
- **AWS ALB Ingress Controller**: Integrates with AWS Application Load Balancer
- **GCE Ingress Controller**: Google Cloud Load Balancer integration
- **Azure Application Gateway**: Azure-native ingress

**Service Mesh Ingress**:
- **Istio Gateway**: Advanced traffic management
- **Ambassador**: API Gateway features
- **Gloo**: Function-level routing

**Routing Examples**:

**Host-based**:
```yaml
rules:
- host: api.example.com
  http:
    paths:
    - path: /
      backend:
        service:
          name: api-service
- host: web.example.com
  http:
    paths:
    - path: /
      backend:
        service:
          name: web-service
```

**Path-based**:
```yaml
rules:
- host: example.com
  http:
    paths:
    - path: /api
      backend:
        service:
          name: api-service
    - path: /web
      backend:
        service:
          name: web-service
```

**TLS/SSL Configuration**:
- Certificate stored in Kubernetes Secret
- SNI (Server Name Indication) support
- Multiple certificates for different hosts
- cert-manager for automatic certificate management

**Use Cases**:
- External API access
- Web application hosting
- Multi-tenant applications
- Microservices gateway
- SSL termination point

#### Network Policies
**Definition**: Specifications for controlling traffic flow between pods and network endpoints.

**Functionality**:
- **Pod-level Firewall**: Control ingress and egress traffic
- **Label-based Selection**: Use labels to define policy scope
- **Namespace Isolation**: Isolate traffic between namespaces
- **Default Deny**: Implement zero-trust networking

**Policy Types**:
- **Ingress**: Controls incoming traffic to pods
- **Egress**: Controls outgoing traffic from pods
- **Both**: Combine ingress and egress rules

**Selectors**:
- **podSelector**: Select pods by labels
- **namespaceSelector**: Select namespaces by labels
- **ipBlock**: CIDR-based IP ranges
- **port**: TCP/UDP port numbers

**CNI Plugin Requirement**:
Network policies require CNI plugin support:
- **Calico**: Most feature-complete, global policies
- **Cilium**: eBPF-based, L7 policies, observability
- **Weave Net**: Simple setup, encryption support
- **Canal**: Combination of Flannel and Calico
- **Not supported**: Flannel (alone), kindnet

**Use Cases**:
- **Microservice Isolation**: Limit communication between services
- **PCI/HIPAA Compliance**: Enforce network segmentation
- **Multi-tenancy**: Isolate tenant traffic
- **Database Access Control**: Restrict DB access to specific services
- **Egress Filtering**: Control external access

**Default Deny Pattern**:
```yaml
# Deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress

# Deny all egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
spec:
  podSelector: {}
  policyTypes:
  - Egress
```

**Best Practices**:
- Start with default deny, explicitly allow needed traffic
- Use namespaces for coarse-grained isolation
- Test policies in non-production first
- Document policy intent
- Monitor denied connections
- Use tooling for policy visualization (kubectl-np-viewer)

#### DNS (CoreDNS)
**Definition**: Cluster DNS server that provides name resolution for services and pods.

**Functionality**:
- **Service Discovery**: Resolve service names to IPs
- **Pod DNS**: Optional DNS records for pods
- **External DNS**: Forward external queries to upstream resolvers
- **Custom DNS**: Add custom DNS entries

**DNS Records**:

**Service A Records**:
- Format: `service-name.namespace.svc.cluster.local`
- Returns: Service ClusterIP
- Example: `my-service.default.svc.cluster.local` → `10.96.0.10`

**Headless Service Records**:
- Returns: Pod IPs directly
- Example: `pod-0.my-service.default.svc.cluster.local` → Pod IP

**Pod Records** (if enabled):
- Format: `pod-ip-address.namespace.pod.cluster.local`
- Example: `10-244-1-5.default.pod.cluster.local`

**Search Domains**:
Pods get search domain list for short name resolution:
```
namespace.svc.cluster.local
svc.cluster.local
cluster.local
```

**DNS Policy**:
- **ClusterFirst** (default): Query CoreDNS first, then upstream
- **ClusterFirstWithHostNet**: For pods using host networking
- **Default**: Use node's DNS settings
- **None**: Custom DNS configuration via `dnsConfig`

**CoreDNS Features**:
- **Caching**: Reduce upstream DNS queries
- **Forwarding**: Custom upstream resolvers
- **Prometheus Metrics**: DNS query monitoring
- **Logging**: Debug DNS issues
- **Custom Plugins**: Extend functionality
- **Rewrite Rules**: Modify DNS queries/responses

**Troubleshooting**:
- Check CoreDNS pods status
- Verify service endpoints
- Test with `nslookup` or `dig` from pod
- Check CoreDNS logs
- Validate network policies
- Review CoreDNS ConfigMap configuration

### Storage

Kubernetes provides a sophisticated storage orchestration system that abstracts underlying storage implementations and enables portability across environments.

#### Persistent Volumes (PV)
**Definition**: Cluster-wide storage resources provisioned by administrators or dynamically created using StorageClasses.

**Characteristics**:
- **Cluster Resource**: Exists independently of pods
- **Lifecycle**: Independent of pod lifecycle
- **Provisioning**: Static (pre-created) or dynamic (on-demand)
- **Reclaim Policy**: Determines what happens when claim is deleted
- **Access Modes**: Define how volume can be mounted

**Access Modes**:
- **ReadWriteOnce (RWO)**: Mounted read-write by single node
- **ReadOnlyMany (ROX)**: Mounted read-only by multiple nodes
- **ReadWriteMany (RWX)**: Mounted read-write by multiple nodes
- **ReadWriteOncePod (RWOP)**: Mounted read-write by single pod (Kubernetes 1.22+)

**Reclaim Policies**:
- **Retain**: Manual reclamation, data preserved after claim deletion
- **Delete**: Automatically delete volume and data
- **Recycle** (deprecated): Basic scrub (`rm -rf /volume/*`)

**Volume Modes**:
- **Filesystem** (default): Mounted as directory
- **Block**: Raw block device for databases, specialized apps

**Capacity**:
- Storage size specified (e.g., 10Gi, 100Gi)
- Node affinity for zone-specific storage
- Volume expansion support (if storage class allows)

**Phases**:
1. **Available**: Free, not yet bound to claim
2. **Bound**: Bound to PVC
3. **Released**: Claim deleted, but not reclaimed
4. **Failed**: Automatic reclamation failed

#### Persistent Volume Claims (PVC)
**Definition**: User's request for storage, similar to how pods consume node resources.

**Purpose**:
- **Abstraction**: Developers request storage without knowing infrastructure
- **Dynamic Provisioning**: Automatically creates PV if matching StorageClass exists
- **Binding**: Kubernetes matches PVC to suitable PV
- **Portability**: Same PVC works across different environments

**Binding Process**:
1. User creates PVC specifying size, access mode, storage class
2. Kubernetes finds matching PV (or creates one dynamically)
3. PV bound to PVC (1:1 relationship)
4. Bound PVC can be used by pods
5. Binding persists until PVC deleted

**Request Specifications**:
- **Resources**: Storage size (requests.storage)
- **Access Modes**: RWO, ROX, RWX
- **Storage Class**: Optional, specifies provisioner
- **Selector**: Label-based PV selection
- **Volume Mode**: Filesystem or Block

**Use in Pods**:
```yaml
volumes:
- name: data
  persistentVolumeClaim:
    claimName: my-pvc
```

**Expansion**:
- Supported by some storage classes
- Edit PVC to request larger size
- May require pod restart
- Cannot shrink volumes

#### Storage Classes
**Definition**: Provides a way to describe different "classes" of storage with different performance characteristics, backup policies, or provisioning parameters.

**Purpose**:
- **Dynamic Provisioning**: Automatically create PVs on-demand
- **Quality of Service**: Different tiers (SSD, HDD, NVMe)
- **Policy-based Storage**: Encryption, replication, backups
- **Cloud Abstraction**: Works across cloud providers

**Components**:
- **Provisioner**: Plugin that creates volumes (kubernetes.io/aws-ebs, kubernetes.io/azure-disk)
- **Parameters**: Provisioner-specific settings (disk type, IOPS, zones)
- **Reclaim Policy**: What happens when PVC deleted
- **Volume Binding Mode**: Immediate or WaitForFirstConsumer
- **Allowed Topologies**: Restrict volume to specific zones/regions

**Volume Binding Modes**:

**Immediate**:
- PV created as soon as PVC created
- May create volume in wrong zone
- Simpler, but less flexible

**WaitForFirstConsumer**:
- Delays PV creation until pod scheduled
- Ensures volume created in same zone as pod
- Recommended for multi-zone clusters
- Avoids cross-zone mount issues

**Cloud Provider Examples**:

**AWS EBS**:
```yaml
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3  # gp2, gp3, io1, io2, st1, sc1
  iops: "3000"
  encrypted: "true"
  kmsKeyId: arn:aws:kms:...
```

**Azure Disk**:
```yaml
provisioner: kubernetes.io/azure-disk
parameters:
  storageaccounttype: Premium_LRS  # Standard_LRS, Premium_LRS, StandardSSD_LRS
  kind: Managed
```

**GCE Persistent Disk**:
```yaml
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd  # pd-standard, pd-ssd, pd-balanced
  replication-type: regional-pd
```

**Default Storage Class**:
- Annotated with `storageclass.kubernetes.io/is-default-class: "true"`
- Used when PVC doesn't specify storage class
- Only one default allowed per cluster

**Use Cases**:
- **Fast Storage**: SSD-backed for databases
- **Slow Storage**: HDD for logs, backups
- **Encrypted Storage**: Compliance requirements
- **Replicated Storage**: High availability
- **Regional Storage**: Multi-zone replication

#### Volume Types

Kubernetes supports numerous volume types for different use cases:

**Ephemeral Volumes**:

**emptyDir**:
- Created when pod assigned to node
- Initially empty
- Shared among containers in pod
- Deleted when pod removed
- Use: Scratch space, cache, shared data between containers
- Can be memory-backed (`emptyDir.medium: Memory`)

**configMap**:
- Mount ConfigMap as files
- Read-only
- Automatically updated when ConfigMap changes
- Use: Configuration files, environment-specific settings

**secret**:
- Mount Secrets as files
- Read-only, stored in tmpfs (memory)
- Not written to disk
- Use: Credentials, certificates, API keys

**downwardAPI**:
- Exposes pod metadata as files
- Pod name, namespace, labels, annotations
- Resource requests/limits
- Use: Application needs to know pod info

**Persistent Volumes**:

**hostPath**:
- Mounts file/directory from host node
- Survives pod restarts (if on same node)
- Security risk (access to host filesystem)
- Use: Development, node-specific data
- **Warning**: Not recommended for production

**local**:
- Similar to hostPath but with node affinity
- Better for StatefulSets
- Survives pod reschedule on same node
- Use: High-performance local SSD storage

**Network Storage**:

**nfs**:
- Network File System
- ReadWriteMany support
- Shared across multiple pods/nodes
- Use: Shared file storage

**cephfs, glusterfs**:
- Distributed file systems
- High availability and scalability
- RWX support
- Use: Large-scale shared storage

**iscsi**:
- iSCSI (SCSI over IP)
- Block storage
- Use: SAN storage

**Cloud Storage**:

**awsElasticBlockStore (EBS)**:
- AWS block storage
- ReadWriteOnce
- Zonal (single AZ)
- High performance

**azureDisk**:
- Azure Managed Disks
- ReadWriteOnce
- Zonal availability
- Different performance tiers

**gcePersistentDisk**:
- Google Compute Engine Persistent Disk
- ReadWriteOnce (RWX for ROX)
- Regional replication available
- Automatic encryption

**Container Storage Interface (CSI)**:

**Modern Approach**:
- Standardized plugin interface
- Out-of-tree storage plugins
- Vendor-maintained drivers
- Supports advanced features (snapshots, cloning, expansion)

**Popular CSI Drivers**:
- AWS EBS CSI
- Azure Disk/File CSI
- GCE PD CSI
- Ceph CSI (RBD, CephFS)
- NetApp Trident
- Pure Storage
- Portworx
- Longhorn (Cloud Native Storage)

**CSI Features**:
- Volume snapshots
- Volume cloning
- Volume expansion
- Topology-aware provisioning
- Raw block volumes
- Ephemeral inline volumes

**Projected Volumes**:
- Combines multiple volume sources into single directory
- Can include: secrets, configMaps, downwardAPI, serviceAccountToken
- Use: Bundle multiple configs into one mount point

**Storage Best Practices**:
- Use StorageClasses for dynamic provisioning
- Implement backup strategies
- Monitor storage utilization
- Set appropriate reclaim policies
- Use WaitForFirstConsumer binding mode
- Implement volume snapshots for backups
- Consider costs (cloud storage can be expensive)
- Test disaster recovery procedures
- Use CSI drivers for modern features
- Apply the principle of least privilege for hostPath

### Configuration & Secrets

Kubernetes provides mechanisms to separate configuration and sensitive data from application code, enabling environment-agnostic deployments.

#### ConfigMaps
**Definition**: API object used to store non-confidential configuration data in key-value pairs.

**Purpose**:
- **Decouple Configuration**: Separate config from container images
- **Environment Portability**: Same image, different configs
- **Dynamic Updates**: Change config without rebuilding images
- **Reusability**: Share config across multiple pods
- **Version Control**: Track config changes in Git

**Data Storage Formats**:
- **Literal Values**: Simple key-value pairs
- **Files**: Entire file contents
- **Directories**: Multiple files from directory
- **Environment Files**: .env format

**Consumption Methods**:

**Environment Variables**:
```yaml
envFrom:
- configMapRef:
    name: app-config
# Or specific keys:
env:
- name: LOG_LEVEL
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: log_level
```

**Volume Mounts**:
```yaml
volumes:
- name: config
  configMap:
    name: app-config
volumeMounts:
- name: config
  mountPath: /etc/config
  readOnly: true
```

**Command Arguments**:
```yaml
command: ["/bin/app"]
args: ["--config=$(CONFIG_FILE)"]
env:
- name: CONFIG_FILE
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: config_file_path
```

**Update Behavior**:
- **Environment Variables**: Not updated (requires pod restart)
- **Volume Mounts**: Updated automatically (with delay, typically 10-60s)
- **Immutability**: Can mark ConfigMap as immutable for performance

**Size Limits**:
- Maximum 1 MiB per ConfigMap
- Stored in etcd (affects cluster performance)
- Large configs should use external config stores

**Use Cases**:
- Application configuration files
- Database connection strings (non-sensitive)
- Feature flags
- Environment-specific settings
- Command-line arguments
- Logging configuration

**Best Practices**:
- Use descriptive names with versioning
- Validate config before deployment
- Document required configuration keys
- Use immutable ConfigMaps for production
- Store in Git for version control
- Separate environment-specific configs
- Avoid storing large files

#### Secrets
**Definition**: Objects used to store sensitive information such as passwords, OAuth tokens, SSH keys, and TLS certificates.

**Purpose**:
- **Security**: Encrypted at rest (if configured)
- **Access Control**: RBAC-based access
- **Auditing**: Track secret access
- **Rotation**: Update secrets without rebuilding images
- **Separation**: Keep sensitive data out of source code

**Secret Types**:

**Opaque** (default):
- Arbitrary user-defined data
- Most common type
- Base64 encoded

**kubernetes.io/service-account-token**:
- Service account authentication tokens
- Automatically created for service accounts
- Used for pod-to-API-server communication

**kubernetes.io/dockerconfigjson**:
- Docker registry credentials
- Used for pulling private images
- Format: Docker config.json

**kubernetes.io/basic-auth**:
- Basic authentication credentials
- Username and password

**kubernetes.io/ssh-auth**:
- SSH private key
- Used for Git operations, SSH access

**kubernetes.io/tls**:
- TLS certificate and key
- Used for Ingress, service mesh
- Contains: tls.crt, tls.key

**bootstrap.kubernetes.io/token**:
- Bootstrap tokens for node joining
- Used by kubeadm

**Consumption Methods**:

**Environment Variables**:
```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password
```

**Volume Mounts**:
```yaml
volumes:
- name: secret-volume
  secret:
    secretName: my-secret
    defaultMode: 0400  # Read-only for owner
volumeMounts:
- name: secret-volume
  mountPath: /etc/secrets
  readOnly: true
```

**Image Pull Secrets**:
```yaml
imagePullSecrets:
- name: registry-secret
```

**Security Features**:

**Encryption at Rest**:
- Requires enabling EncryptionConfiguration
- Encrypts secrets in etcd
- Multiple encryption providers (AES-CBC, AES-GCM, KMS)
- Not enabled by default

**Access Control**:
- RBAC policies control secret access
- Service accounts can be restricted
- Namespace isolation
- Can prevent secret listing (only get by name)

**Memory-backed Storage**:
- Secrets mounted as volumes stored in tmpfs
- Never written to disk on nodes
- Cleared when pod deleted

**Best Practices**:

**Creation and Management**:
- Never commit secrets to Git
- Use external secret management (Vault, AWS Secrets Manager)
- Rotate secrets regularly
- Use strong, random passwords
- Minimize secret access scope

**External Secret Management**:

**HashiCorp Vault**:
- Centralized secret storage
- Dynamic secrets
- Encryption as a service
- Audit logging
- Kubernetes auth method

**Sealed Secrets**:
- Encrypt secrets for Git storage
- Controller decrypts in-cluster
- Public key encryption
- GitOps-friendly

**External Secrets Operator**:
- Syncs secrets from external sources
- Supports: AWS Secrets Manager, GCP Secret Manager, Azure Key Vault, Vault
- Automatic rotation
- Multiple backend support

**AWS Secrets Manager / Parameter Store**:
- Native AWS integration
- Automatic rotation
- Cross-region replication
- IAM-based access control

**GCP Secret Manager**:
- Google Cloud native
- Versioning and rotation
- IAM integration
- Audit logging

**Azure Key Vault**:
- Azure native secret storage
- Hardware security modules (HSM)
- Certificate management
- RBAC integration

**Security Considerations**:
- Enable encryption at rest
- Use external secret management for production
- Implement secret rotation
- Audit secret access
- Use pod security policies/admission
- Limit secret scope (per-service secrets)
- Use workload identity (cloud-specific)
- Consider secret zero-ization tools
- Monitor secret access patterns

#### Environment Variables
**Definition**: Key-value pairs injected into container runtime environment.

**Sources**:
- **Static Values**: Hardcoded in pod spec
- **ConfigMaps**: Non-sensitive configuration
- **Secrets**: Sensitive data
- **Downward API**: Pod/container metadata
- **Field References**: Resource requests/limits

**Injection Methods**:

**Direct Definition**:
```yaml
env:
- name: APP_ENV
  value: "production"
- name: PORT
  value: "8080"
```

**From ConfigMap**:
```yaml
# All keys from ConfigMap
envFrom:
- configMapRef:
    name: app-config
    prefix: APP_  # Optional prefix

# Specific keys
env:
- name: LOG_LEVEL
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: log_level
```

**From Secret**:
```yaml
# All keys from Secret
envFrom:
- secretRef:
    name: app-secret

# Specific keys
env:
- name: API_KEY
  valueFrom:
    secretKeyRef:
      name: app-secret
      key: api_key
```

**From Downward API**:
```yaml
env:
- name: POD_NAME
  valueFrom:
    fieldRef:
      fieldPath: metadata.name
- name: POD_NAMESPACE
  valueFrom:
    fieldRef:
      fieldPath: metadata.namespace
- name: CPU_REQUEST
  valueFrom:
    resourceFieldRef:
      containerName: app
      resource: requests.cpu
```

**Available Field References**:
- `metadata.name`: Pod name
- `metadata.namespace`: Pod namespace
- `metadata.labels['<KEY>']`: Specific label
- `metadata.annotations['<KEY>']`: Specific annotation
- `spec.serviceAccountName`: Service account
- `spec.nodeName`: Node name
- `status.hostIP`: Node IP
- `status.podIP`: Pod IP

**Available Resource References**:
- `requests.cpu`: CPU request
- `requests.memory`: Memory request
- `requests.ephemeral-storage`: Storage request
- `limits.cpu`: CPU limit
- `limits.memory`: Memory limit
- `limits.ephemeral-storage`: Storage limit

**Precedence and Conflicts**:
- Later definitions override earlier ones
- `env` overrides `envFrom`
- Prefix helps avoid conflicts with `envFrom`
- Duplicate keys use last definition

**Use Cases**:
- Application runtime configuration
- Feature flags
- Service URLs and ports
- Database connection strings
- API keys and credentials
- Debug/logging levels
- Pod identity and metadata

**Best Practices**:
- Use ConfigMaps for non-sensitive data
- Use Secrets for sensitive data
- Avoid hardcoding values
- Document required environment variables
- Use meaningful names (UPPER_SNAKE_CASE convention)
- Validate required variables in application startup
- Consider using .env files during development
- Be cautious with variable expansion
- Use prefixes to organize variables
- Monitor environment variable size (limits exist)

### Security

Kubernetes provides multiple layers of security controls to protect cluster resources, workloads, and data.

#### RBAC (Role-Based Access Control)
**Definition**: Authorization mechanism that regulates access to Kubernetes resources based on roles assigned to users, groups, or service accounts.

**Core Concepts**:

**Subjects** (Who):
- **Users**: Human operators (external identity)
- **Groups**: Collections of users
- **ServiceAccounts**: Pod identities (Kubernetes-managed)

**Resources** (What):
- Kubernetes API objects (pods, services, secrets, etc.)
- Resource types and API groups
- Non-resource endpoints (/healthz, /metrics)

**Verbs** (Actions):
- `get`, `list`, `watch`: Read operations
- `create`, `update`, `patch`: Modification operations
- `delete`, `deletecollection`: Deletion operations
- `*`: All verbs (use cautiously)

**RBAC Objects**:

**Role**:
- Namespace-scoped permissions
- Grants access to resources within single namespace
- Example: Allow reading pods in "development" namespace

**ClusterRole**:
- Cluster-wide permissions
- Can grant access to:
  - Cluster-scoped resources (nodes, persistentVolumes)
  - Non-resource endpoints
  - Namespaced resources across all namespaces
- Example: Allow listing all pods cluster-wide

**RoleBinding**:
- Grants permissions defined in Role to subjects
- Namespace-scoped (applies within one namespace)
- Can reference Role or ClusterRole
- Example: Bind "pod-reader" role to user "jane" in "dev" namespace

**ClusterRoleBinding**:
- Grants permissions defined in ClusterRole to subjects
- Cluster-wide scope
- Affects all namespaces
- Example: Bind "cluster-admin" to user "admin"

**Permission Model**:
- **Additive Only**: No deny rules (only allow)
- **Least Privilege**: Start with minimal permissions
- **Explicit Grants**: No implicit permissions
- **Aggregation**: ClusterRoles can aggregate other ClusterRoles

**Default Roles**:

**cluster-admin**:
- Full cluster access
- All resources, all verbs
- Cluster-wide scope
- Should be tightly controlled

**admin**:
- Full access within namespace
- Can create roles and rolebindings
- Cannot modify namespace or resource quotas
- Suitable for namespace administrators

**edit**:
- Read/write access to most resources
- Cannot view or modify roles/rolebindings
- Cannot view secrets by default
- Suitable for developers

**view**:
- Read-only access to most resources
- Cannot view secrets
- Cannot view roles/rolebindings
- Suitable for monitoring, auditing

**Best Practices**:
- Use principle of least privilege
- Create specific roles for each use case
- Avoid wildcard permissions (`*`)
- Regularly audit RBAC policies
- Use namespace isolation
- Separate admin and developer roles
- Implement break-glass procedures
- Document role purposes
- Use groups instead of individual users
- Test permissions with `kubectl auth can-i`

**Advanced Features**:

**Aggregated ClusterRoles**:
```yaml
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-custom: "true"
```

**Impersonation**:
- Test permissions as another user
- `kubectl --as=user --as-group=group`
- Requires impersonation permissions

**RBAC Tools**:
- **kubectl-who-can**: Find who can perform action
- **rbac-lookup**: Reverse lookup for permissions
- **rakkess**: Show access matrix
- **rbac-police**: Audit RBAC policies

#### Service Accounts
**Definition**: Kubernetes-managed identities for pods to authenticate with the API server.

**Purpose**:
- Provide pod identity
- Enable pod-to-API-server authentication
- Namespace-scoped
- Automatic token generation
- RBAC integration

**Automatic Creation**:
- Default service account created per namespace
- Named "default"
- Limited permissions (discovery only)
- Automatically mounted into pods

**Token Management**:

**Legacy Tokens** (pre-1.24):
- Long-lived, stored in secrets
- Automatically created
- Manually rotated
- Security concern (never expire)

**Bound Service Account Tokens** (1.22+):
- Time-limited (default 1 hour)
- Audience-bound
- Automatically rotated
- Projected volume mount
- More secure (short-lived)

**Token Projection**:
```yaml
volumes:
- name: token
  projected:
    sources:
    - serviceAccountToken:
        path: token
        expirationSeconds: 3600
        audience: api
```

**Use Cases**:
- CI/CD pipelines accessing API
- Applications managing Kubernetes resources
- Service mesh identity
- Custom controllers and operators
- Monitoring and logging agents

**Best Practices**:
- Create dedicated service accounts per application
- Grant minimal RBAC permissions
- Disable auto-mounting if not needed
- Use workload identity (cloud providers)
- Implement pod security admission
- Regular permission audits
- Use bound tokens (modern approach)

#### Pod Security Policies / Standards
**Definition**: Mechanisms to enforce security constraints on pod specifications.

**Pod Security Policies (PSP)** - Deprecated in 1.21, Removed in 1.25:
- Cluster-level admission controller
- Validates pod security settings
- Can mutate pods to apply defaults
- Complex to configure
- Replaced by Pod Security Admission

**Pod Security Standards (PSS)**:
Three policies enforced at namespace level:

**Privileged**:
- Unrestricted (no restrictions)
- For system-level workloads
- Allows all privileged operations
- Use case: CNI plugins, storage drivers, monitoring agents

**Baseline**:
- Minimally restrictive
- Prevents known privilege escalations
- Allows common deployment patterns
- Use case: Most applications

**Restricted**:
- Heavily restricted (security hardened)
- Follow current pod hardening best practices
- More restrictive than baseline
- Use case: Security-sensitive workloads

**Pod Security Admission**:
- Built-in admission controller (1.23+)
- Namespace-level enforcement
- Three modes per policy:

**enforce**: Reject non-compliant pods
**audit**: Log violations, allow pod
**warn**: Send warning to user, allow pod

**Namespace Labels**:
```yaml
pod-security.kubernetes.io/enforce: restricted
pod-security.kubernetes.io/audit: restricted
pod-security.kubernetes.io/warn: restricted
pod-security.kubernetes.io/enforce-version: latest
```

**Restricted Requirements**:
- Non-root user required
- No privilege escalation
- Drop all capabilities
- No host namespaces (network, PID, IPC)
- No host paths
- seccompProfile: RuntimeDefault or Localhost
- AppArmor or SELinux
- Limited volume types

**Alternative Solutions**:

**OPA Gatekeeper**:
- Policy-as-code using Rego language
- Highly flexible and customizable
- Can enforce any policy
- Template-based constraints
- Audit and enforcement modes

**Kyverno**:
- Kubernetes-native policy engine
- Policies written in YAML
- Validation, mutation, generation
- Report mode for compliance
- Easier than OPA for Kubernetes-specific policies

#### Network Policies (Security Aspect)
**Purpose**: Implement microsegmentation and zero-trust networking.

**Security Use Cases**:
- **Namespace Isolation**: Prevent cross-namespace traffic
- **Database Protection**: Restrict database access to specific services
- **External Access Control**: Control egress to external services
- **Compliance**: Meet regulatory requirements (PCI-DSS, HIPAA)
- **Lateral Movement Prevention**: Limit attacker movement

**Defense in Depth**:
- Network policies are one layer
- Combine with RBAC, pod security, secrets encryption
- Monitor denied connections
- Regular policy reviews

**Advanced Policies**:
- **Global Policies** (Calico): Cluster-wide rules
- **Layer 7 Policies** (Cilium): HTTP method, path filtering
- **DNS Policies**: Control DNS queries
- **Service-based Policies**: Policy based on service selection

#### Secrets Management
**Advanced Approaches**:

**Encryption at Rest Configuration**:
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key1
          secret: <base64-encoded-32-byte-key>
    - identity: {}
```

**KMS Provider**:
- Integrates with cloud KMS (AWS KMS, GCP KMS, Azure Key Vault)
- Envelope encryption
- Key rotation support
- Audit logging
- Higher security assurance

**Secrets Scanning**:
- Scan images for secrets
- Prevent secret commits to Git
- Tools: GitGuardian, TruffleHog, git-secrets
- CI/CD integration

**Security Best Practices Summary**:

**Authentication**:
- Use external identity providers (OIDC, LDAP)
- Implement MFA for human access
- Certificate-based authentication for services
- Regular credential rotation

**Authorization**:
- RBAC for all access
- Least privilege principle
- Regular permission audits
- Separate admin and user roles

**Network Security**:
- Network policies enabled
- Default deny policies
- Encrypt traffic (service mesh, TLS)
- Limit external access

**Pod Security**:
- Enforce pod security standards
- Use policy engines (OPA, Kyverno)
- Non-root containers
- Read-only root filesystems
- Drop unnecessary capabilities
- Seccomp and AppArmor profiles

**Data Security**:
- Encrypt secrets at rest
- External secret management
- Backup encryption
- Data access auditing

**Image Security**:
- Scan images for vulnerabilities
- Use minimal base images
- Sign and verify images
- Private registries with authentication
- Admission control for allowed registries

**Audit and Monitoring**:
- Enable audit logging
- Monitor suspicious activity
- Alert on policy violations
- Regular security reviews
- Compliance scanning

**Supply Chain Security**:
- Verify image provenance
- Software Bill of Materials (SBOM)
- Signed images (cosign)
- Admission webhooks for verification

### Scaling & Resource Management

Kubernetes provides sophisticated mechanisms for automatic scaling and efficient resource utilization across the cluster.

#### Horizontal Pod Autoscaler (HPA)
**Definition**: Automatically scales the number of pod replicas based on observed metrics.

**How It Works**:
1. Metrics Server collects resource metrics from kubelets
2. HPA controller queries metrics every 15 seconds (default)
3. Calculates desired replica count based on current metrics
4. Compares desired count with current count
5. Updates Deployment/ReplicaSet/StatefulSet scale
6. Respects cooldown periods to prevent flapping

**Scaling Algorithm**:
```
desiredReplicas = ceil[currentReplicas * (currentMetricValue / targetMetricValue)]
```

**Metric Types**:

**Resource Metrics** (CPU, Memory):
- Based on resource requests (must be set)
- CPU: Target utilization percentage
- Memory: Average value or utilization
- Most common use case

**Custom Metrics**:
- Application-specific metrics (requests per second, queue length)
- Requires metrics adapter (Prometheus Adapter, Datadog, etc.)
- More accurate for application behavior
- Examples: HTTP request rate, message queue depth, cache hit ratio

**External Metrics**:
- Metrics from external systems
- Cloud provider metrics (AWS CloudWatch, GCP Monitoring)
- Third-party services
- Examples: SQS queue length, pub/sub subscription backlog

**Multiple Metrics**:
- HPA supports multiple metrics simultaneously
- Calculates desired replicas for each metric
- Uses highest replica count (most conservative)
- Ensures all metrics are satisfied

**Scaling Behavior**:

**Scale Up**:
- Fast by default (scale immediately when needed)
- Configurable stabilization window
- Can set maximum scale-up rate

**Scale Down**:
- Conservative (default 5-minute stabilization)
- Prevents flapping
- Configurable scale-down rate
- Can set minimum scale-down interval

**Cooldown Periods**:
- **Scale Up**: Default 3 minutes
- **Scale Down**: Default 5 minutes
- Prevents rapid oscillation
- Adjustable per use case

**Advanced Configuration**:

**Scaling Policies** (v2 API):
```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300
    policies:
    - type: Percent
      value: 50
      periodSeconds: 60
    - type: Pods
      value: 2
      periodSeconds: 60
  scaleUp:
    stabilizationWindowSeconds: 0
    policies:
    - type: Percent
      value: 100
      periodSeconds: 30
    - type: Pods
      value: 4
      periodSeconds: 30
```

**Limitations and Considerations**:
- Cannot scale to zero (use KEDA for event-driven scaling)
- Minimum replica count: 1 (or specified minReplicas)
- Requires metrics server or custom metrics adapter
- Metrics lag can cause delayed scaling
- Cold start time affects scaling effectiveness
- Resource requests must be defined

**Best Practices**:
- Set appropriate resource requests
- Choose relevant metrics for your application
- Test scaling behavior under load
- Set reasonable min/max replicas
- Monitor HPA events and metrics
- Use multiple metrics for complex apps
- Consider startup time in scaling decisions
- Implement graceful shutdown
- Use PodDisruptionBudgets with HPA
- Account for downstream dependencies

**Use Cases**:
- Web applications with variable traffic
- API services
- Background job processors
- Microservices with fluctuating load
- Batch processing systems
- Real-time data processing

#### Vertical Pod Autoscaler (VPA)
**Definition**: Automatically adjusts CPU and memory requests/limits for containers based on historical usage.

**Purpose**:
- Right-size resource requests and limits
- Eliminate resource waste
- Prevent resource starvation
- Optimize cluster utilization
- Reduce manual tuning effort

**How It Works**:
1. VPA Recommender analyzes historical resource usage
2. Generates recommendations for requests and limits
3. VPA Updater evicts pods that need updates
4. VPA Admission Controller injects updated values into new pods
5. Process repeats continuously

**Components**:

**Recommender**:
- Analyzes metrics history (default: 8 days)
- Calculates optimal requests and limits
- Considers safety margins
- Provides recommendations even without updates

**Updater**:
- Evicts pods that need resource updates
- Respects PodDisruptionBudgets
- Can be disabled for recommendation-only mode
- Causes pod recreation (downtime)

**Admission Controller**:
- Webhook that mutates pod specs
- Injects recommended resources
- Works with Updater for seamless updates
- Handles initial pod creation

**Update Modes**:

**Off**:
- VPA disabled (no recommendations or updates)
- Can still query for recommendations
- Use for testing VPA installation

**Initial**:
- Only applies recommendations to new pods
- Doesn't evict running pods
- Good for stateful apps
- No disruption to existing workloads

**Auto**:
- Applies recommendations to running pods
- Evicts pods to apply new resources
- Fully automated
- Causes pod recreation

**Recreate**:
- Same as Auto (legacy compatibility)
- Evicts and recreates pods
- Consider downtime implications

**Resource Policies**:

**Container Policies**:
- Per-container configuration
- Minimum and maximum allowed resources
- Control which resources VPA can manage
- Mode overrides per container

**Update Policy**:
```yaml
updatePolicy:
  updateMode: "Auto"
resourcePolicy:
  containerPolicies:
  - containerName: app
    minAllowed:
      cpu: 100m
      memory: 128Mi
    maxAllowed:
      cpu: 2
      memory: 2Gi
    controlledResources: ["cpu", "memory"]
    mode: "Auto"
```

**Compatibility with HPA**:
- VPA and HPA conflict on CPU/memory metrics
- **Don't use together** on same metric
- Safe: HPA on custom metrics + VPA on resources
- Alternative: VPA recommendations only (mode: Off)
- Future: Multidimensional Pod Autoscaler

**Limitations**:
- Requires pod recreation (downtime)
- Cannot work with HPA on same resource
- Historical data required for good recommendations
- Initial recommendations may be inaccurate
- Not suitable for single-replica apps without downtime tolerance

**Best Practices**:
- Start with recommendation mode (Off)
- Review recommendations before enabling Auto mode
- Use Initial mode for stateful apps
- Set appropriate min/max limits
- Monitor actual usage vs recommendations
- Consider using with PodDisruptionBudgets
- Test in non-production first
- Combine with cluster autoscaler
- Document expected behavior
- Plan for pod disruptions

**Use Cases**:
- Long-running applications with changing resource needs
- Over-provisioned workloads
- Under-provisioned workloads causing OOM
- Applications with seasonal patterns
- Microservices with unpredictable resource usage
- Cost optimization initiatives
- Resource capacity planning

#### Cluster Autoscaler
**Definition**: Automatically adjusts the number of nodes in a cluster based on pending pods and resource utilization.

**Purpose**:
- Scale cluster capacity up when pods can't be scheduled
- Scale cluster capacity down when nodes are underutilized
- Optimize infrastructure costs
- Maintain application availability
- Respond to demand changes

**How It Works**:

**Scale Up**:
1. Pods remain in Pending state (unschedulable)
2. Cluster Autoscaler detects pending pods
3. Simulates scheduling pods on new nodes
4. Selects appropriate node group/type
5. Requests new nodes from cloud provider
6. Waits for nodes to become Ready
7. Scheduler assigns pending pods to new nodes

**Scale Down**:
1. Monitors node utilization (default: CPU and memory)
2. Identifies underutilized nodes (< 50% utilization by default)
3. Checks if pods can be rescheduled to other nodes
4. Respects PodDisruptionBudgets
5. Waits for scale-down delay (default: 10 minutes)
6. Cordons and drains node
7. Deletes node from cloud provider

**Node Groups**:
- Group of nodes with same characteristics
- Cloud provider concept (AWS ASG, GCP MIG, Azure VMSS)
- Different node types for different workloads
- Each group can have min/max size
- Cluster Autoscaler picks best node group for pending pods

**Scale-Up Strategy**:
- **least-waste**: Minimize unused resources (default)
- **most-pods**: Maximize pod count on new node
- **priority**: Use expander priorities
- **price**: Choose cheapest option (experimental)
- **random**: Random selection

**Nodes That Won't Be Scaled Down**:
- Nodes running kube-system pods (without PDB)
- Nodes with local storage pods
- Nodes with pods that have local storage volumes
- Nodes with pods that can't be evicted (no replica, no controller)
- Nodes with pods having restrictive PodDisruptionBudget
- Nodes with specific annotations (manual control)

**Configuration Options**:

**Scale-Up Parameters**:
- `--max-nodes-total`: Maximum total nodes
- `--scale-up-from-zero`: Enable scaling from 0 nodes
- `--max-node-provision-time`: Timeout for node provisioning (default: 15 min)
- `--scale-up-cool-down-period`: Wait after scale-up (default: 10 min)

**Scale-Down Parameters**:
- `--scale-down-enabled`: Enable scale-down (default: true)
- `--scale-down-delay-after-add`: Wait after node added (default: 10 min)
- `--scale-down-unneeded-time`: Time before considering scale-down (default: 10 min)
- `--scale-down-utilization-threshold`: Threshold for underutilization (default: 0.5)

**Cloud Provider Integration**:

**AWS**:
- Auto Scaling Groups (ASG)
- Mixed instance types
- Spot instances support
- Multiple ASGs per cluster

**GCP**:
- Managed Instance Groups (MIG)
- Multiple node pools
- Preemptible instances
- Regional clusters

**Azure**:
- Virtual Machine Scale Sets (VMSS)
- Multiple node pools
- Spot VMs
- Availability zones

**Pod Annotations for Control**:
```yaml
cluster-autoscaler.kubernetes.io/safe-to-evict: "false"  # Prevent eviction
cluster-autoscaler.kubernetes.io/safe-to-evict: "true"   # Allow eviction
```

**Node Annotations**:
```yaml
cluster-autoscaler.kubernetes.io/scale-down-disabled: "true"  # Prevent scale-down
```

**Best Practices**:
- Set PodDisruptionBudgets for critical applications
- Use node affinity to guide pod placement
- Set appropriate resource requests (required for scaling decisions)
- Use multiple node groups for different workload types
- Monitor cluster autoscaler events and metrics
- Configure reasonable min/max node counts
- Test scale-up and scale-down behavior
- Use priority classes for critical pods
- Implement graceful shutdown handlers
- Consider startup time in scale-up delay
- Use taints/tolerations for specialized nodes
- Plan for cloud provider API rate limits

**Challenges**:
- Cloud provider delays (minutes to provision nodes)
- Scale-up may be slower than HPA
- Cost optimization vs availability trade-off
- Complexity with multiple node groups
- Stateful applications requiring careful handling
- IP address exhaustion in VPC
- Cloud provider quotas

**Integration with HPA and VPA**:
- **HPA**: Scales pods → may trigger cluster autoscaler
- **VPA**: Adjusts resources → may trigger cluster autoscaler
- **Cluster Autoscaler**: Adds/removes nodes
- All three work together for complete autoscaling solution

#### Resource Quotas
**Definition**: Policy-based resource consumption limits at the namespace level.

**Purpose**:
- Prevent resource hogging
- Fair resource distribution
- Multi-tenancy isolation
- Cost control
- Capacity planning
- Prevent "noisy neighbor" problems

**Resource Types**:

**Compute Resources**:
- `requests.cpu`: Total CPU requests
- `requests.memory`: Total memory requests
- `limits.cpu`: Total CPU limits
- `limits.memory`: Total memory limits
- `requests.storage`: Total storage requests
- `persistentvolumeclaims`: Number of PVCs

**Object Count**:
- `pods`: Maximum number of pods
- `services`: Maximum number of services
- `secrets`: Maximum number of secrets
- `configmaps`: Maximum number of ConfigMaps
- `replicationcontrollers`: Maximum RCs
- `deployments.apps`: Maximum Deployments
- `statefulsets.apps`: Maximum StatefulSets
- `jobs.batch`: Maximum Jobs
- `cronjobs.batch`: Maximum CronJobs

**Storage Classes**:
- Per-StorageClass quotas
- Format: `<storage-class-name>.storageclass.storage.k8s.io/requests.storage`
- Example: `fast-ssd.storageclass.storage.k8s.io/requests.storage: 100Gi`

**Scope Selectors**:
- **BestEffort**: Pods with no resource requests/limits
- **NotBestEffort**: Pods with requests or limits
- **Terminating**: Pods with `activeDeadlineSeconds`
- **NotTerminating**: Pods without active deadline
- **PriorityClass**: Pods with specific priority

**Enforcement**:
- Admission control at creation time
- Prevents resource creation if quota exceeded
- Updates also checked against quota
- Cross-namespace quotas not supported

**Priority Class Integration**:
```yaml
scopeSelector:
  matchExpressions:
  - operator: In
    scopeName: PriorityClass
    values: ["high"]
```

**Best Practices**:
- Set quotas for all namespaces (except kube-system)
- Differentiate dev/staging/prod quotas
- Set both requests and limits quotas
- Monitor quota usage
- Alert on quota approaching limits
- Document quota rationale
- Review quotas periodically
- Use meaningful namespace names
- Combine with LimitRanges

#### Limit Ranges
**Definition**: Enforce minimum, maximum, and default resource constraints for pods and containers within a namespace.

**Purpose**:
- Prevent single pod from consuming too much
- Set default requests/limits
- Enforce resource best practices
- Prevent resource omission
- Complement Resource Quotas

**Constraints**:

**Container-level**:
- Min/max CPU
- Min/max memory
- Min/max ephemeral storage
- Default requests
- Default limits
- Default request/limit ratio

**Pod-level**:
- Total resources across all containers
- Useful for preventing many small containers

**PVC-level**:
- Min/max storage size
- Prevents very large volumes

**Example Configuration**:
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
spec:
  limits:
  - max:
      cpu: "4"
      memory: "8Gi"
    min:
      cpu: "100m"
      memory: "128Mi"
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "200m"
      memory: "256Mi"
    maxLimitRequestRatio:
      cpu: "10"
      memory: "2"
    type: Container
  - max:
      cpu: "8"
      memory: "16Gi"
    type: Pod
  - max:
      storage: "100Gi"
    min:
      storage: "1Gi"
    type: PersistentVolumeClaim
```

**Default Behavior**:
- If container has no requests: LimitRange default request applied
- If container has no limits: LimitRange default limit applied
- If no LimitRange defaults: No resources set (dangerous)

**Validation**:
- Checked at pod/PVC creation time
- Updates also validated
- Prevents non-compliant resources

**Best Practices**:
- Define defaults for every namespace
- Set reasonable minimums to prevent resource starvation
- Set maximums to prevent resource hogging
- Use with Resource Quotas for complete control
- Document reasoning for limits
- Different limits for different environments
- Review and adjust based on actual usage
- Educate developers on resource management

**Combined Resource Management Strategy**:
1. **LimitRange**: Set defaults and per-resource limits
2. **ResourceQuota**: Set namespace-level aggregates
3. **HPA**: Auto-scale replicas based on load
4. **VPA**: Right-size individual pods
5. **Cluster Autoscaler**: Scale cluster infrastructure
6. **PodDisruptionBudget**: Ensure availability during disruptions

## High Availability (HA)

High availability in Kubernetes ensures minimal downtime and continuous service availability through redundancy, automated failover, and resilient architecture.

### Control Plane HA

A highly available control plane prevents single points of failure in cluster management.

#### Multi-Master Setup
**Architecture**:
- **Minimum 3 control plane nodes** (odd number for quorum)
- **5 or 7 nodes** for larger production clusters
- **Load balancer** distributes API requests across API servers
- **Separated or co-located** etcd (separate recommended for large clusters)

**Components Distribution**:

**API Server**:
- Active-active configuration
- All instances serve traffic simultaneously
- Load balancer distributes requests (round-robin, least connections)
- Each instance is stateless
- Horizontal scaling for increased capacity

**Controller Manager**:
- Active-passive (leader election)
- Only one instance actively manages controllers
- Others remain in standby
- Automatic failover if leader fails
- Uses lease-based leader election

**Scheduler**:
- Active-passive (leader election)
- One active scheduler makes scheduling decisions
- Others wait for leader failure
- Quick failover (typically seconds)
- Lease-based election mechanism

**Load Balancer Requirements**:
- Health checks on API server (`:6443/healthz`)
- TCP load balancing (layer 4)
- Session affinity not required
- High availability itself (redundant LB)
- Options: HAProxy, NGINX, cloud provider LB (AWS NLB, Azure LB)

**Failure Scenarios**:
- **Single API server failure**: Other API servers continue serving
- **Single controller-manager failure**: Standby promoted to leader
- **Single scheduler failure**: Standby takes over scheduling
- **Load balancer failure**: Requires redundant LB configuration
- **Network partition**: Etcd quorum ensures consistency

#### etcd Clustering
**Purpose**:
- Distributed consensus for cluster state
- High availability for cluster data
- Fault tolerance through replication
- Consistent reads and writes

**Cluster Sizing**:
- **3 nodes**: Tolerates 1 failure, recommended minimum
- **5 nodes**: Tolerates 2 failures, recommended for production
- **7 nodes**: Tolerates 3 failures, for critical large clusters
- **Odd numbers only**: Required for quorum (majority)

**Quorum Requirements**:
- Majority must be available: `(n/2) + 1`
- 3-node cluster: Requires 2 nodes (tolerates 1 failure)
- 5-node cluster: Requires 3 nodes (tolerates 2 failures)
- 7-node cluster: Requires 4 nodes (tolerates 3 failures)
- Loss of quorum: Cluster becomes read-only

**Topology Options**:

**Stacked etcd**:
- etcd runs on same nodes as control plane
- Simpler setup and management
- Fewer servers required
- Coupled failure domain (node failure affects both)
- Suitable for smaller clusters

**External etcd**:
- etcd on dedicated nodes
- Decoupled failure domains
- Better performance isolation
- More complex setup
- Recommended for large production clusters
- Allows independent scaling

**etcd Performance Considerations**:
- **Disk I/O Critical**: Use SSDs (required for production)
- **Latency Sensitive**: Low-latency network required
- **Network Bandwidth**: Replication traffic between nodes
- **CPU**: 2-4 cores typical for moderate clusters
- **Memory**: 8-16GB typical, depends on cluster size
- **Dedicated Hardware**: Recommended for large clusters

**Best Practices**:
- Use fast SSDs for etcd storage
- Monitor etcd latency and disk fsync duration
- Keep etcd cluster close together (low latency)
- Separate etcd traffic (dedicated network interface)
- Regular backups (critical for disaster recovery)
- Automated backup schedules
- Monitor etcd metrics (Prometheus)
- Test restore procedures
- Limit API server request rates
- Keep etcd version current

#### Leader Election
**Mechanism**:
- Uses Kubernetes lease objects (coordination.k8s.io/v1)
- Atomic operations ensure single leader
- Lease duration typically 15 seconds
- Renew interval typically 10 seconds
- Retry period typically 2 seconds

**How It Works**:
1. Component attempts to acquire lease
2. If lease doesn't exist or expired, acquire it
3. Leader continuously renews lease
4. If leader fails, lease expires
5. Other instances detect expiration
6. New leader elected through acquisition race
7. New leader starts active operations

**Failover Time**:
- Detection: Lease duration (default 15s)
- Acquisition: Typically 1-2 seconds
- **Total failover**: ~15-20 seconds typically

**Monitoring**:
- Watch leader election metrics
- Alert on frequent leader changes
- Monitor lease renewal failures
- Check for split-brain scenarios

#### API Server Load Balancing
**Purpose**:
- Distribute client requests across API servers
- Provide single endpoint for cluster access
- Health-check API server availability
- Enable rolling updates of control plane

**Load Balancer Types**:

**Layer 4 (TCP)**:
- Simple pass-through load balancing
- No SSL termination at LB
- Lower latency
- Client certificate auth preserved
- Most common for Kubernetes

**Layer 7 (HTTP/HTTPS)**:
- Can inspect requests
- URL-based routing possible
- SSL termination at LB
- More complex configuration
- May complicate client auth

**Implementation Options**:

**HAProxy**:
```
frontend kubernetes-api
    bind *:6443
    mode tcp
    option tcplog
    default_backend kubernetes-api

backend kubernetes-api
    mode tcp
    balance roundrobin
    option tcp-check
    server master1 10.0.0.1:6443 check
    server master2 10.0.0.2:6443 check
    server master3 10.0.0.3:6443 check
```

**NGINX**:
```
stream {
    upstream k8s_api {
        least_conn;
        server 10.0.0.1:6443 max_fails=3 fail_timeout=30s;
        server 10.0.0.2:6443 max_fails=3 fail_timeout=30s;
        server 10.0.0.3:6443 max_fails=3 fail_timeout=30s;
    }
    server {
        listen 6443;
        proxy_pass k8s_api;
        proxy_timeout 10s;
        proxy_connect_timeout 1s;
    }
}
```

**Cloud Provider Load Balancers**:
- AWS Network Load Balancer (NLB)
- Azure Load Balancer
- GCP Network Load Balancer
- Fully managed, highly available
- Automatic health checks
- Cross-zone load balancing

**Health Check Configuration**:
- Endpoint: `/healthz` or `/readyz`
- Protocol: HTTPS
- Expected response: 200 OK
- Interval: 5-10 seconds
- Timeout: 2-5 seconds
- Unhealthy threshold: 2-3 failures

### Application HA

Ensuring application-level high availability through redundancy and automated recovery.

#### Multiple Replicas
**Strategy**:
- Run multiple pod instances
- Distribute across nodes and zones
- Ensure load balancing
- Enable rolling updates
- Implement circuit breakers

**Replica Count Considerations**:
- **Minimum 2**: Basic redundancy, single failure tolerance
- **3+**: Better availability, rolling update safety
- **Odd numbers**: Better for consensus-based apps
- **N+1 or N+2**: Capacity for maintenance and failures

**Deployment Strategy**:
```yaml
replicas: 3
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # Extra pods during update
    maxUnavailable: 1  # Pods unavailable during update
```

#### Pod Anti-Affinity
**Purpose**:
- Spread pods across different nodes
- Spread across availability zones
- Avoid single point of failure
- Improve fault tolerance

**Types**:

**requiredDuringSchedulingIgnoredDuringExecution**:
- Hard requirement
- Pod not scheduled if rule can't be satisfied
- Use for critical applications
- May leave pods pending if no suitable node

**preferredDuringSchedulingIgnoredDuringExecution**:
- Soft requirement
- Best-effort spreading
- Pod scheduled even if rule not satisfied
- More flexible, prevents pending pods

**Examples**:

**Node Anti-Affinity**:
```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values:
          - myapp
      topologyKey: kubernetes.io/hostname  # Different nodes
```

**Zone Anti-Affinity**:
```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - myapp
        topologyKey: topology.kubernetes.io/zone  # Different zones
```

**Best Practices**:
- Use preferred anti-affinity for flexibility
- Combine node and zone anti-affinity
- Ensure enough nodes/zones available
- Test behavior during node failures
- Monitor unschedulable pods

#### Readiness and Liveness Probes
**Purpose**:
- Automated health checking
- Automatic recovery from failures
- Safe rolling updates
- Load balancer integration

**Liveness Probe**:
- **Purpose**: Detect if container is alive
- **Action**: Restart container if fails
- **Use Case**: Deadlocks, infinite loops, crashes
- **Failure Impact**: Container restart (may impact service)

**Readiness Probe**:
- **Purpose**: Detect if container ready to serve traffic
- **Action**: Remove from service endpoints if fails
- **Use Case**: Startup time, temporary unavailability, dependencies
- **Failure Impact**: Traffic stopped, no restart

**Startup Probe**:
- **Purpose**: Detect when slow-starting container ready
- **Action**: Disable liveness/readiness until succeeds
- **Use Case**: Legacy apps with long startup
- **Failure Impact**: Container restarted if exceeds time

**Probe Mechanisms**:

**HTTP GET**:
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
    - name: Custom-Header
      value: Value
  initialDelaySeconds: 15
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
  successThreshold: 1
```

**TCP Socket**:
```yaml
readinessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

**Exec Command**:
```yaml
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

**gRPC** (Kubernetes 1.24+):
```yaml
livenessProbe:
  grpc:
    port: 9090
  initialDelaySeconds: 10
```

**Configuration Parameters**:
- **initialDelaySeconds**: Wait before first probe (allow startup)
- **periodSeconds**: How often to probe
- **timeoutSeconds**: Probe timeout
- **successThreshold**: Consecutive successes needed (default 1)
- **failureThreshold**: Consecutive failures before action (default 3)

**Best Practices**:
- Always define readiness probe for user-facing apps
- Use liveness probe for long-running apps
- Set appropriate timeouts (don't be too aggressive)
- Probe endpoints should be lightweight
- Don't probe external dependencies in liveness
- Include dependency checks in readiness
- Test probe behavior under load
- Monitor probe failures
- Consider startup time in initial delay

#### PodDisruptionBudgets (PDB)
**Definition**: Limit the number of pods that can be voluntarily disrupted simultaneously.

**Purpose**:
- Maintain application availability during voluntary disruptions
- Safe cluster maintenance (node drains, upgrades)
- Safe application updates
- Protection against cascade failures

**Voluntary vs Involuntary Disruptions**:

**Voluntary** (PDB applies):
- Node drain for maintenance
- Cluster autoscaler scale-down
- Deployment rolling updates
- Manual pod deletion
- Node pool updates

**Involuntary** (PDB doesn't apply):
- Hardware failure
- Node crash or kernel panic
- Out of memory kill
- Network partition
- Power outage

**Configuration Options**:

**minAvailable**:
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2  # or "50%"
  selector:
    matchLabels:
      app: myapp
```

**maxUnavailable**:
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  maxUnavailable: 1  # or "25%"
  selector:
    matchLabels:
      app: myapp
```

**Choosing Between min/max**:
- **minAvailable**: Guarantee minimum running (e.g., 2 for quorum)
- **maxUnavailable**: Limit disruption rate (e.g., 1 at a time)
- Use percentages for dynamic replica counts
- Cannot specify both simultaneously

**Unhealthy Pod Eviction Policy** (1.26+):
```yaml
spec:
  unhealthyPodEvictionPolicy: IfHealthyBudget  # or AlwaysAllow
  minAvailable: 2
```

**Best Practices**:
- Set PDB for all critical applications
- Use percentages for auto-scaled apps
- Test with kubectl drain
- Monitor PDB status
- Don't set too restrictive (blocks maintenance)
- Coordinate with HPA settings
- Document PDB rationale

#### Rolling Updates
**Purpose**:
- Zero-downtime deployments
- Gradual rollout of changes
- Automatic rollback on failures
- Controlled update pace

**Update Strategy Configuration**:
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%         # Extra pods during update
    maxUnavailable: 25%   # Pods unavailable during update
```

**Update Process**:
1. Create new ReplicaSet with updated spec
2. Scale up new ReplicaSet gradually
3. Wait for new pods to become Ready
4. Scale down old ReplicaSet
5. Repeat until all pods updated
6. Keep old ReplicaSets for rollback

**Progressive Delivery Patterns**:

**Canary Deployment**:
- Deploy new version to small subset
- Monitor metrics and errors
- Gradually increase traffic
- Rollback if issues detected
- Tools: Flagger, Argo Rollouts

**Blue-Green Deployment**:
- Deploy complete new version alongside old
- Switch traffic atomically
- Quick rollback by switching back
- Requires double resources temporarily

**A/B Testing**:
- Route traffic based on user attributes
- Compare metrics between versions
- Requires service mesh or ingress rules
- Gradual migration based on results

**Rollback Procedures**:
```bash
# View rollout history
kubectl rollout history deployment/myapp

# Rollback to previous version
kubectl rollout undo deployment/myapp

# Rollback to specific revision
kubectl rollout undo deployment/myapp --to-revision=2

# Pause/resume rollout
kubectl rollout pause deployment/myapp
kubectl rollout resume deployment/myapp
```

### Infrastructure HA

Ensuring the underlying infrastructure supports high availability requirements.

#### Multi-Zone Deployment
**Purpose**:
- Protect against datacenter failures
- Comply with disaster recovery requirements
- Improve latency through geographic distribution
- Meet regulatory requirements

**Node Distribution**:
- Spread nodes across 3+ availability zones
- Use zone-aware topology labels
- Configure zone-aware scheduling
- Balance node count per zone

**Zone Labels**:
- `topology.kubernetes.io/zone`: Availability zone
- `topology.kubernetes.io/region`: Cloud region
- Used by schedulers and CSI drivers

**Topology Spread Constraints**:
```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule  # or ScheduleAnyway
  labelSelector:
    matchLabels:
      app: myapp
```

**Storage Considerations**:
- Use regional storage when possible
- WaitForFirstConsumer volume binding
- Replicated storage across zones
- Backup to different region
- Consider cross-zone latency

**Network Considerations**:
- Cross-zone traffic costs
- Increased latency (typically 1-2ms)
- Bandwidth limits
- NAT gateway per zone

#### Node Redundancy
**Capacity Planning**:
- **N+1**: Extra capacity for one node failure
- **N+2**: Extra capacity for two node failures
- Consider maintenance windows
- Account for autoscaling lag
- Plan for peak load + failures

**Node Pools**:
- Separate pools for different workload types
- System nodes (control plane components)
- Application nodes (user workloads)
- Storage nodes (local storage)
- GPU/specialized hardware nodes

**Taints and Tolerations**:
```yaml
# System nodes
taints:
- key: node-role.kubernetes.io/system
  value: "true"
  effect: NoSchedule

# Tolerate in system pods
tolerations:
- key: node-role.kubernetes.io/system
  operator: Equal
  value: "true"
  effect: NoSchedule
```

#### Backup and Disaster Recovery
**Critical Components to Backup**:

**etcd**:
- Most critical component
- Contains entire cluster state
- Backup frequency: Hourly or more
- Test restore regularly
- Store backups off-cluster

**Backup Methods**:
```bash
# etcdctl snapshot
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify snapshot
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db
```

**Application Data**:
- Persistent volume snapshots
- Database backups
- Use CSI snapshot features
- Automated backup schedules
- Off-site backup storage

**Backup Tools**:

**Velero** (formerly Heptio Ark):
- Kubernetes-native backup solution
- Backup namespaces, resources, persistent volumes
- Scheduled backups
- Disaster recovery
- Cluster migration
- Cloud provider integration

**Backup Strategy**:
- **RPO** (Recovery Point Objective): How much data loss acceptable
- **RTO** (Recovery Time Objective): How quickly recover
- Regular backup testing
- Documented restore procedures
- Off-site backup storage
- Backup encryption
- Retention policies

**Disaster Recovery Procedures**:
1. Document cluster configuration
2. Store backup scripts and configs in Git
3. Maintain disaster recovery runbooks
4. Regular DR drills
5. Test restore in separate environment
6. Validate application functionality after restore
7. Update DR documentation regularly

**High Availability Checklist**:
- [ ] Multi-master control plane (3+ nodes)
- [ ] etcd cluster (3 or 5 nodes)
- [ ] Load balancer for API server
- [ ] Multi-zone node distribution
- [ ] Application replicas (3+)
- [ ] Pod anti-affinity configured
- [ ] Readiness and liveness probes
- [ ] PodDisruptionBudgets defined
- [ ] Resource requests and limits set
- [ ] Horizontal Pod Autoscaler
- [ ] Cluster Autoscaler
- [ ] Regular etcd backups
- [ ] Disaster recovery plan documented
- [ ] DR drills performed regularly
- [ ] Monitoring and alerting configured
- [ ] On-call rotation established

## Infrastructure as Code (IaC)

Infrastructure as Code treats infrastructure configuration as software, enabling version control, testing, and automation of Kubernetes deployments.

### Declarative Configuration

The foundation of Kubernetes IaC is declarative configuration, where you specify the desired state rather than the steps to achieve it.

#### YAML/JSON Manifests
**Purpose**:
- Define desired state of resources
- Version-controlled configuration
- Self-documenting infrastructure
- Reproducible deployments
- Environment parity

**Manifest Structure**:
```yaml
apiVersion: apps/v1           # API version
kind: Deployment              # Resource type
metadata:                     # Resource metadata
  name: myapp
  namespace: production
  labels:
    app: myapp
    version: "1.0"
  annotations:
    description: "My application deployment"
spec:                         # Desired state specification
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:                   # Pod template
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0.0
        ports:
        - containerPort: 8080
```

**Organization Strategies**:

**By Resource Type**:
```
k8s/
├── namespaces/
│   └── production.yaml
├── deployments/
│   ├── api.yaml
│   └── web.yaml
├── services/
│   ├── api-service.yaml
│   └── web-service.yaml
└── ingress/
    └── main-ingress.yaml
```

**By Application**:
```
k8s/
├── api/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── hpa.yaml
│   └── pdb.yaml
├── web/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
└── database/
    ├── statefulset.yaml
    ├── service.yaml
    └── pvc.yaml
```

**By Environment**:
```
k8s/
├── base/                     # Common configuration
│   ├── deployment.yaml
│   └── service.yaml
├── overlays/
│   ├── development/
│   │   └── kustomization.yaml
│   ├── staging/
│   │   └── kustomization.yaml
│   └── production/
│       └── kustomization.yaml
```

#### kubectl apply
**Idempotent Operations**:
- Safe to run multiple times
- Creates resources if they don't exist
- Updates resources if they exist
- Three-way merge (last applied, current, new)
- Preserves manual changes when possible

**Apply Strategies**:

**Single File**:
```bash
kubectl apply -f deployment.yaml
```

**Directory**:
```bash
kubectl apply -f ./k8s/
kubectl apply -R -f ./k8s/  # Recursive
```

**Multiple Files**:
```bash
kubectl apply -f namespace.yaml -f deployment.yaml -f service.yaml
```

**From URL**:
```bash
kubectl apply -f https://example.com/manifests/app.yaml
```

**Server-Side Apply** (1.22+):
```bash
kubectl apply --server-side -f deployment.yaml
```

**Dry Run**:
```bash
# Client-side validation
kubectl apply -f deployment.yaml --dry-run=client

# Server-side validation
kubectl apply -f deployment.yaml --dry-run=server

# Show diff before applying
kubectl diff -f deployment.yaml
```

**Best Practices**:
- Always use `apply` instead of `create` for IaC
- Include `--prune` for garbage collection
- Use `--record` to track change-cause (deprecated, use annotations)
- Validate with `--dry-run` and `diff`
- Use `-o yaml` to see computed configuration
- Store last-applied-configuration annotation

#### GitOps
**Definition**: Operational model where Git is the single source of truth for declarative infrastructure and applications.

**Core Principles**:
1. **Declarative**: System state described declaratively
2. **Versioned**: State stored in Git (immutable, versioned)
3. **Pulled**: Changes pulled from Git automatically
4. **Continuous**: System continuously reconciles with Git

**Benefits**:
- **Audit Trail**: Complete change history in Git
- **Disaster Recovery**: Restore from Git
- **Collaboration**: Pull request workflows
- **Security**: Git access controls and encryption
- **Rollback**: Revert Git commits to rollback
- **Consistency**: Same config across environments
- **Automation**: CI/CD integration

**GitOps Workflow**:
```
┌─────────────┐
│  Developer  │
└──────┬──────┘
       │ 1. Push changes
       ▼
┌─────────────┐
│     Git     │◄────────────┐
│ Repository  │             │ 6. Update status
└──────┬──────┘             │
       │ 2. Webhook/Poll    │
       ▼                    │
┌─────────────┐      ┌──────┴──────┐
│   CI/CD     │      │   GitOps    │
│  Pipeline   │      │  Operator   │
└──────┬──────┘      └──────┬──────┘
       │ 3. Build & Test    │
       │ 4. Push image      │ 5. Apply config
       ▼                    ▼
┌─────────────┐      ┌─────────────┐
│   Registry  │      │ Kubernetes  │
│             │      │   Cluster   │
└─────────────┘      └─────────────┘
```

**Repository Structure**:

**Monorepo**:
```
repo/
├── apps/
│   ├── app1/
│   │   ├── base/
│   │   └── overlays/
│   └── app2/
├── infrastructure/
│   ├── namespaces/
│   ├── rbac/
│   └── network-policies/
└── clusters/
    ├── dev/
    ├── staging/
    └── prod/
```

**Multi-Repo**:
- Separate repos per application
- Separate repo for infrastructure
- Separate repos per environment
- Better access control granularity
- More complex to manage

**GitOps Tools**: See GitOps Workflows section below

### IaC Tools Integration

Infrastructure as Code tools that work with Kubernetes for complete infrastructure management.

#### Terraform
**Purpose**: Provision and manage Kubernetes clusters and resources using HCL (HashiCorp Configuration Language).

**Capabilities**:
- **Cluster Provisioning**: Create EKS, AKS, GKE clusters
- **Resource Management**: Create Kubernetes resources
- **State Management**: Track infrastructure state
- **Module System**: Reusable infrastructure components
- **Provider Ecosystem**: 2000+ providers

**Kubernetes Provider**:
```hcl
provider "kubernetes" {
  config_path = "~/.kube/config"
  config_context = "my-cluster"
}

resource "kubernetes_namespace" "production" {
  metadata {
    name = "production"
    labels = {
      environment = "production"
    }
  }
}

resource "kubernetes_deployment" "myapp" {
  metadata {
    name      = "myapp"
    namespace = kubernetes_namespace.production.metadata[0].name
  }
  spec {
    replicas = 3
    selector {
      match_labels = {
        app = "myapp"
      }
    }
    template {
      metadata {
        labels = {
          app = "myapp"
        }
      }
      spec {
        container {
          name  = "myapp"
          image = "myapp:1.0.0"
          port {
            container_port = 8080
          }
        }
      }
    }
  }
}
```

**Best Practices**:
- Use Terraform for cluster infrastructure
- Use native Kubernetes tools for application deployment
- Store state in remote backend (S3, GCS, Azure Storage)
- Use workspaces for environments
- Implement state locking
- Use modules for reusability
- Version Terraform and providers

#### Pulumi
**Purpose**: Modern Infrastructure as Code using general-purpose programming languages.

**Supported Languages**:
- TypeScript/JavaScript
- Python
- Go
- C#/.NET
- Java
- YAML

**Capabilities**:
- Full programming language features
- Type safety and IDE support
- Reusable components (functions, classes)
- Native testing support
- Policy as Code
- Secret management

**Example (TypeScript)**:
```typescript
import * as k8s from "@pulumi/kubernetes";

const namespace = new k8s.core.v1.Namespace("production", {
    metadata: { name: "production" }
});

const deployment = new k8s.apps.v1.Deployment("myapp", {
    metadata: {
        namespace: namespace.metadata.name,
    },
    spec: {
        replicas: 3,
        selector: { matchLabels: { app: "myapp" } },
        template: {
            metadata: { labels: { app: "myapp" } },
            spec: {
                containers: [{
                    name: "myapp",
                    image: "myapp:1.0.0",
                    ports: [{ containerPort: 8080 }]
                }]
            }
        }
    }
});
```

**Advantages**:
- Use familiar programming languages
- Share code with application development
- Rich ecosystem of libraries
- Better testing capabilities
- Type safety prevents errors

#### Ansible
**Purpose**: Automation tool for configuration management, application deployment, and orchestration.

**Kubernetes Integration**:
- `community.kubernetes` collection
- Manage Kubernetes resources
- Cluster provisioning
- Application deployment
- Configuration management

**Example Playbook**:
```yaml
---
- name: Deploy application to Kubernetes
  hosts: localhost
  collections:
    - community.kubernetes
  tasks:
    - name: Create namespace
      k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Namespace
          metadata:
            name: production
    
    - name: Deploy application
      k8s:
        state: present
        definition: "{{ lookup('file', 'deployment.yaml') | from_yaml }}"
    
    - name: Wait for deployment
      k8s_info:
        kind: Deployment
        name: myapp
        namespace: production
      register: deployment
      until: deployment.resources[0].status.availableReplicas == 3
      retries: 30
      delay: 10
```

**Use Cases**:
- Configuration management
- Multi-stage deployments
- Hybrid infrastructure
- Complex orchestration
- Post-deployment tasks

#### Cloud-Native IaC

**AWS CloudFormation**:
```yaml
Resources:
  EKSCluster:
    Type: AWS::EKS::Cluster
    Properties:
      Name: my-cluster
      Version: "1.28"
      RoleArn: !GetAtt EKSClusterRole.Arn
      ResourcesVpcConfig:
        SubnetIds:
          - !Ref PrivateSubnet1
          - !Ref PrivateSubnet2
```

**Azure Resource Manager (ARM)**:
```json
{
  "type": "Microsoft.ContainerService/managedClusters",
  "apiVersion": "2023-01-01",
  "name": "my-aks-cluster",
  "location": "eastus",
  "properties": {
    "kubernetesVersion": "1.28.0",
    "dnsPrefix": "myaks",
    "agentPoolProfiles": [...]
  }
}
```

**Azure Bicep**:
```bicep
resource aksCluster 'Microsoft.ContainerService/managedClusters@2023-01-01' = {
  name: 'my-aks-cluster'
  location: 'eastus'
  properties: {
    kubernetesVersion: '1.28.0'
    dnsPrefix: 'myaks'
    agentPoolProfiles: [...]
  }
}
```

### Cluster Provisioning Tools

Tools specifically designed for Kubernetes cluster creation and management.

#### kubeadm
**Purpose**: Official tool to bootstrap production-ready Kubernetes clusters.

**Features**:
- Best practices configuration
- Certificate management
- Control plane setup
- Worker node joining
- Cluster upgrades

**Usage**:
```bash
# Initialize control plane
kubeadm init --pod-network-cidr=10.244.0.0/16

# Join worker node
kubeadm join <control-plane>:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>

# Upgrade cluster
kubeadm upgrade plan
kubeadm upgrade apply v1.28.0
```

**Use Cases**:
- Self-managed clusters
- On-premises deployments
- Custom configurations
- Learning and development

#### kops (Kubernetes Operations)
**Purpose**: Production-grade Kubernetes cluster management, primarily for AWS.

**Features**:
- Automated cluster provisioning
- High availability setup
- Rolling updates
- Cluster templates
- Add-ons management

**Workflow**:
```bash
# Create cluster configuration
kops create cluster \
  --name=mycluster.k8s.local \
  --state=s3://my-kops-state \
  --zones=us-east-1a,us-east-1b,us-east-1c \
  --node-count=3 \
  --node-size=t3.medium \
  --master-size=t3.medium \
  --master-zones=us-east-1a,us-east-1b,us-east-1c

# Review and edit
kops edit cluster mycluster.k8s.local

# Apply configuration
kops update cluster mycluster.k8s.local --yes

# Rolling update
kops rolling-update cluster mycluster.k8s.local --yes
```

**Use Cases**:
- AWS production clusters
- Multi-master HA setups
- Automated cluster lifecycle
- Infrastructure as Code

#### Rancher
**Purpose**: Complete container management platform with multi-cluster Kubernetes management.

**Features**:
- Multi-cluster dashboard
- Cluster provisioning (RKE, hosted)
- RBAC management
- Monitoring and logging
- Application catalog
- GitOps support

**Capabilities**:
- Import existing clusters
- Provision new clusters
- Centralized authentication
- Policy management
- Backup and disaster recovery

**Use Cases**:
- Multi-cluster management
- Hybrid/multi-cloud
- Team collaboration
- Centralized governance

#### Managed Kubernetes Services

**EKS (Elastic Kubernetes Service)**:
- AWS-managed control plane
- Automatic upgrades
- Native AWS integration
- Fargate support

**AKS (Azure Kubernetes Service)**:
- Azure-managed control plane
- Integrated monitoring
- Virtual nodes (serverless)
- Azure Active Directory integration

**GKE (Google Kubernetes Engine)**:
- Google-managed control plane
- Auto-upgrade and repair
- Autopilot mode (serverless)
- Tight GCP integration

**Benefits**:
- Managed control plane
- Automatic updates
- Native cloud integration
- Enterprise support
- Simplified operations

### Package Management

Tools for managing Kubernetes application packages and configurations.

#### Helm
**Definition**: Package manager for Kubernetes, using charts to define, install, and upgrade applications.

**Concepts**:

**Chart**:
- Package format for Kubernetes resources
- Contains templates, values, and metadata
- Reusable and versioned
- Can be shared via repositories

**Release**:
- Instance of a chart deployed to cluster
- Has unique name and namespace
- Tracked in cluster
- Can be upgraded or rolled back

**Repository**:
- Collection of charts
- Public (Artifact Hub) or private
- HTTP/HTTPS accessible
- OCI registry support

**Chart Structure**:
```
mychart/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default values
├── charts/             # Dependent charts
├── templates/          # Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl   # Template helpers
│   └── NOTES.txt      # Post-install notes
├── .helmignore        # Ignore patterns
└── README.md
```

**Template Example**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        ports:
        - containerPort: {{ .Values.service.port }}
```

**Commands**:
```bash
# Install chart
helm install myapp ./mychart

# Install with custom values
helm install myapp ./mychart -f custom-values.yaml

# Upgrade release
helm upgrade myapp ./mychart

# Rollback release
helm rollback myapp 1

# Uninstall release
helm uninstall myapp

# List releases
helm list

# Chart repository management
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo nginx
```

**Best Practices**:
- Use semantic versioning for charts
- Document values.yaml thoroughly
- Provide sane defaults
- Use `_helpers.tpl` for reusable templates
- Test charts with `helm lint` and `helm test`
- Sign charts for security
- Use NOTES.txt for user guidance
- Pin dependency versions

#### Kustomize
**Definition**: Template-free configuration customization using overlay and patching.

**Philosophy**:
- No templates, just pure YAML
- Declarative customization
- Compose and customize without forking
- Native kubectl integration

**Concepts**:

**Base**:
- Common configuration
- Reusable across environments
- No environment-specific values

**Overlay**:
- Environment-specific customizations
- Patches and additions
- References base configuration

**Kustomization File**:
```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml

namePrefix: dev-
nameSuffix: -v1

commonLabels:
  app: myapp
  environment: development

configMapGenerator:
- name: app-config
  literals:
  - APP_ENV=development
  - LOG_LEVEL=debug

secretGenerator:
- name: app-secret
  literals:
  - API_KEY=secret123

images:
- name: myapp
  newTag: 1.2.3

replicas:
- name: myapp
  count: 3
```

**Overlay Structure**:
```
base/
├── deployment.yaml
├── service.yaml
└── kustomization.yaml

overlays/
├── development/
│   ├── kustomization.yaml
│   └── replica-count.yaml
├── staging/
│   ├── kustomization.yaml
│   └── resources-patch.yaml
└── production/
    ├── kustomization.yaml
    ├── replica-count.yaml
    └── hpa.yaml
```

**Patches**:

**Strategic Merge Patch**:
```yaml
# replica-count.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5
```

**JSON Patch**:
```yaml
# kustomization.yaml
patches:
- target:
    kind: Deployment
    name: myapp
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 5
```

**Commands**:
```bash
# Build configuration
kubectl kustomize overlays/production

# Apply directly
kubectl apply -k overlays/production

# Diff before applying
kubectl diff -k overlays/production
```

**Best Practices**:
- Keep base minimal and generic
- Use overlays for environment differences
- Prefer patches over complete file replacement
- Use generators for ConfigMaps/Secrets
- Combine with Helm for complex apps
- Version control all layers

#### Operators
**Definition**: Custom controllers that extend Kubernetes to manage complex, stateful applications.

**Purpose**:
- Automate operational knowledge
- Manage application lifecycle
- Handle complex configurations
- Implement custom business logic
- Self-healing applications

**Operator Pattern**:
1. Define Custom Resource Definition (CRD)
2. Implement controller watching CRD
3. Reconciliation loop maintains desired state
4. Handle create, update, delete operations
5. Implement operational intelligence

**Popular Operators**:
- **Database**: MySQL, PostgreSQL, MongoDB, Redis
- **Messaging**: Kafka, RabbitMQ, NATS
- **Monitoring**: Prometheus, Grafana
- **Storage**: Rook (Ceph), OpenEBS
- **Service Mesh**: Istio, Linkerd
- **CI/CD**: Tekton, ArgoCD

**Example CRD**:
```yaml
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
                type: integer
              version:
                type: string
```

**Operator Frameworks**:
- **Operator SDK**: Build operators easily
- **Kubebuilder**: Kubernetes API extension framework
- **KUDO**: Kubernetes Universal Declarative Operator
- **Metacontroller**: Lightweight custom controller framework

### GitOps Workflows

GitOps tools that implement continuous delivery for Kubernetes.

#### ArgoCD
**Definition**: Declarative, GitOps continuous delivery tool for Kubernetes.

**Features**:
- **Automated Sync**: Automatically applies changes from Git
- **Manual Sync**: Option for manual approval
- **Multi-Cluster**: Manage multiple clusters
- **RBAC**: Fine-grained access control
- **SSO Integration**: OIDC, SAML, LDAP
- **Webhook Support**: GitHub, GitLab, Bitbucket
- **ApplicationSets**: Template applications across clusters
- **Rollback**: Easy revert to previous versions

**Architecture**:
- **API Server**: REST API and UI
- **Repository Server**: Manages Git repositories
- **Application Controller**: Monitors applications and syncs state

**Application Definition**:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myrepo
    targetRevision: main
    path: k8s/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

**Sync Strategies**:
- **Manual**: Require manual approval
- **Automated**: Sync automatically on change
- **Prune**: Delete resources not in Git
- **Self-Heal**: Revert manual changes

**Best Practices**:
- Use ApplicationSets for multi-environment
- Implement progressive delivery with ArgoCD Rollouts
- Use projects for multi-tenancy
- Enable notifications for sync failures
- Use SSO for authentication
- Implement RBAC policies
- Monitor sync status
- Use hooks for pre/post sync actions

#### Flux
**Definition**: GitOps operator for Kubernetes, part of CNCF.

**Components**:
- **Source Controller**: Manages Git/Helm repositories
- **Kustomize Controller**: Applies Kustomize configurations
- **Helm Controller**: Manages Helm releases
- **Notification Controller**: Sends alerts and webhooks
- **Image Automation**: Automates image updates

**Architecture**:
- Pull-based model
- Continuously monitors Git
- Reconciles cluster state
- Pushes status back to Git

**GitRepository**:
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/myorg/myrepo
  ref:
    branch: main
```

**Kustomization**:
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 5m
  path: ./k8s/production
  prune: true
  sourceRef:
    kind: GitRepository
    name: myapp
```

**Image Automation**:
```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageUpdateAutomation
metadata:
  name: myapp
spec:
  interval: 1m
  sourceRef:
    kind: GitRepository
    name: myapp
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: flux@example.com
      messageTemplate: 'Update image to {{range .Updated.Images}}{{println .}}{{end}}'
```

**Features**:
- Multi-tenancy support
- Progressive delivery (Flagger integration)
- Automated image updates
- Slack/Teams notifications
- Webhook receivers
- Helm repository scanning

#### Jenkins X
**Definition**: CI/CD solution for cloud-native applications on Kubernetes.

**Features**:
- Automated CI/CD pipelines
- GitOps promotion
- Preview environments for PRs
- Automatic versioning
- Cloud provider integration
- Tekton-based pipelines

**Pipeline Example**:
```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: myapp-pipeline
spec:
  tasks:
  - name: build
    taskRef:
      name: build-task
  - name: test
    taskRef:
      name: test-task
    runAfter:
    - build
  - name: deploy
    taskRef:
      name: deploy-task
    runAfter:
    - test
```

### Best Practices for IaC

#### Version Control
- **All Configuration in Git**: Manifests, Helm charts, scripts
- **Branching Strategy**: Gitflow, trunk-based, or custom
- **Pull Request Workflow**: Code review for changes
- **Commit Messages**: Descriptive, follow convention
- **Tags and Releases**: Version significant changes
- **Secrets Management**: Never commit secrets (use external tools)

#### Environment Separation
- **Namespace Isolation**: Separate namespaces per environment
- **Cluster Separation**: Different clusters for prod/non-prod
- **Network Policies**: Enforce isolation
- **RBAC**: Separate access per environment
- **Resource Quotas**: Limit resource usage per environment

#### CI/CD Pipelines
- **Automated Testing**: Lint, validate, test manifests
- **Security Scanning**: Scan images, manifests, secrets
- **Progressive Deployment**: Canary, blue-green strategies
- **Approval Gates**: Manual approval for production
- **Rollback Procedures**: Automated rollback on failure
- **Audit Logging**: Track all changes

#### Container Images
- **Immutable Tags**: Use SHA digests, not `latest`
- **Semantic Versioning**: Version images properly
- **Image Scanning**: Security vulnerability scanning
- **Minimal Images**: Use slim/alpine base images
- **Multi-stage Builds**: Optimize image size
- **Image Registry**: Private registry with access control

#### Configuration Management
- **Externalize Configuration**: ConfigMaps and Secrets
- **Environment Variables**: Use for runtime configuration
- **Feature Flags**: Enable/disable features without deployment
- **Config Validation**: Validate before applying
- **Secret Rotation**: Regular rotation of secrets
- **Least Privilege**: Minimal necessary permissions

#### Policy as Code
- **Admission Control**: OPA Gatekeeper, Kyverno
- **Security Policies**: Pod security, network policies
- **Resource Policies**: Quotas, limits
- **Compliance**: Automated compliance checking
- **Policy Testing**: Test policies before enforcement

#### Testing
- **Unit Tests**: Test individual manifests
- **Integration Tests**: Test application deployment
- **End-to-End Tests**: Full workflow testing
- **Chaos Engineering**: Test resilience
- **Load Testing**: Performance validation
- **Tools**: kind, minikube, k3s for local testing

#### Documentation
- **Architecture Diagrams**: Document cluster architecture
- **Runbooks**: Operational procedures
- **Troubleshooting Guides**: Common issues and solutions
- **Change Management**: Document changes and reasons
- **Disaster Recovery**: Recovery procedures
- **Onboarding**: Developer setup guides

#### Monitoring and Observability
- **Metrics**: Prometheus, Grafana
- **Logging**: Fluentd, Loki, Elasticsearch
- **Tracing**: Jaeger, Zipkin
- **Alerting**: Alert on critical issues
- **Dashboards**: Visualize cluster and app health
- **SLOs/SLIs**: Define and track service levels

#### Security Best Practices
- **Least Privilege**: RBAC with minimal permissions
- **Network Policies**: Restrict pod communication
- **Pod Security**: Enforce pod security standards
- **Image Security**: Scan and sign images
- **Secrets Management**: External secret stores
- **Audit Logging**: Track API access
- **Regular Updates**: Keep Kubernetes and apps updated
- **Vulnerability Scanning**: Regular security scans
