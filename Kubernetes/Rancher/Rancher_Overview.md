# Rancher Overview

## Related Guides

- [Rancher CLI Guide](Rancher_CLI.md)

## What is Rancher?

Rancher is a complete container management platform that simplifies the deployment and management of Kubernetes clusters across any infrastructure. Originally developed by Rancher Labs (acquired by SUSE in 2020), Rancher provides a unified interface for managing multiple Kubernetes clusters, whether they're hosted on-premises, in the cloud, or at the edge.

### Key Characteristics

**Multi-Cluster Management**: Rancher excels at managing multiple Kubernetes clusters from a single control plane. Organizations can manage hundreds or thousands of clusters across different cloud providers, data centers, and edge locations through one unified interface.

**Cluster Provisioning**: Rancher simplifies Kubernetes cluster creation through multiple methods:
- **Hosted Kubernetes**: Import and manage existing EKS, AKS, GKE clusters
- **Infrastructure Providers**: Provision clusters on AWS, Azure, GCP, vSphere, DigitalOcean
- **Custom Clusters**: Bring your own nodes (BYOH)
- **RKE/RKE2**: Rancher Kubernetes Engine for production-grade clusters
- **K3s**: Lightweight Kubernetes for edge and IoT

**Centralized Authentication**: Integrate with enterprise identity providers (Active Directory, LDAP, SAML, OIDC, GitHub, Google) for single sign-on across all managed clusters. Rancher acts as an authentication proxy, eliminating the need to manage users separately in each cluster.

**Role-Based Access Control**: Rancher extends Kubernetes RBAC with additional layers:
- **Global Permissions**: Admin, users, restricted users
- **Cluster Roles**: Cluster owner, member, viewer, custom roles
- **Project Roles**: Project owner, member, read-only, custom roles
- Hierarchical permission inheritance simplifies access management

**Application Catalog**: Deploy applications easily through curated Helm charts and Rancher's application catalog. Pre-configured applications include monitoring (Prometheus, Grafana), logging (Elasticsearch, Fluentd), CI/CD tools, databases, and more.

**Monitoring and Alerting**: Built-in monitoring powered by Prometheus and Grafana provides cluster and application visibility. Pre-configured dashboards for cluster metrics, workload performance, and resource utilization. Alert manager integration for proactive issue detection.

**Multi-Tenancy**: Projects provide a layer of abstraction above namespaces, enabling multi-tenant isolation with resource quotas, network policies, and pod security policies. Teams can self-manage their projects within defined boundaries.

**GitOps with Fleet**: Fleet enables GitOps-based continuous delivery at scale, managing deployments across thousands of clusters from Git repositories. Supports cluster groups, staged rollouts, and drift detection.

**Security and Compliance**: CIS benchmark scanning, pod security policy templates, network segmentation, secrets management, and RBAC enforcement help meet security and compliance requirements.

**Backup and Disaster Recovery**: etcd backup and restore, cluster snapshots, and disaster recovery procedures are built-in and automated.

### Rancher Versions

**Rancher v2.x**:
- Modern architecture based on Kubernetes
- Complete rewrite from v1.x
- Multi-cluster management focus
- Current production version

**RKE (Rancher Kubernetes Engine)**:
- Production-ready Kubernetes distribution
- Deployed via Docker containers
- Fast and simple installation
- CNCF certified Kubernetes

**RKE2**:
- Security-focused Kubernetes distribution
- Meets FIPS 140-2 requirements
- CIS hardened by default
- Built on containerd
- Recommended for government and regulated industries

**K3s**:
- Lightweight Kubernetes (< 50MB binary)
- Perfect for edge, IoT, CI/CD
- ARM support
- Low resource requirements (512MB RAM minimum)
- Single binary with embedded etcd

### Use Cases

**Multi-Cloud Strategy**: Manage Kubernetes clusters across AWS, Azure, GCP, and on-premises from a single interface. Avoid vendor lock-in while maintaining consistent operations.

**Edge Computing**: Deploy and manage thousands of K3s clusters at edge locations (retail stores, factories, cell towers). Centralized management with local autonomy.

**Enterprise Kubernetes**: Provide developers with self-service Kubernetes while maintaining governance, security, and compliance. Operations teams get visibility and control.

**Hybrid Cloud**: Seamlessly manage on-premises and cloud Kubernetes clusters together. Workload portability and consistent tooling across environments.

**Development and Testing**: Quick cluster provisioning for development teams. Isolated environments with resource controls. Easy cleanup and recreation.

**Migration to Kubernetes**: Import existing clusters, gradually migrate workloads, and standardize on Kubernetes across the organization.

## Architecture and Components

Rancher's architecture consists of a management cluster (Rancher server) and downstream clusters (managed Kubernetes clusters).

### Rancher Server Architecture

The Rancher server itself runs as a Kubernetes deployment in a management cluster.

#### Management Cluster

**Purpose**:
- Hosts the Rancher application
- Stores cluster configurations and metadata
- Manages authentication and authorization
- Coordinates operations across downstream clusters
- Runs Fleet, monitoring, and other Rancher components

**Installation Options**:

**High Availability (HA)**:
- Recommended for production
- 3+ nodes for etcd quorum
- Load balancer for Rancher pods
- External database or embedded etcd
- Horizontal scaling of Rancher pods

**Single Node**:
- Development and testing
- Docker-based installation
- Single point of failure
- Quick setup
- Not recommended for production

**Hosted Kubernetes**:
- Deploy on existing EKS, AKS, GKE
- Leverage managed control plane
- Simplified operations
- Pay for managed service

#### Rancher Components

**Rancher API Server**:
- RESTful API for all Rancher operations
- Handles authentication and authorization
- Proxies requests to downstream clusters
- Manages Rancher resources (clusters, projects, catalogs)
- Serves the Rancher UI

**Cluster Controller**:
- Monitors downstream cluster state
- Deploys and manages cluster agents
- Handles cluster provisioning
- Manages cluster upgrades
- Reconciles cluster configuration

**Auth Service**:
- Integrates with identity providers
- Manages user sessions
- Handles authentication tokens
- Maps external groups to Rancher roles
- Provides SSO capabilities

**Catalog Service**:
- Manages Helm chart repositories
- Provides application marketplace
- Handles chart deployment
- Manages app versions and upgrades

**Fleet**:
- GitOps continuous delivery
- Multi-cluster deployment orchestration
- Git repository monitoring
- Cluster grouping and targeting
- Drift detection and remediation

**Cluster Agent**:
- Runs in each downstream cluster
- Maintains connection to Rancher server
- Reports cluster status
- Executes Rancher commands
- Handles workload management

**Node Agent**:
- Runs on each node in RKE clusters
- Manages Docker containers for Kubernetes components
- Handles node registration
- Monitors node health

### Downstream Cluster Types

Rancher manages different types of Kubernetes clusters:

#### RKE Clusters (Rancher Kubernetes Engine)

**Characteristics**:
- Deployed via Docker containers
- Rancher's original Kubernetes distribution
- Fast provisioning (5-10 minutes)
- Full customization of Kubernetes versions
- Easy upgrades and rollbacks

**Node Roles**:
- **etcd**: Distributed key-value store
- **Control Plane**: API server, scheduler, controller manager
- **Worker**: Run application workloads
- Can combine roles on same node

**Provisioning Methods**:

**Infrastructure Providers**:
- AWS EC2, Azure VMs, GCP Compute
- DigitalOcean, Linode, Vultr
- vSphere, Harvester
- Rancher provisions VMs and installs Kubernetes

**Custom Nodes**:
- Bring your own nodes (bare metal, VMs)
- Run Rancher agent on nodes
- Assign roles during registration
- Flexible for any infrastructure

**Cloud Drivers**:
- Node templates for consistent provisioning
- Credentials management
- Auto-scaling groups
- Multiple regions/zones

#### RKE2 Clusters

**Characteristics**:
- Security-focused distribution
- FIPS 140-2 compliant
- CIS hardened by default
- Uses containerd (not Docker)
- Systemd-based services
- Binary distribution (not containers)

**Security Features**:
- SELinux support
- Etcd automatic snapshots
- Secrets encryption at rest
- Pod Security Standards enforcement
- CIS benchmark alignment

**Use Cases**:
- Government deployments
- Regulated industries (healthcare, finance)
- Security-focused organizations
- Compliance requirements (FedRAMP, HIPAA)

#### K3s Clusters

**Characteristics**:
- Lightweight (< 50MB binary)
- Low resource usage (512MB RAM minimum)
- Single binary installation
- Embedded SQLite (default) or external database
- Perfect for edge and resource-constrained environments

