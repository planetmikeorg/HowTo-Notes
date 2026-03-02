# Kubernetes Affinity Troubleshooting Guide

## Overview
Affinity rules control pod placement on nodes in a Kubernetes cluster. This guide covers common troubleshooting scenarios for node affinity, pod affinity, and pod anti-affinity.

## Common Issues

### 1. Pod Not Being Deployed (Pending State)

#### Symptoms
- Pod remains in `Pending` state
- No node is selected for the pod
- Events show scheduling failures

#### Troubleshooting Steps

##### Check Pod Events
```bash
kubectl describe pod <pod-name> -n <namespace>
```

Look for events like:
- `0/N nodes are available: N node(s) didn't match Pod's node affinity/selector`
- `0/N nodes are available: N node(s) had taint {key: value}, that the pod didn't tolerate`

##### Verify Node Labels
```bash
# List all nodes with labels
kubectl get nodes --show-labels

# Check specific node labels
kubectl describe node <node-name>
```

Ensure the nodes have the labels that match your affinity rules.

##### Verify RequiredDuringSchedulingIgnoredDuringExecution Rules
If using `requiredDuringSchedulingIgnoredDuringExecution`, the pod **must** find a matching node:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disktype
          operator: In
          values:
          - ssd
```

**Common mistakes:**
- Label key typos (case-sensitive)
- Wrong operator (In, NotIn, Exists, DoesNotExist, Gt, Lt)
- No nodes have the required labels
- Values don't match (case-sensitive)

##### Check Pod Affinity/Anti-Affinity Conflicts
```bash
# Check pods running on nodes
kubectl get pods -o wide --all-namespaces

# Check specific pod's affinity rules
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A 20 affinity
```

**Conflicts to look for:**
- Required pod anti-affinity rules that exclude all available nodes
- Topology constraints that cannot be satisfied
- Conflicting affinity rules (both node and pod affinity)

##### Verify Topology Keys
For pod affinity/anti-affinity, ensure topology keys exist on nodes:

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: web
      topologyKey: kubernetes.io/hostname  # Must exist on nodes
```

Common topology keys:
- `kubernetes.io/hostname` - per node
- `topology.kubernetes.io/zone` - per availability zone
- `topology.kubernetes.io/region` - per region

##### Check Resource Availability
Even with matching affinity, pods need resources:

```bash
# Check node capacity and allocatable resources
kubectl describe nodes

# Check resource requests in pod spec
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A 5 resources
```

##### Verify Taints and Tolerations
Affinity rules work alongside taints:

```bash
# Check node taints
kubectl describe node <node-name> | grep Taints

# Check pod tolerations
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A 5 tolerations
```

---

### 2. Pod Deployed to Same Node Despite preferredDuringSchedulingIgnoredDuringExecution

#### Symptoms
- Multiple pods scheduled on the same node
- `preferredDuringSchedulingIgnoredDuringExecution` appears to be ignored
- Uneven pod distribution across nodes

#### Understanding Preferred vs Required
- **Required**: Hard constraint - pod won't schedule if not met
- **Preferred**: Soft constraint - scheduler tries to honor but may ignore

#### Troubleshooting Steps

##### Check Scheduler Scoring
The scheduler uses a scoring algorithm with weights:

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100  # Weight: 1-100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: web
        topologyKey: kubernetes.io/hostname
```

**Key points:**
- Weight range: 1-100 (higher = more important)
- Scheduler combines weights from all preferences
- Other factors (resources, priorities) also affect scoring

##### Verify Node Scores
Enable scheduler verbose logging to see scoring:

```bash
# Check scheduler logs
kubectl logs -n kube-system <scheduler-pod-name>

# Look for scoring details
kubectl logs -n kube-system <scheduler-pod-name> | grep -i score
```

##### Check Competing Factors

**Resource pressure:**
```bash
# Check node resource utilization
kubectl top nodes

