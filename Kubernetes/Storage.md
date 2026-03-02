# Kubernetes PV and PVC Cheatsheet

## Persistent Volume (PV) and Persistent Volume Claim (PVC) Basics

### List Resources
```bash
kubectl get pv
kubectl get pvc -n <namespace>
kubectl get pvc --all-namespaces
```

### Describe Resources
```bash
kubectl describe pv <pv-name>
kubectl describe pvc <pvc-name> -n <namespace>
```

### Check PV/PVC Status
```bash
kubectl get pv <pv-name> -o yaml
kubectl get pvc <pvc-name> -n <namespace> -o jsonpath='{.status.phase}'
```

## PVC States
- **Pending**: PVC is waiting for a PV to bind
- **Bound**: PVC is bound to a PV
- **Lost**: PV associated with PVC no longer exists

## Troubleshooting

### Check PVC Events
```bash
kubectl describe pvc <pvc-name> -n <namespace>
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

### Check Storage Class
```bash
kubectl get sc
kubectl describe sc <storage-class-name>
```

### Check PV Binding
```bash
kubectl get pv -o custom-columns=NAME:.metadata.name,CLAIM:.spec.claimRef.name,STATUS:.status.phase
```

### Force Delete Stuck PVC
```bash
kubectl patch pvc <pvc-name> -n <namespace> -p '{"metadata":{"finalizers":null}}'
kubectl delete pvc <pvc-name> -n <namespace> --force --grace-period=0
```

## Managing Finalizers

### View Finalizers
```bash
kubectl get pv <pv-name> -o jsonpath='{.metadata.finalizers}'
kubectl get pvc <pvc-name> -n <namespace> -o jsonpath='{.metadata.finalizers}'
```

### Common Finalizers
- `kubernetes.io/pv-protection`: Prevents deletion of PV in use
- `kubernetes.io/pvc-protection`: Prevents deletion of PVC in use

### Remove Finalizers (Use with Caution)
```bash
# Remove PVC finalizers
kubectl patch pvc <pvc-name> -n <namespace> -p '{"metadata":{"finalizers":[]}}' --type=merge

# Remove PV finalizers
kubectl patch pv <pv-name> -p '{"metadata":{"finalizers":[]}}' --type=merge
```

### Edit Finalizers Manually
```bash
kubectl edit pvc <pvc-name> -n <namespace>
kubectl edit pv <pv-name>
```

## HostPath Volumes

### HostPath PV Example
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
    name: hostpath-pv
spec:
    capacity:
        storage: 10Gi
    accessModes:
        - ReadWriteOnce
    hostPath:
        path: /data/volumes/pv1
        type: DirectoryOrCreate
    persistentVolumeReclaimPolicy: Retain
```

### HostPath Types
- `DirectoryOrCreate`: Create directory if missing
- `Directory`: Directory must exist
- `FileOrCreate`: Create file if missing
- `File`: File must exist
- `Socket`: UNIX socket must exist
- `CharDevice`: Character device must exist
- `BlockDevice`: Block device must exist

### HostPath PVC Example
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: hostpath-pvc
spec:
    accessModes:
        - ReadWriteOnce
    resources:
        requests:
            storage: 10Gi
```

### HostPath Troubleshooting
```bash
# Verify node has the directory
kubectl get pv <pv-name> -o jsonpath='{.spec.hostPath.path}'

# Check which node the pod is scheduled on
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.nodeName}'

# Verify permissions on the node
kubectl debug node/<node-name> -it --image=busybox
ls -la /host/data/volumes/pv1
```

### HostPath Best Practices
- ⚠️ **Not recommended for production** (node-specific, no portability)
- Useful for single-node testing and development
- Ensure proper permissions on the host directory
- Use `DirectoryOrCreate` for automatic directory creation

## Reclaim Policies
- **Retain**: Manual reclamation required
- **Delete**: Automatic deletion of volume
- **Recycle**: Basic scrub (deprecated)

### Change Reclaim Policy
```bash
kubectl patch pv <pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```