**Features**:
- ARM support (Raspberry Pi)
- Built-in local storage (local-path-provisioner)
- Simplified networking (Flannel)
- Minimal dependencies
- Fast startup

**Use Cases**:
- Edge computing
- IoT devices
- CI/CD agents
- Developer laptops
- Embedded systems

#### Hosted Kubernetes Clusters

**Importing Existing Clusters**:
- Amazon EKS
- Azure AKS
- Google GKE
- Alibaba ACK
- Any Kubernetes cluster

**Capabilities**:
- Full Rancher management features
- RBAC integration
- Monitoring and logging
- Application deployment
- No control over cluster provisioning

**Provisioning Hosted Clusters**:
- Create EKS, AKS, GKE through Rancher
- Simplified configuration
- Rancher manages credentials
- Integrated billing and quotas

#### Registered Clusters

**Purpose**:
- Import any existing Kubernetes cluster
- Minimal Rancher footprint
- Read-only or full management
- Migration path to Rancher

**Process**:
1. Generate registration command in Rancher
2. Apply manifest to target cluster
3. Cluster appears in Rancher
4. Grant permissions as needed

### Networking Architecture

**Rancher Server to Downstream Clusters**:

**Agent-Initiated Connection**:
- Cluster agent initiates connection to Rancher
- WebSocket tunnel for bidirectional communication
- No inbound connectivity required to clusters
- Firewall-friendly design
- Works across NAT and proxies

**Communication Flow**:
```
Rancher Server (Management Cluster)
    ↑ WebSocket
    |
Cluster Agent (Downstream Cluster)
    ↓ kubectl commands
Kubernetes API Server
```

**Authorized Cluster Endpoint**:
- Direct access to downstream cluster API
- Bypass Rancher for kubectl commands
- Reduces latency
- Offline cluster management
- Requires network connectivity to cluster

### Storage Architecture

**Rancher Management Cluster Storage**:
- Persistent storage for Rancher configuration
- etcd for cluster metadata
- Application data (monitoring, logging)
- Backup snapshots

**Downstream Cluster Storage**:
- Managed independently per cluster
- Storage classes configured per cluster
- Longhorn integration for persistent storage
- CSI drivers for cloud providers

### High Availability Architecture

**Management Cluster HA**:
```
                Load Balancer
               /      |      \
              /       |       \
    Rancher Pod  Rancher Pod  Rancher Pod
    (Node 1)     (Node 2)     (Node 3)
         \          |          /
          \         |         /
           \        |        /
          etcd   etcd   etcd
        (Cluster State Storage)
```

**Components**:
- **Load Balancer**: Layer 4 or Layer 7, health checks on `/healthz`
- **Rancher Pods**: 3+ replicas for redundancy
- **etcd**: Embedded or external, 3/5/7 nodes
- **Persistent Storage**: For Rancher data and logs

**Downstream Cluster HA**:
- Independent HA configuration per cluster
- Multiple control plane nodes
- Multiple worker nodes
- Multi-zone distribution

## Cluster Management

Rancher provides comprehensive cluster lifecycle management from creation to deletion.

### Cluster Creation

#### Creating RKE Clusters

**AWS EC2 Example**:
1. **Cloud Credentials**: Add AWS access key/secret
2. **Node Template**: Define EC2 instance configuration
   - AMI, instance type, security groups
   - IAM profile, VPC, subnets
   - Block device mappings
3. **Cluster Configuration**: 
   - Cluster name, Kubernetes version
   - Node pools (etcd, control plane, worker)
   - Networking (Flannel, Calico, Canal, Weave)
   - Cloud provider integration
4. **Advanced Options**:
   - Add-ons (monitoring, logging)
   - Private registry authentication
   - Custom Kubernetes parameters
5. **Create**: Rancher provisions nodes and installs Kubernetes

**Custom Cluster**:
1. **Create Cluster**: Select "Custom" option
2. **Configuration**: Name, version, network plugin
3. **Node Registration**: Get registration command
4. **Run Command**: Execute on each node with role flags
   ```bash
   sudo docker run -d --privileged --restart=unless-stopped \
     --net=host -v /etc/kubernetes:/etc/kubernetes \
     -v /var/run:/var/run rancher/rancher-agent:v2.7.0 \
     --server https://rancher.example.com \
     --token <token> \
     --ca-checksum <checksum> \
     --etcd --controlplane --worker
   ```
5. **Cluster Provisioning**: Rancher installs Kubernetes components

#### Creating RKE2 Clusters

**Process**:
1. **Select RKE2**: Choose RKE2 cluster type
2. **Node Configuration**: Define node pools
3. **Security Settings**: Enable FIPS, CIS profile
4. **Networking**: CNI selection (Canal, Calico, Cilium)
5. **System Configuration**: Hardening options
6. **Provision**: Rancher installs RKE2 on nodes

**Security Configurations**:
- FIPS mode for cryptographic compliance
- CIS profile application
- Pod Security Standards
- SELinux enforcement
- Secrets encryption

#### Creating K3s Clusters

**Quick Setup**:
1. **Select K3s**: Choose K3s cluster type
2. **Minimal Configuration**: Name and version
3. **Node Registration**: Lightweight agent
4. **Fast Deployment**: Ready in minutes
5. **Edge Optimized**: Low resource usage

#### Importing Clusters

**Import Process**:
1. **Select Import**: Choose "Import Existing"
2. **Cluster Name**: Provide descriptive name
3. **Generate Manifest**: Rancher creates YAML
4. **Apply Manifest**: Run kubectl apply on target cluster
   ```bash
   kubectl apply -f import-manifest.yaml
   ```
5. **Cluster Appears**: Available in Rancher immediately

**Import Options**:
- **Full Management**: Complete Rancher control
- **Read-Only**: Monitoring and visibility only
- **Cluster Members**: Define initial access

### Cluster Configuration

**Basic Settings**:
- Cluster name and description
- Kubernetes version
- Network provider (CNI)
- Cloud provider integration
- Private registry authentication

**Advanced Settings**:

**Kubernetes Options**:
- API server arguments
- Controller manager configuration
- Scheduler parameters
- Kubelet options
- kube-proxy settings

**Network Configuration**:
- Pod CIDR range
- Service CIDR range
- Cluster DNS provider
- Network plugin configuration
- Dual-stack IPv4/IPv6

**Cloud Provider**:
- AWS cloud controller manager
- Azure cloud provider
- GCP integration
- vSphere cloud provider
- Custom cloud provider

**Add-ons**:
- Monitoring (Prometheus/Grafana)
- Logging (Elasticsearch/Fluentd/Kibana)
- Istio service mesh
- Longhorn storage
- OPA Gatekeeper

**Upgrade Strategy**:
- Control plane upgrade settings
- Worker node upgrade method
- Drain timeout configuration
- Max unavailable nodes

### Cluster Upgrade

**Kubernetes Version Upgrade**:
1. **Check Compatibility**: Review version compatibility
2. **Backup Cluster**: Create etcd snapshot
3. **Select Version**: Choose target Kubernetes version
4. **Initiate Upgrade**: Start upgrade process
5. **Monitor Progress**: Watch upgrade status

**Upgrade Process**:
- Control plane nodes upgraded first
- One node at a time (rolling upgrade)
- Worker nodes upgraded after control plane
- Configurable drain settings
- Automatic rollback on failure

**Upgrade Strategies**:

**Rolling Upgrade**:
- Default method
- Minimal downtime
- Gradual rollout
- Safe for production

**Batch Upgrade**:
- Upgrade multiple nodes simultaneously
- Faster completion
- Higher risk
- Better for non-production

### Cluster Operations

**Scaling Clusters**:

**Node Pools**:
- Add/remove node pools
- Change pool size
- Modify node configuration
- Auto-scaling integration

**Manual Scaling**:
- Add individual nodes
- Remove specific nodes
- Drain and delete nodes
- Cordon nodes

**Cluster Snapshots**:

**etcd Snapshots**:
- On-demand snapshots
- Scheduled automatic snapshots
- Snapshot retention policies
- Snapshot location (local, S3)

**Creating Snapshot**:
1. Navigate to cluster
2. Select "Snapshots" tab
3. Click "Take Snapshot"
4. Provide snapshot name
5. Wait for completion

**Restoring from Snapshot**:
1. Select snapshot to restore
2. Click "Restore"
3. Confirm restoration
4. Cluster restored to snapshot state

**Cluster Rotation**:

**Certificate Rotation**:
- Rotate Kubernetes certificates
- CA certificate rotation
- Service account token rotation
- Webhook certificate rotation