# Check if preferred nodes are full
kubectl describe node <preferred-node-name>
```

If preferred nodes are resource-constrained, scheduler may override preference.

**Pod priority:**
```bash
# Check pod priority class
kubectl get pod <pod-name> -n <namespace> -o yaml | grep priorityClassName

# List priority classes
kubectl get priorityclasses
```

High-priority pods may override affinity preferences.

**Node conditions:**
```bash
# Check node conditions
kubectl describe node <node-name> | grep -A 10 Conditions
```

Nodes with `MemoryPressure`, `DiskPressure`, or `PIDPressure` get lower scores.

##### Verify Multiple Preferences
If you have multiple preferred rules, they're scored independently:

```yaml
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 80
      preference:
        matchExpressions:
        - key: disktype
          operator: In
          values:
          - ssd
    - weight: 20
      preference:
        matchExpressions:
        - key: zone
          operator: In
          values:
          - us-west-1a
```

The scheduler sums weighted scores for each node.

##### Check Pod Topology Spread Constraints
Topology spread constraints may conflict with affinity preferences:

```bash
# Check if topology spread is configured
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A 10 topologySpreadConstraints
```

These constraints can override preferred affinity rules.

##### Examine Scheduler Configuration
Check if custom scheduler configuration affects behavior:

```bash
# Get scheduler config
kubectl get configmap -n kube-system kube-scheduler -o yaml

# Check scheduler profile
kubectl get pod -n kube-system <scheduler-pod> -o yaml | grep -A 5 KubeSchedulerConfiguration
```

##### Test with Required Rules
To verify affinity rules work, temporarily switch to `required`:

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:  # Changed from preferred
    - labelSelector:
        matchLabels:
          app: web
      topologyKey: kubernetes.io/hostname
```

If this spreads pods correctly, your original preferred rules are being overridden by other factors.

---

## Diagnostic Commands Cheat Sheet

```bash
# Pod scheduling details
kubectl describe pod <pod-name> -n <namespace>
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Node information
kubectl get nodes --show-labels
kubectl describe node <node-name>
kubectl top nodes

# Pod distribution
kubectl get pods -o wide -n <namespace>
kubectl get pods -o wide --all-namespaces | grep <app-label>

# Affinity rules
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A 30 affinity

# Scheduler logs
kubectl logs -n kube-system $(kubectl get pods -n kube-system -l component=kube-scheduler -o name | head -1) --tail=100

# Debug scheduling
kubectl get events --field-selector involvedObject.name=<pod-name> -n <namespace>
```

---

## Best Practices

1. **Start with Preferred Rules**: Use `preferredDuringSchedulingIgnoredDuringExecution` first, upgrade to required only if necessary

2. **Use Appropriate Weights**: Assign higher weights (80-100) to critical preferences, lower weights (1-20) to nice-to-have preferences

3. **Verify Labels Exist**: Always verify node labels exist before deploying affinity rules
   ```bash
   kubectl get nodes -l <label-key>=<label-value>
   ```

4. **Combine with Topology Spread**: Use `topologySpreadConstraints` for even distribution alongside affinity rules

5. **Monitor Node Resources**: Preferred affinity can be overridden by resource constraints

6. **Test Incrementally**: Deploy one replica first to verify affinity rules work before scaling

7. **Use Anti-Affinity for HA**: Spread critical application pods across nodes/zones for high availability

8. **Document Custom Labels**: Maintain documentation of custom node labels used in affinity rules

---

## Advanced Debugging

### Enable Scheduler Debug Logging

Edit the scheduler manifest (static pod):
```bash
kubectl edit pod kube-scheduler-<node-name> -n kube-system
```

Add `--v=5` or `--v=10` for more verbose logging.

### Simulate Scheduling

Use the `kubectl-debug-scheduler` plugin or examine scheduler extender logs if custom schedulers are in use.

### Check Webhook Denials

Admission webhooks can block pod scheduling:
```bash
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations
```

### Identify PreBind Plugin Failures