**Credential Rotation**:
- Update cloud provider credentials
- Rotate registry credentials
- Update API tokens
- Refresh service account keys

**Cluster Deletion**:

**Safe Deletion Process**:
1. **Backup**: Create final snapshot
2. **Drain Workloads**: Move applications
3. **Delete**: Confirm cluster deletion
4. **Cleanup**: Remove cloud resources

**Delete Options**:
- Delete cluster only (keep nodes)
- Delete cluster and nodes
- Delete cloud resources
- Keep backups

### Cluster Templates

**Purpose**:
- Standardize cluster configuration
- Enforce organizational policies
- Simplify cluster creation
- Version control for cluster specs

**Template Components**:
- Kubernetes version
- Network plugin
- Cloud provider settings
- Add-ons and monitoring
- Security policies

**Creating Templates**:
1. **Define Configuration**: Set all parameters
2. **Save as Template**: Name and describe
3. **Version Control**: Track template changes
4. **Share**: Make available to users
5. **Enforce**: Require template usage

**Template Revisions**:
- Version history
- Rollback capabilities
- Change tracking
- Audit trail

**Cluster from Template**:
1. Select template
2. Choose revision
3. Override parameters (if allowed)
4. Provision cluster
5. Template compliance maintained

### Node Management

**Node Roles in RKE**:

**etcd Nodes**:
- Run etcd distributed database
- Store cluster state
- Odd number required (3, 5, 7)
- High I/O requirements
- Should not run workloads

**Control Plane Nodes**:
- API server, scheduler, controller manager
- Cluster management operations
- Can combine with etcd role
- 2+ recommended for HA
- Moderate resource requirements

**Worker Nodes**:
- Run application workloads
- Can be scaled independently
- Varied instance types
- Auto-scaling friendly
- Most nodes in cluster

**Node Operations**:

**Cordon Node**:
- Mark node unschedulable
- Prevent new pods
- Existing pods remain
- Use for maintenance

**Drain Node**:
- Evict all pods
- Respects PodDisruptionBudgets
- Graceful termination
- Prepare for removal

**Delete Node**:
- Remove from cluster
- Clean up resources
- Update cloud infrastructure
- Permanent operation

**Node Templates**:
- Reusable node configurations
- Cloud-specific settings
- Instance type, AMI, security
- Simplify node addition

**Node Pools**:
- Group of nodes with same configuration
- Separate pools for different workload types
- Independent scaling
- Different instance types

## Multi-Cluster Management

Rancher's core strength is managing multiple Kubernetes clusters from a single interface.

### Global View

**Dashboard**:
- Overview of all clusters
- Cluster health status
- Resource utilization metrics
- Alert summary
- Recent events

**Cluster Metrics**:
- CPU usage across clusters
- Memory utilization
- Pod count and status
- Node availability
- Storage consumption

**Multi-Cluster Search**:
- Search across all clusters
- Find resources by name/label
- Filter by cluster, namespace, type
- Quick access to resources

### Cluster Groups

**Purpose**:
- Organize clusters logically
- Apply policies consistently
- Simplify fleet deployments
- Manage similar clusters together

**Grouping Strategies**:
- By environment (dev, staging, prod)
- By region (us-east, eu-west, ap-south)
- By application (app-a, app-b)
- By team (platform, data, ml)
- By customer (multi-tenant scenarios)

**Cluster Labels**:
- Key-value pairs for classification
- Used by Fleet for targeting
- Policy application
- Monitoring and alerting
- Automation triggers

### Global DNS

**Purpose**:
- Manage DNS across clusters
- External DNS integration
- Global load balancing
- Failover and disaster recovery

**Providers**:
- Route53 (AWS)
- Cloud DNS (GCP)
- Azure DNS
- Akamai
- CloudFlare

**Configuration**:
- DNS provider credentials
- Hosted zone configuration
- TTL and record types
- Health check integration

**Use Cases**:
- Multi-cluster ingress
- Geographic load distribution
- Active-active deployments
- Disaster recovery

### Global Catalog

**Centralized App Marketplace**:
- Deploy apps to multiple clusters
- Consistent versions across environments
- Centralized update management
- Custom catalog support

**Catalog Types**:
- Helm charts (Helm 3)
- Rancher apps
- Partner catalogs
- Custom catalogs

**Multi-Cluster App Deployment**:
1. Select app from catalog
2. Choose target clusters
3. Configure per-cluster settings
4. Deploy simultaneously
5. Monitor deployment status

### Cross-Cluster Resource Management

**Workload Migration**:
- Move workloads between clusters
- Disaster recovery scenarios
- Load balancing across clusters
- Blue-green deployments

**Configuration Sync**:
- Sync ConfigMaps and Secrets
- Consistent configuration
- Environment-specific overrides
- Automated updates

**Resource Quotas**:
- Cluster-level quotas
- Project-level quotas
- Namespace-level quotas
- Hierarchical enforcement

## Projects and Namespaces

Projects are Rancher's abstraction layer above Kubernetes namespaces, providing multi-tenancy and resource isolation.

### Project Concept

**Purpose**:
- Group related namespaces
- Apply policies consistently
- Multi-tenant isolation
- Simplified access control
- Resource quota management

**Project vs Namespace**:
- **Project**: Rancher construct, groups namespaces
- **Namespace**: Kubernetes construct, isolates resources
- Projects contain one or more namespaces
- Namespace belongs to one project
- RBAC applied at project level cascades to namespaces

**Default Projects**:
- **System**: Kubernetes system components (kube-system, cattle-system)
- **Default**: User workloads without specific project

### Creating Projects

**Project Creation**:
1. **Navigate to Cluster**: Select target cluster
2. **Projects/Namespaces**: Access project management
3. **Add Project**: Click create
4. **Configuration**:
   - Project name and description
   - Resource quotas
   - Resource limits
   - Network isolation
   - Pod security policy
5. **Members**: Add users and assign roles
6. **Create**: Project ready for use

**Project Configuration**:

**Resource Quotas**:
- Limit total resources in project
- CPU, memory, storage requests/limits
- Number of pods, services, volumes
- Prevents resource exhaustion

**Container Default Resource Limit**:
- Default CPU and memory for containers
- Applied if not specified in pod spec
- Ensures resource management
- Prevents unlimited consumption

**Network Isolation**:
- Enable project network isolation
- Pods can only communicate within project
- Kubernetes NetworkPolicies applied
- External communication explicitly allowed

**Pod Security Policy**:
- Select PSP template
- Restricted, unrestricted, or custom
- Applied to all project namespaces
- Security baseline enforcement

### Namespace Management

**Creating Namespaces**:
1. **Select Project**: Choose parent project
2. **Add Namespace**: Create new namespace
3. **Configuration**: Name and labels
4. **Resource Quotas**: Namespace-specific limits
5. **Create**: Namespace ready

**Namespace Operations**:
- Move between projects
- Resource quota assignment
- Label and annotation management
- Namespace deletion

**System Namespaces**:
- `cattle-system`: Rancher components
- `kube-system`: Kubernetes system pods
- `kube-public`: Public cluster information
- `ingress-nginx`: Nginx ingress controller
- `cattle-monitoring-system`: Monitoring stack

### Project Members and Roles

**Project Roles**:

**Project Owner**:
- Full control over project
- Manage members and permissions
- Configure project settings
- Create and delete namespaces
- Deploy applications

**Project Member**:
- Deploy and manage workloads
- View project resources
- Cannot modify project settings
- Cannot manage members

**Read-Only**:
- View project resources
- Cannot modify anything
- Useful for monitoring and auditing

**Custom Roles**:
- Define specific permissions
- Combine individual rights
- Tailored to organization needs

**Adding Members**:
1. **Navigate to Project**: Select project
2. **Members Tab**: Access member management
3. **Add Member**: Select user/group
4. **Assign Role**: Choose appropriate role
5. **Save**: Member has access

### Resource Quotas

**Project Resource Quotas**:
- Applied across all project namespaces
- Cumulative limits
- Prevents single namespace from consuming all resources

**Configuration**:
```yaml
CPU Reservation: 10 cores
CPU Limit: 20 cores
Memory Reservation: 20 GB
Memory Limit: 40 GB
Storage: 100 GB
Pods: 100
Services: 50
ConfigMaps: 100
Secrets: 100
```

**Namespace Resource Quotas**:
- Specific limits per namespace
- Must fit within project quota
- More granular control

**Enforcement**:
- Admission controller checks quotas
- Rejects resources exceeding quota
- Real-time quota tracking
- Alerts on quota approaching limit

### Project Network Isolation

**Purpose**:
- Prevent unauthorized cross-namespace communication
- Implement zero-trust networking
- Compliance requirements
- Security best practices

**Implementation**:
- Kubernetes NetworkPolicies
- Default deny all ingress
- Explicitly allow required traffic
- Project-level isolation

**Allowed Traffic**:
- Within project namespaces
- From ingress controller
- To external services (if configured)
- DNS queries

**Configuration**:
1. Enable project network isolation
2. Define allowed ingress sources
3. Define allowed egress destinations
4. Apply policies automatically

### Multi-Tenancy with Projects

**Tenant Isolation**:
- Each tenant gets a project
- Complete resource isolation
- Separate RBAC boundaries
- Independent resource quotas

**Organizational Structure**:
```
Cluster
├── Project: Team-A
│   ├── Namespace: team-a-dev
│   ├── Namespace: team-a-staging
│   └── Namespace: team-a-prod
├── Project: Team-B
│   ├── Namespace: team-b-dev
│   └── Namespace: team-b-prod
└── Project: Shared-Services
    ├── Namespace: monitoring
    ├── Namespace: logging
    └── Namespace: ingress
```

**Benefits**:
- Clear ownership boundaries
- Self-service for teams
- Centralized governance
- Cost allocation and tracking

## Users and Authentication

Rancher provides flexible authentication options and centralized user management across all clusters.

### Authentication Providers

Rancher supports multiple authentication providers for enterprise integration.

#### Local Authentication

**Built-in Provider**:
- Username/password authentication
- No external dependencies
- Good for testing and small deployments
- Not recommended for production at scale

**Local Users**:
- Admin user (created during installation)
- Additional local users
- Password policies
- API keys and tokens

#### Active Directory (AD)

**Integration**:
- LDAP-based authentication
- Group synchronization
- Nested group support
- Multiple AD servers

**Configuration**:
- AD server hostname and port
- TLS/StartTLS encryption
- Service account credentials
- User/group search base DNs
- Search filters and attributes

**Features**:
- Single sign-on
- Automatic group membership
- Group-based access control
- Regular synchronization

#### LDAP

**OpenLDAP, FreeIPA**:
- Standard LDAP protocol
- Similar to AD configuration
- Group membership mapping
- Attribute-based search

**Configuration**:
- LDAP server endpoints
- Bind credentials
- Search scope and filters
- Group member attribute

#### SAML

**SAML 2.0 Providers**:
- Okta, OneLogin, Ping Identity
- Azure AD SAML
- Custom SAML IdPs
- Federation support

**Configuration**:
- Metadata URL or file
- Entity ID and service URL
- SSO endpoint
- Certificate validation

**Attributes Mapping**:
- UID, display name, email
- Group membership
- Custom attributes

#### OAuth/OIDC

**Supported Providers**:
- **GitHub**: Organization/team-based access
- **Google**: G Suite integration
- **Azure AD**: Microsoft 365 integration
- **Keycloak**: Open-source IdP
- **Generic OIDC**: Any compliant provider

**GitHub Example**:
- OAuth application registration
- Organization and team sync
- Public/private repository access
- Fine-grained permissions

**Configuration**:
- Client ID and secret
- Authorization/token endpoints
- Scopes and claims
- Group claim mapping

#### Certificate-Based Authentication

**mTLS Authentication**:
- Client certificate validation
- PKI infrastructure
- Certificate CN/OU mapping
- High security scenarios

### User Management

**Creating Users** (Local Auth):
1. **Global > Security > Users**
2. **Add User**: Click create
3. **User Details**: Username, password, display name
4. **Global Role**: Assign permissions
5. **Create**: User can log in

**User Properties**:
- Username (unique identifier)
- Display name
- Email address
- Global permissions
- Enabled/disabled status

**User Sessions**:
- TTL configuration
- Session revocation
- Concurrent sessions
- Activity tracking

**API Keys and Tokens**:
- Personal API keys (user-specific)
- Cluster API keys (cluster access)
- Project API keys (project access)
- Token expiration
- Token scoping

**Creating API Keys**:
1. **User Profile > API & Keys**
2. **Add Key**: Create new
3. **Scope**: Global, cluster, or project
4. **Expiration**: Never, or set date
5. **Save**: Copy key (shown once)

### Groups and Team Management

**External Groups**:
- Synced from authentication provider
- Automatic membership updates
- Group-based RBAC
- Simplified access management

**Group Synchronization**:
- Regular sync intervals
- Manual refresh option
- Group nesting support
- Membership caching

**Assigning Permissions to Groups**:
1. **Navigate to Resource**: Cluster or project
2. **Members Tab**: Add member
3. **Select Group**: Choose from list
4. **Assign Role**: Set permissions
5. **Save**: All group members get access

**Best Practices**:
- Use groups over individual users
- Mirror organizational structure
- Regular group audits
- Principle of least privilege

## RBAC and Authorization

Rancher implements a hierarchical RBAC model extending Kubernetes native RBAC.

### Global Roles

Applied across all clusters and resources in Rancher.

**Administrator**:
- Complete Rancher access
- Manage all clusters
- Configure authentication
- Manage users and permissions
- Global settings control
- Reserved for platform administrators

**Standard User**:
- Create clusters
- Import clusters
- Access assigned clusters
- Cannot manage global settings
- Cannot manage other users
- Most common role

**User-Base** (Restricted User):
- No cluster creation
- Only access explicitly granted clusters
- Cannot import clusters
- Minimal default permissions
- Suitable for regulated environments

**Custom Global Roles**:
- Combine specific permissions
- Fine-tuned access control
- Organization-specific needs

**Global Permissions**:
- Manage authentication
- Manage catalogs
- Manage clusters
- Manage node drivers
- Manage pod security policies
- Manage roles
- Manage settings
- Manage users

### Cluster Roles

Applied within a specific cluster.

**Cluster Owner**:
- Full control over cluster
- Manage cluster members
- Create and manage projects
- Configure cluster settings
- Delete cluster
- View all resources

**Cluster Member**:
- Create projects
- View cluster resources
- Cannot modify cluster settings
- Cannot manage cluster members
- Access to assigned projects

**Cluster Viewer** (Read-Only):
- View cluster details
- View projects and namespaces
- Cannot create or modify
- Useful for monitoring and auditing

**Custom Cluster Roles**:
- Tailored permissions
- Specific resource access
- Verb-based control (get, list, create, update, delete)

**Cluster-Level Permissions**:
- Manage cluster members
- Create projects
- Manage namespaces
- View cluster monitoring
- Manage cluster catalogs
- Configure cluster drivers

### Project Roles

Applied within a specific project (and its namespaces).

**Project Owner**:
- Full project control
- Manage project members
- Create namespaces
- Configure project settings
- Deploy applications
- Manage resources

**Project Member**:
- Deploy workloads
- Create namespaces
- Manage applications
- View project resources
- Cannot manage project settings
- Cannot manage members

**Project Read-Only**:
- View all project resources
- Cannot modify anything
- Good for auditors and viewers

**Custom Project Roles**:
- Granular permissions
- Resource-specific access
- Workflow-based roles

**Project-Level Permissions**:
- Create namespaces
- Manage workloads
- Manage services and ingresses
- Manage volumes
- Manage ConfigMaps and Secrets
- View logs and metrics
- Execute into containers

### Permission Inheritance

**Hierarchical Model**:
```
Global Role
    ↓ (can grant)
Cluster Role
    ↓ (can grant)
Project Role
    ↓ (applies to)
Namespace Resources
```

**Rules**:
- Global admin can do anything
- Cluster owner can manage all projects
- Project owner can manage all namespaces
- Higher role can grant lower roles
- Cannot escalate own privileges

### Custom Roles

**Creating Custom Roles**:
1. **Global > Security > Roles**
2. **Add Role**: Select scope (global, cluster, project)
3. **Grant Resources**: Select Kubernetes resources
4. **Grant Verbs**: Choose operations (get, list, create, update, delete)
5. **Built-in Roles**: Inherit from existing
6. **Create**: Role available for assignment

**Example Custom Role** (Developer):
- Get, list, create, update, delete: Deployments, Services, Ingress
- Get, list: Pods, Logs
- No access: Secrets, RBAC resources
- Restricted namespace creation

### Kubernetes RBAC Integration

**Rancher to Kubernetes Mapping**:
- Rancher roles create Kubernetes Roles and RoleBindings
- Project roles create RoleBindings in project namespaces
- Cluster roles create ClusterRoles and ClusterRoleBindings
- Seamless integration

**Direct Kubernetes RBAC**:
- Can use kubectl to manage RBAC
- Rancher UI shows all bindings
- Mix Rancher and native RBAC
- Rancher roles preferred for consistency