PreBind plugins run after a node is selected but before the pod is bound. Failures here cause scheduling to fail even when affinity rules are satisfied.

#### Check Pod Events for PreBind Failures
```bash
# Get detailed pod events
kubectl describe pod <pod-name> -n <namespace>

# Filter for PreBind-related events
kubectl get events -n <namespace> --field-selector involvedObject.name=<pod-name> | grep -i prebind
```

Look for events like:
- `running PreBind plugin "PluginName": <error details>`
- `PreBind plugin rejected pod`

#### Get Scheduler Logs with Plugin Details
```bash
# Get scheduler pod name
SCHEDULER_POD=$(kubectl get pods -n kube-system -l component=kube-scheduler -o jsonpath='{.items[0].metadata.name}')

# View recent logs with plugin information
kubectl logs -n kube-system $SCHEDULER_POD --tail=200 | grep -A 10 -i prebind

# Filter for specific pod
kubectl logs -n kube-system $SCHEDULER_POD | grep "<pod-name>" | grep -i prebind
```

#### Enable Verbose Scheduler Logging
For detailed plugin execution traces:
```bash
# Edit scheduler to add verbosity flag
kubectl edit pod kube-scheduler-<node-name> -n kube-system

# Add or modify the --v flag in the command section:
# --v=4  : Basic plugin execution info
# --v=5  : Detailed plugin decisions
# --v=10 : Full debug output including plugin state
```

After saving, the scheduler pod will restart automatically.

#### Check Common PreBind Plugin Issues

**Volume attachment failures:**
```bash
# Check VolumeAttachment objects
kubectl get volumeattachments

# Check for errors in CSI driver
kubectl logs -n kube-system -l app=csi-driver --tail=100
```

**Network plugin issues:**
```bash
# Check CNI plugin logs
kubectl logs -n kube-system -l k8s-app=kube-proxy
kubectl logs -n kube-system -l app=calico-node  # or your CNI
```

**Resource quota exceeded:**
```bash
# Check resource quotas in namespace
kubectl describe resourcequota -n <namespace>

# Check limit ranges
kubectl describe limitrange -n <namespace>
```

#### Identify Which Plugin Failed
In scheduler logs, look for the plugin name:
```bash
kubectl logs -n kube-system $SCHEDULER_POD | grep "PreBind plugin" | grep -i error
```

Common PreBind plugins:
- `VolumeBinding` - Volume attachment issues
- `NodePorts` - Port allocation conflicts
- `PodTopologySpread` - Topology constraint violations
- Custom admission plugins

#### Get Detailed Failure Information
```bash
# Full event details in JSON
kubectl get events -n <namespace> -o json | jq '.items[] | select(.involvedObject.name=="<pod-name>") | select(.message | contains("PreBind"))'

# Scheduler decision trace
kubectl logs -n kube-system $SCHEDULER_POD --tail=500 | grep -B 5 -A 10 "<pod-name>"
```

#### Debug Volume Binding PreBind Failures
```bash
# Check PVC status
kubectl get pvc -n <namespace>
kubectl describe pvc <pvc-name> -n <namespace>

# Check storage class
kubectl get storageclass
kubectl describe storageclass <storage-class-name>

# Check persistent volumes
kubectl get pv
kubectl describe pv <pv-name>
```

---

## Common Patterns and Solutions

### Pattern: High Availability with Anti-Affinity
```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: critical-service
      topologyKey: kubernetes.io/hostname
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: critical-service
        topologyKey: topology.kubernetes.io/zone
```

This ensures pods never share a node (required) and prefer different zones (preferred).

### Pattern: Workload Locality
```yaml
affinity:
  podAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: cache
        topologyKey: kubernetes.io/hostname
```

Schedules pods near cache pods for reduced latency.

### Pattern: GPU/Special Hardware
```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: accelerator
          operator: In
          values:
          - nvidia-tesla-v100
```

Ensures pods land on nodes with specific hardware.