### Best Practices

**Access Control**:
- Use authentication providers (not local auth)
- Group-based permissions
- Principle of least privilege
- Regular access reviews
- Service accounts for automation

**Role Management**:
- Use built-in roles when possible
- Custom roles for specific needs
- Document custom role purposes
- Version control role definitions
- Test role permissions

**Security**:
- MFA for administrators
- API key rotation
- Audit log monitoring
- Restrict cluster owner role
- Separate production access

## Catalogs and Apps

Rancher's application catalog system simplifies deploying and managing applications across clusters.

### Catalog Types

**Helm Chart Repositories**:
- Standard Helm chart repositories
- Helm 3 support
- Public and private repositories
- OCI registry support

**Rancher Charts**:
- Official Rancher applications
- Optimized for Rancher
- Pre-configured monitoring, logging
- Integrated with Rancher features

**Partner Catalogs**:
- Verified partner applications
- Enterprise software
- Commercial support available
- Regular updates

**Custom Catalogs**:
- Organization-specific charts
- Internal applications
- Private Git repositories
- S3/HTTP hosting

### Built-in Catalogs

**Rancher Library**:
- Curated Helm charts
- Maintained by Rancher team
- Popular applications
- Regular security updates

**Rancher Partner Charts**:
- Partner-verified applications
- Commercial software
- Enterprise-grade apps
- Support agreements

**Helm Stable** (Legacy):
- Original Helm stable charts
- Deprecated in favor of individual repos
- Historical reference

### Adding Custom Catalogs

**Git Repository Catalog**:
1. **Global > Apps > Manage Catalogs**
2. **Add Catalog**: Click create
3. **Configuration**:
   - Name and URL
   - Git branch
   - Authentication (private repos)
   - Helm version
4. **Create**: Catalog refreshes

**Helm Repository**:
1. **Add Catalog**: Select Helm repository
2. **Repository URL**: Provide endpoint
3. **Authentication**: Username/password if required
4. **Create**: Charts available

**OCI Registry**:
- Docker-compatible registries
- Helm charts stored as OCI artifacts
- Authentication via Docker credentials
- Modern approach to chart distribution

**Catalog Requirements**:
- Helm chart format
- Chart.yaml metadata
- values.yaml for configuration
- README for documentation
- NOTES.txt for post-install info

### Deploying Applications

**From Catalog**:
1. **Navigate to Cluster/Project**
2. **Apps > Charts**
3. **Browse Catalog**: Search for application
4. **Select Chart**: Choose version
5. **Configure**:
   - Application name and namespace
   - values.yaml customization
   - Answers (form-based values)
6. **Install**: Deploy application

**Chart Configuration Options**:

**Form-Based (Answers)**:
- GUI input fields
- Defined by questions.yaml
- User-friendly for common options
- Limited to predefined questions

**values.yaml**:
- Full Helm values file
- Complete customization
- YAML editor in UI
- Power user option

**Deployment Options**:
- Target namespace
- Project assignment
- Resource allocation
- Custom labels and annotations

### Managing Applications

**App Operations**:

**Upgrade**:
- New chart version
- Values modifications
- Rolling deployment
- Rollback capability

**Rollback**:
- Return to previous version
- Restore previous values
- Quick recovery
- Version history maintained

**Delete**:
- Remove application
- Clean up resources
- Optional resource retention
- Namespace cleanup

**Application Status**:
- Deployment progress
- Resource status (Deployments, Pods, Services)
- Events and logs
- Health checks

**Application Details**:
- Kubernetes resources created
- values.yaml used
- Chart version deployed
- Revision history
- Related resources

### Multi-Cluster Apps

**Global Applications**:
- Deploy to multiple clusters simultaneously
- Consistent versions across environments
- Centralized management
- Staged rollouts

**Creating Multi-Cluster App**:
1. **Global > Apps > Multi-Cluster Apps**
2. **Add App**: Select chart
3. **Target Clusters**: Choose destinations
4. **Configuration**: 
   - Per-cluster values
   - Shared configuration
   - Cluster-specific overrides
5. **Deploy**: Rollout to all clusters

**Rollout Strategies**:
- **All clusters simultaneously**: Fast deployment
- **Staged rollout**: One cluster at a time
- **Cluster groups**: Deploy to groups sequentially
- **Manual approval**: Require confirmation per stage

**Use Cases**:
- Monitoring stack across all clusters
- Logging infrastructure
- Security tools (policy enforcement)
- Service mesh control plane
- Centralized management tools

### App Management Best Practices

**Chart Management**:
- Pin chart versions in production
- Test upgrades in non-production
- Review chart changelogs
- Monitor for security updates
- Maintain private chart repository

**Configuration Management**:
- Version control values files
- Environment-specific values
- Secrets in external stores
- ConfigMap patterns
- GitOps for app deployment

**Lifecycle Management**:
- Regular upgrade schedule
- Rollback procedures documented
- Health monitoring
- Dependency management
- Deprecation planning

## Monitoring and Logging

Rancher provides integrated monitoring and logging capabilities powered by industry-standard tools.

### Monitoring Stack

**Architecture**:
- **Prometheus**: Metrics collection and storage
- **Grafana**: Visualization and dashboards
- **Alertmanager**: Alert routing and notification
- **Prometheus Operator**: Kubernetes-native management

**Deployment**:
1. **Cluster > Tools > Monitoring**
2. **Install Monitoring**: Enable monitoring
3. **Configuration**:
   - Resource allocation
   - Data retention period
   - Persistent storage
   - Ingress configuration
4. **Install**: Deploy stack

**Components Deployed**:
- Prometheus server(s)
- Grafana instance
- Alertmanager
- Node exporter (metrics from nodes)
- kube-state-metrics (Kubernetes object metrics)
- Prometheus operator
- Service monitors

### Metrics Collection

**Cluster Metrics**:
- Node CPU, memory, disk, network
- Kubernetes API server metrics
- etcd performance metrics
- Control plane component health
- Cluster-wide resource usage

**Workload Metrics**:
- Pod CPU and memory usage
- Container resource consumption
- Deployment replica status
- Pod restarts and failures
- Request rates and latency

**Application Metrics**:
- Custom application metrics
- ServiceMonitor CRDs
- Prometheus annotations
- PodMonitor resources
- Automatic discovery

**Enabling Application Metrics**:
1. **Expose Metrics**: Application provides /metrics endpoint
2. **ServiceMonitor**: Create ServiceMonitor resource
   ```yaml
   apiVersion: monitoring.coreos.com/v1
   kind: ServiceMonitor
   metadata:
     name: myapp-metrics
   spec:
     selector:
       matchLabels:
         app: myapp
     endpoints:
     - port: metrics
       interval: 30s
   ```
3. **Prometheus**: Automatically discovers and scrapes

### Grafana Dashboards

**Pre-built Dashboards**:
- Cluster compute resources
- Namespace resource usage
- Node statistics
- Pod resources
- Deployment metrics
- StatefulSet metrics
- Ingress controller metrics
- etcd performance

**Dashboard Categories**:
- **Cluster**: Overall cluster health
- **Nodes**: Individual node metrics
- **Namespaces**: Resource usage per namespace
- **Workloads**: Deployment/StatefulSet status
- **Pods**: Detailed pod metrics

**Custom Dashboards**:
- Import from Grafana.com
- Create custom visualizations
- Share across teams
- Export as JSON
- Version control

**Accessing Grafana**:
1. **Cluster > Monitoring**
2. **Grafana**: Click to open
3. **SSO**: Automatic authentication via Rancher
4. **Browse**: Select dashboard

### Alerting

**Alert Rules**:
- Pre-defined alert rules
- Custom PrometheusRule resources
- PromQL-based conditions
- Severity levels (warning, critical)

**Default Alerts**:
- Node down
- High CPU/memory usage
- Disk space low
- Pod crash looping
- Deployment replica mismatch
- etcd performance degraded
- Certificate expiration

**Creating Custom Alerts**:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-alerts
spec:
  groups:
  - name: myapp
    rules:
    - alert: HighErrorRate
      expr: rate(http_requests_total{status="500"}[5m]) > 0.05
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "High error rate detected"
        description: "Error rate is {{ $value }}"
```

**Notification Channels**:

**Alertmanager Configuration**:
- Email notifications
- Slack integration
- PagerDuty
- Webhook (custom integrations)
- Microsoft Teams
- Opsgenie

**Configuring Notifications**:
1. **Cluster > Tools > Monitoring**
2. **Edit**: Modify monitoring config
3. **Alerting**: Configure Alertmanager
4. **Receivers**: Add notification channels
5. **Routes**: Define alert routing logic

**Alert Routing**:
```yaml
route:
  group_by: ['alertname', 'cluster']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'slack'
  routes:
  - match:
      severity: critical
    receiver: 'pagerduty'
  - match:
      team: platform
    receiver: 'platform-slack'
```

### Logging Stack

**Architecture**:
- **Fluentd/Fluent Bit**: Log collection
- **Elasticsearch**: Log storage and indexing
- **Kibana**: Log visualization and search
- Alternative: Loki (lightweight, simpler)

**Deployment Options**:

**Elasticsearch Stack**:
1. **Cluster > Tools > Logging**
2. **Install Logging**: Enable logging
3. **Target**: Elasticsearch cluster
4. **Configuration**:
   - Elasticsearch endpoint
   - Index patterns
   - Authentication
5. **Install**: Deploy log collectors

**Log Levels**:
- **Cluster-level**: All pods in cluster
- **Project-level**: Specific project pods
- **Workload-level**: Individual application logs

**Log Shipping Targets**:
- **Elasticsearch**: Full-text search, complex queries
- **Splunk**: Enterprise log management
- **Kafka**: Stream to message queue
- **Syslog**: Traditional syslog servers
- **Fluentd/Custom**: Custom log processors

### Log Access and Search

**Viewing Logs**:

**Real-time Logs**:
1. **Workload**: Navigate to Deployment/Pod
2. **Logs**: Click logs icon
3. **Live Stream**: View real-time output
4. **Container Selection**: Multi-container pods
5. **Timestamp**: Prefixed to each line

**Historical Logs**:
- Kibana interface
- Full-text search
- Time range selection
- Field-based filtering
- Log aggregation

**Kibana Features**:
- Discover (log search)
- Visualizations (charts, graphs)
- Dashboards (combined views)
- Saved searches
- Alert integration

**Log Search Examples**:
```
# Search by namespace
kubernetes.namespace_name:"production"

# Search by pod name
kubernetes.pod_name:myapp-*

# Search by log level
log.level:ERROR

# Combined search
kubernetes.namespace_name:"production" AND log.level:ERROR
```

### Monitoring and Logging Best Practices

**Monitoring**:
- Enable monitoring on all clusters
- Configure persistent storage for metrics
- Set appropriate data retention (30-90 days)
- Create custom dashboards for applications
- Alert on actionable conditions only
- Regular review of alert rules
- Document alert response procedures
- Monitor the monitoring stack itself

**Logging**:
- Enable logging early in cluster lifecycle
- Configure log retention policies
- Index optimization for search performance
- Regular log cleanup
- Structured logging in applications
- Correlation IDs for request tracing
- Sensitive data filtering
- Compliance with data regulations

**Performance**:
- Resource allocation for monitoring/logging
- Separate storage for metrics and logs
- Monitoring stack scaling
- Log sampling for high-volume apps
- Metric cardinality management
- Index lifecycle management (ILM)

**Integration**:
- Correlate metrics and logs
- Single pane of glass for observability
- Tracing integration (Jaeger, Zipkin)
- Service mesh observability (Istio, Linkerd)
- Application Performance Monitoring (APM)

## Security

Rancher provides multiple layers of security controls for cluster and workload protection.

### CIS Benchmark Scanning

**Purpose**:
- Assess cluster security posture
- Compliance with CIS Kubernetes Benchmark
- Identify security gaps
- Remediation guidance

**Running CIS Scans**:
1. **Cluster > Tools > CIS Scans**
2. **Run Scan**: Initiate scan
3. **Select Profile**: Choose CIS benchmark version
4. **Schedule**: One-time or recurring
5. **Execute**: Scan runs on all nodes

**Scan Results**:
- Pass/Fail for each check
- Severity levels (High, Medium, Low)
- Remediation steps
- Historical comparison
- Compliance score

**CIS Benchmark Checks**:
- Control plane configuration
- etcd security settings
- Worker node configuration
- Pod Security Policies
- Network policies
- RBAC configuration
- Secrets encryption
- Audit logging

**Remediation**:
- Automated fixes (where possible)
- Manual remediation steps
- Configuration templates
- Best practice guidance

### Pod Security Policies (PSPs)

**PSP Templates**:

**Restricted**:
- No privileged containers
- No host namespace access
- No host networking
- Read-only root filesystem
- Drop all capabilities
- Non-root user required
- Most secure, may break some apps

**Unrestricted**:
- No restrictions
- Full privileges allowed
- For trusted system components
- Not recommended for user workloads

**Custom PSPs**:
- Tailored to application needs
- Balance security and functionality
- Organizationally appropriate

**Applying PSPs**:
1. **Cluster**: Define PSPs at cluster level
2. **Project**: Apply PSP to project
3. **Namespace**: PSP inherited by namespaces
4. **Pods**: PSP validated on creation

**PSP Configuration**:
```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
  - ALL
  runAsUser:
    rule: MustRunAsNonRoot
  seLinux:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  volumes:
  - 'configMap'
  - 'emptyDir'
  - 'secret'
  - 'persistentVolumeClaim'
```

**Migration to Pod Security Standards**:
- PSP deprecated in Kubernetes 1.21
- Removed in Kubernetes 1.25
- Migrate to Pod Security Admission
- OPA Gatekeeper alternative
- Kyverno policy engine

### Network Security

**Network Policy Enforcement**:
- Project network isolation
- Default deny policies
- Explicit allow rules
- Microsegmentation

**Canal Network Plugin**:
- Combines Flannel (networking) and Calico (policies)
- NetworkPolicy support
- Simple setup
- Good performance

**Calico**:
- Advanced NetworkPolicy
- Global network policies
- Egress controls
- Encryption (WireGuard)

**Cilium**:
- eBPF-based networking
- Layer 7 policies
- Service mesh integration
- Advanced observability

**Network Policy Example**:
```yaml
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
```

### Secrets Management

**Kubernetes Secrets**:
- Base64 encoded (not encrypted by default)
- Stored in etcd
- Accessible via RBAC
- Mounted as volumes or environment variables

**Secrets Encryption**:
- Enable encryption at rest
- Kubernetes EncryptionConfiguration
- Key management
- Regular key rotation

**External Secrets**:

**HashiCorp Vault Integration**:
- Centralized secret storage
- Dynamic secrets
- Audit logging
- Fine-grained access control

**External Secrets Operator**:
- Sync secrets from external stores
- AWS Secrets Manager
- Azure Key Vault
- GCP Secret Manager
- HashiCorp Vault

**Best Practices**:
- Never commit secrets to Git
- Use external secret stores
- Rotate secrets regularly
- Limit secret access (RBAC)
- Audit secret usage
- Encrypt etcd
- Use sealed secrets for GitOps

### RBAC Enforcement

**Principle of Least Privilege**:
- Grant minimum necessary permissions
- Use projects for team isolation
- Custom roles for specific needs
- Regular access reviews

**Service Accounts**:
- Dedicated service accounts per application
- Limit permissions to required resources
- Avoid using default service account
- Token rotation

**Preventing Privilege Escalation**:
- Restrict role creation
- Limit cluster-admin access
- Audit role bindings
- Monitor for suspicious permissions

### Security Scanning

**Image Scanning**:
- Scan container images for vulnerabilities
- Integration with security scanners
- Block vulnerable images (admission control)
- Regular rescanning of running images

**Scanning Tools**:
- **Trivy**: Comprehensive vulnerability scanner
- **Clair**: Container vulnerability analysis
- **Anchore**: Policy-based compliance
- **Aqua Security**: Commercial solution
- **Sysdig**: Runtime security

**Admission Control**:
- OPA Gatekeeper for policy enforcement
- Kyverno for Kubernetes-native policies
- Image verification (signed images)
- Resource validation
- Mutation policies

### Compliance and Auditing

**Audit Logging**:
- Enable Kubernetes audit logging
- Record API requests
- Long-term log retention
- SIEM integration

**Compliance Frameworks**:
- CIS Kubernetes Benchmark
- PCI-DSS requirements
- HIPAA compliance
- SOC 2 controls
- GDPR data protection

**Audit Reports**:
- Access reviews
- Permission changes
- Resource modifications
- Login attempts
- Policy violations

### Security Best Practices

**Cluster Hardening**:
- Regular security updates
- Minimal node OS (RancherOS, Flatcar)
- Disable unnecessary services
- Firewall rules
- Network segmentation

**Workload Security**:
- Run as non-root user
- Read-only root filesystem
- Drop unnecessary capabilities
- Resource limits
- Security contexts

**Supply Chain Security**:
- Signed container images
- Image provenance verification
- Trusted registries only
- Vulnerability scanning
- SBOM (Software Bill of Materials)

**Operational Security**:
- Multi-factor authentication
- Strong password policies
- API key rotation
- Certificate management
- Regular security training

## Fleet (GitOps)

Fleet is Rancher's GitOps continuous delivery solution, designed to manage deployments across massive fleets of clusters.

### Fleet Architecture

**Components**:
- **Fleet Manager**: Runs in management cluster
- **Fleet Agent**: Runs in each managed cluster
- **Git Repository**: Source of truth for deployments
- **Bundle**: Unit of deployment (Helm chart, manifests, Kustomize)

**Workflow**:
```
Git Repository
     ↓ (monitors)
Fleet Manager
     ↓ (deploys)
Fleet Agents (per cluster)
     ↓ (applies)
Kubernetes Resources
```

### Fleet Concepts

**GitRepo**:
- Represents a Git repository
- Defines what to deploy
- Cluster targeting rules
- Update frequency

**Bundle**:
- Collection of Kubernetes resources
- Can be Helm charts, manifests, or Kustomize
- Deployed as atomic unit
- Version tracked

**ClusterGroup**:
- Logical grouping of clusters
- Based on cluster labels
- Used for targeting deployments
- Hierarchical organization

**Cluster**:
- Kubernetes cluster managed by Fleet
- Has labels for targeting
- Reports bundle status
- Receives deployments

### Setting Up Fleet

**Enable Fleet**:
- Enabled by default in Rancher 2.5+
- Runs in `fleet-system` namespace
- Automatic agent deployment

**Registering Clusters**:
1. **Cluster**: Imported/created clusters auto-registered
2. **Labels**: Apply labels for targeting
3. **ClusterGroup**: Add to groups (optional)
4. **Ready**: Cluster appears in Fleet

**Cluster Labels**:
```yaml
env: production
region: us-west
team: platform
purpose: apps
```

### Creating GitRepo

**GitRepo Resource**:
```yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: myapp
  namespace: fleet-default
spec:
  repo: https://github.com/myorg/myapp-fleet
  branch: main
  paths:
  - manifests/production
  targets:
  - name: production-clusters
    clusterSelector:
      matchLabels:
        env: production
```

**Via Rancher UI**:
1. **Continuous Delivery (Fleet)**
2. **Git Repos**: Add repository
3. **Configuration**:
   - Repository URL
   - Branch/tag/commit
   - Paths to deploy
   - Target clusters
4. **Authentication**: SSH key or HTTPS credentials
5. **Create**: Fleet monitors repository

### Targeting Clusters

**Cluster Selector**:
```yaml
targets:
- name: production-west
  clusterSelector:
    matchLabels:
      env: production
      region: us-west
```

**Cluster Group**:
```yaml
targets:
- clusterGroup: production
```

**Multiple Targets**:
```yaml
targets:
- name: all-production
  clusterSelector:
    matchLabels:
      env: production
- name: staging-eu
  clusterSelector:
    matchExpressions:
    - key: env
      operator: In
      values: [staging]
    - key: region
      operator: In
      values: [eu-central, eu-west]
```

**All Clusters**:
```yaml
targets:
- clusterSelector: {}  # Empty selector matches all
```

### Fleet.yaml Configuration

**Purpose**:
- Configure bundle deployment
- Helm values per cluster
- Kustomize overlays
- Diff options
- Dependencies

**Example fleet.yaml**:
```yaml
# Helm chart configuration
helm:
  chart: ./chart
  repo: https://charts.example.com/stable
  version: 1.2.3
  releaseName: myapp
  values:
    replicaCount: 3
    image:
      repository: myapp
      tag: v1.2.3

# Per-cluster customization
targetCustomizations:
- name: production
  clusterSelector:
    matchLabels:
      env: production
  helm:
    values:
      replicaCount: 5
      resources:
        limits:
          cpu: 2
          memory: 4Gi

- name: staging
  clusterSelector:
    matchLabels:
      env: staging
  helm:
    values:
      replicaCount: 2
      resources:
        limits:
          cpu: 1
          memory: 2Gi

# Kustomize support
kustomize:
  dir: ./overlays/production

# Diff options
diff:
  comparePatches:
  - apiVersion: apps/v1
    kind: Deployment
    operations:
    - {"op": "remove", "path": "/spec/replicas"}

# Dependencies
dependsOn:
- name: database
  selector:
    matchLabels:
      app: postgres
```

### Deployment Strategies

**Rolling Deployment**:
```yaml
spec:
  rolloutStrategy:
    maxUnavailable: 1
    maxUnavailablePartitions: 20%
```

**Staged Rollout**:
```yaml
targetCustomizations:
- name: canary
  clusterSelector:
    matchLabels:
      canary: "true"
  
- name: production
  clusterSelector:
    matchLabels:
      env: production
  deploymentStrategy:
    rollout:
      maxUnavailable: 1
      autoPartitionSize: 10%
      partitions:
      - name: canary
        maxUnavailable: 100%
      - name: region-1
        maxUnavailable: 20%
      - name: region-2
        maxUnavailable: 20%
```

**Blue-Green**:
- Deploy to separate cluster group
- Switch traffic via DNS/load balancer
- Manual or automated cutover
- Quick rollback capability

### Monitoring Fleet Deployments

**Bundle Status**:
- **Ready**: Successfully deployed on all targets
- **Modified**: Cluster resources differ from Git
- **NotReady**: Deployment failed or pending
- **WaitApplied**: Waiting for resources to be ready

**Fleet Dashboard**:
1. **Continuous Delivery (Fleet)**
2. **Repos/Clusters**: View status
3. **Bundle Details**: Per-cluster status
4. **Diff**: View modifications
5. **Logs**: Fleet agent logs

**Git Commit to Deployment**:
- Commit pushed to Git
- Fleet detects change (polling interval: 15s)
- Manager creates/updates bundles
- Agents pull and apply bundles
- Resources created/updated in clusters
- Status reported back to manager

**Drift Detection**:
- Fleet monitors for manual changes
- Marks bundle as "Modified"
- Can auto-reconcile (force sync)
- Alert on drift

### Fleet Best Practices

**Repository Structure**:
```
fleet-repo/
├── base/                    # Common configuration
│   ├── deployment.yaml
│   └── service.yaml
├── overlays/
│   ├── production/
│   │   ├── fleet.yaml
│   │   └── kustomization.yaml
│   ├── staging/
│   └── development/
└── README.md
```

**GitOps Workflow**:
- All changes via Git
- Pull request review process
- Automated testing in CI
- Staging deployment first
- Gradual production rollout
- Rollback via Git revert

**Security**:
- Private Git repositories
- SSH key authentication
- Read-only deploy keys
- No secrets in Git (use external stores)
- Signed commits

**Monitoring**:
- Alert on deployment failures
- Monitor bundle status
- Track deployment duration
- Drift detection alerts
- Git sync monitoring

## Backup and Disaster Recovery

Rancher provides built-in backup and disaster recovery capabilities for both the management cluster and downstream clusters.

### Rancher Backup

**Backup Operator**:
- Backs up Rancher management cluster
- Rancher configuration and state
- Cluster definitions and metadata
- etcd snapshots
- Disaster recovery

**Installation**:
1. **Cluster > Rancher Backups**
2. **Install Backup Operator**: Deploy backup CRD and controller
3. **Configuration**: Storage location (S3, Minio, local)
4. **Install**: Operator ready

**Backup Contents**:
- Rancher configuration
- Cluster registrations
- Project and namespace metadata
- User accounts and permissions
- Catalogs and apps
- Monitoring and logging settings

**Creating Backups**:

**On-Demand Backup**:
1. **Rancher Backups > Backups**
2. **Create**: Initiate backup
3. **Configuration**:
   - Backup name
   - Storage location
   - Encryption
4. **Create**: Backup executes

**Scheduled Backups**:
```yaml
apiVersion: resources.cattle.io/v1
kind: Backup
metadata:
  name: daily-backup
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  retentionCount: 30
  storageLocation:
    s3:
      bucketName: rancher-backups
      endpoint: s3.amazonaws.com
      region: us-west-2
      folder: backups
      credentialSecretNamespace: default
      credentialSecretName: s3-creds
  encryptionConfigSecretName: backup-encryption
```

**Storage Locations**:
- **S3**: AWS S3 or compatible
- **Minio**: Self-hosted S3
- **PersistentVolume**: Kubernetes storage
- **NFS**: Network file system

### Restoring Rancher

**Restore Process**:
1. **Install Fresh Rancher**: Same version as backup
2. **Install Backup Operator**: Same configuration
3. **Create Restore Resource**:
   ```yaml
   apiVersion: resources.cattle.io/v1
   kind: Restore
   metadata:
     name: restore-latest
   spec:
     backupFilename: rancher-backup-20260303.tar.gz
     storageLocation:
       s3:
         # Same configuration as backup
     encryptionConfigSecretName: backup-encryption
   ```
4. **Apply Restore**: `kubectl apply -f restore.yaml`
5. **Wait for Completion**: Monitor restore status
6. **Verify**: Check Rancher UI and cluster access

**Disaster Recovery Scenarios**:

**Complete Cluster Loss**:
1. Provision new management cluster
2. Install Rancher (same version)
3. Install backup operator
4. Restore from backup
5. Downstream clusters reconnect automatically

**Rancher Pod Failure**:
- Automatic recovery via Deployment
- No backup restore needed
- Persistent data intact

**etcd Corruption**:
- Restore etcd from snapshot (Kubernetes level)
- Or restore Rancher from backup

### Downstream Cluster Backups

**etcd Snapshots** (RKE/RKE2):

**Automatic Snapshots**:
- Enabled by default
- Every 6 hours (configurable)
- Retention: 5 snapshots (configurable)
- Stored on etcd nodes

**Snapshot Configuration**:
```yaml
services:
  etcd:
    backup_config:
      enabled: true
      interval_hours: 6
      retention: 5
      s3_backup_config:
        access_key: AWS_ACCESS_KEY
        secret_key: AWS_SECRET_KEY
        bucket_name: etcd-backups
        endpoint: s3.amazonaws.com
        region: us-west-2
```

**On-Demand Snapshot**:
1. **Cluster > Snapshots**
2. **Take Snapshot**: Create snapshot
3. **Name**: Provide descriptive name
4. **Create**: Snapshot saved

**Restoring from Snapshot**:
1. **Snapshots Tab**: View available snapshots
2. **Select Snapshot**: Choose restore point
3. **Restore**: Initiate restoration
4. **Cluster Reboots**: Control plane components restart
5. **Verify**: Check cluster state

**Snapshot Best Practices**:
- Regular snapshots before upgrades
- Off-node snapshot storage (S3)
- Encrypted snapshots
- Test restore procedures
- Document recovery RTO/RPO
- Multiple retention periods

### Application Backup (Velero)

**Velero Integration**:
- Backup and restore Kubernetes resources
- Disaster recovery
- Cluster migration
- Namespace backup

**Installation**:
1. **Install Velero CLI**: On management machine
2. **Deploy Velero**: In cluster
   ```bash
   velero install \
     --provider aws \
     --bucket velero-backups \
     --secret-file ./credentials-velero \
     --backup-location-config region=us-west-2 \
     --snapshot-location-config region=us-west-2 \
     --use-volume-snapshots=true
   ```

**Creating Backups**:
```bash
# Backup entire namespace
velero backup create my-backup --include-namespaces myapp

# Backup with label selector
velero backup create app-backup --selector app=myapp

# Schedule regular backups
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces production
```

**Restoring**:
```bash
# Restore backup
velero restore create --from-backup my-backup

# Restore to different namespace
velero restore create --from-backup my-backup \
  --namespace-mappings old-ns:new-ns
```

### Backup Best Practices

**Backup Strategy**:
- **3-2-1 Rule**: 3 copies, 2 different media, 1 off-site
- Regular automated backups
- Test restores regularly (monthly)
- Document procedures
- Encrypt sensitive backups
- Monitor backup success/failure

**Retention Policies**:
- Daily backups: 7 days
- Weekly backups: 4 weeks
- Monthly backups: 12 months
- Critical clusters: More frequent

**Testing**:
- Regular restore drills
- Separate test environment
- Verify application functionality
- Measure recovery time
- Update runbooks

**Monitoring**:
- Alert on backup failures
- Track backup size and duration
- Storage capacity monitoring
- Restore time metrics
- Compliance reporting

## Best Practices

### Cluster Management

**Standardization**:
- Use cluster templates
- Consistent Kubernetes versions
- Standard CNI plugin
- Common add-ons across clusters
- Naming conventions

**Lifecycle Management**:
- Regular upgrade schedule
- Test upgrades in non-production
- Snapshot before upgrades
- Phased rollout to production
- Document rollback procedures

**Capacity Planning**:
- Right-size node pools
- Plan for growth
- Monitor resource utilization
- Auto-scaling where appropriate
- Cost optimization

### Multi-Cluster Strategy

**Cluster Segmentation**:
- Separate production from non-production
- Environment-based clusters (dev, staging, prod)
- Region-based clusters
- Team/application-based clusters

**Management Cluster**:
- Dedicated management cluster for Rancher
- High availability configuration
- Regular backups
- Restricted access
- Monitoring and alerting

**Fleet Organization**:
- Logical cluster grouping
- Consistent labeling strategy
- GitOps for all deployments
- Staged rollouts
- Drift monitoring

### Security Best Practices

**Access Control**:
- Use external authentication (AD, OIDC)
- Group-based permissions
- Regular access reviews
- MFA for administrators
- API key rotation

**Network Security**:
- Network policies enabled
- Project network isolation
- Ingress security (WAF, rate limiting)
- Private cluster endpoints where possible
- Certificate management (cert-manager)

**Workload Security**:
- Pod Security Standards enforcement
- Non-root containers
- Read-only root filesystems
- Resource limits
- Image scanning

**Compliance**:
- Regular CIS scans
- Audit logging enabled
- Compliance reporting
- Policy as code (OPA, Kyverno)
- Security training

### Operational Excellence

**Monitoring and Alerting**:
- Enable monitoring on all clusters
- Custom dashboards for applications
- Alert fatigue prevention
- On-call rotations
- Incident response procedures

**Logging**:
- Centralized logging
- Log retention policies
- Sensitive data filtering
- Log analysis and alerting
- Cost optimization

**Automation**:
- GitOps for deployments
- Infrastructure as Code
- Automated testing
- CI/CD pipelines
- Self-service for developers

**Documentation**:
- Architecture diagrams
- Runbooks for common tasks
- Troubleshooting guides
- Disaster recovery procedures
- Onboarding documentation

### Cost Optimization

**Resource Management**:
- Right-size workloads
- Resource quotas and limits
- Vertical Pod Autoscaler
- Cluster autoscaling
- Node deprovisioning

**Storage Optimization**:
- Volume cleanup
- Appropriate storage classes
- Snapshot management
- Data lifecycle policies

**Cluster Consolidation**:
- Multi-tenancy via projects
- Consolidate small clusters
- Edge cluster optimization (K3s)
- Spot instances for non-critical workloads

### Performance Optimization

**Cluster Performance**:
- SSD for etcd
- Adequate control plane resources
- Network bandwidth planning
- Separate etcd from control plane (large clusters)

**Application Performance**:
- Resource requests and limits
- Horizontal Pod Autoscaling
- Pod anti-affinity for HA
- PodDisruptionBudgets
- Readiness and liveness probes

**Monitoring Stack**:
- Adequate resources for Prometheus
- Metric retention tuning
- Cardinality management
- Log sampling for high volume

### Disaster Recovery

**Planning**:
- Define RTO and RPO
- Regular backup testing
- Multiple backup locations
- Cross-region backups
- Documented procedures

**Testing**:
- Quarterly DR drills
- Measure recovery times
- Update runbooks
- Team training
- Improve processes

### Migration and Adoption

**Phased Approach**:
- Start with non-critical workloads
- Pilot projects
- Team training
- Gradual expansion
- Lessons learned documentation

**Change Management**:
- Clear communication
- Training programs
- Documentation
- Support channels
- Feedback loops

## Troubleshooting

### Common Issues

**Cluster Connectivity**:
- Check cluster agent status
- Verify network connectivity
- Review firewall rules
- Check DNS resolution
- Validate certificates

**Authentication Problems**:
- Verify authentication provider config
- Check user/group sync
- Validate credentials
- Review RBAC bindings
- Check audit logs

**Application Deployment Failures**:
- Review pod events
- Check resource quotas
- Validate image pull
- Review NetworkPolicies
- Check PodSecurityPolicies

**Performance Issues**:
- Check resource utilization
- Review etcd performance
- Network latency
- Storage I/O
- Control plane metrics

### Getting Help

**Documentation**:
- Rancher docs: https://rancher.com/docs
- Community forums
- GitHub issues
- Slack channel

**Support**:
- SUSE support (enterprise)
- Community support
- Training and certification
- Professional services
