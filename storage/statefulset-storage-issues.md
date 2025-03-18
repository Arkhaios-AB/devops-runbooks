# Troubleshooting StatefulSet Storage Issues

## Symptoms

* StatefulSet pods stuck in `Pending` state due to PVC issues
* PVCs not being created for StatefulSet pods
* StatefulSet pods unable to mount volumes
* Data persistence issues between pod restarts
* Volumes not being retained when pods are rescheduled
* StatefulSet scale down operations failing
* Orphaned PVCs after StatefulSet deletion
* Inconsistent volume attachment across StatefulSet pods
* StatefulSet pods starting without waiting for volume attachment
* Volume claim templates not creating expected PVCs

## Possible Causes

1. **StorageClass Issues**: Problems with the StorageClass used by the StatefulSet
2. **Volume Binding Mode**: Inappropriate volume binding mode
3. **PVC Template Misconfiguration**: Errors in the volumeClaimTemplates section
4. **Storage Provisioner Problems**: Issues with the storage provisioner
5. **Zone/Topology Constraints**: Pod and volume in different availability zones
6. **PV Reclaim Policy**: Incorrect reclaim policy causing unexpected behavior
7. **Volume Expansion Issues**: Problems expanding volumes for growing StatefulSets
8. **Finalizer Problems**: Finalizers preventing proper cleanup
9. **VolumeSnapshot Issues**: Problems with volume snapshots for StatefulSet data
10. **Orphaned Resources**: Orphaned PVs/PVCs from previous StatefulSet instances

## Diagnosis Steps

### 1. Check StatefulSet Configuration

```bash
# Get StatefulSet details
kubectl get statefulset <statefulset-name> -n <namespace>

# Check StatefulSet configuration
kubectl describe statefulset <statefulset-name> -n <namespace>

# Examine the volumeClaimTemplates section
kubectl get statefulset <statefulset-name> -n <namespace> -o jsonpath='{.spec.volumeClaimTemplates}'
```

### 2. Check PVC Status

```bash
# List PVCs created by the StatefulSet
kubectl get pvc -n <namespace> | grep <statefulset-name>

# Check details of a specific PVC
kubectl describe pvc <pvc-name> -n <namespace>
```

### 3. Check PV Status

```bash
# List PVs bound to the StatefulSet's PVCs
kubectl get pv | grep <pvc-name>

# Check details of a specific PV
kubectl describe pv <pv-name>
```

### 4. Check StorageClass

```bash
# Get information about the StorageClass
kubectl get storageclass <storage-class-name>

# Check StorageClass details
kubectl describe storageclass <storage-class-name>
```

### 5. Check Pod Status and Events

```bash
# List StatefulSet pods and their status
kubectl get pods -n <namespace> -l app=<statefulset-label>

# Check events related to pod issues
kubectl get events -n <namespace> | grep <pod-name>
```

### 6. Check Storage Provisioner

```bash
# Check storage provisioner pods
kubectl get pods -n kube-system | grep -E 'csi|provisioner'

# Check provisioner logs
kubectl logs -n kube-system <provisioner-pod-name>
```

## Resolution Steps

### 1. Fix StorageClass Issues

If the StorageClass is problematic:

```bash
# Create or update the StorageClass
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: <storage-class-name>
provisioner: <provisioner>
parameters:
  type: gp2  # Example for AWS EBS
  fsType: ext4
volumeBindingMode: WaitForFirstConsumer  # Recommended for StatefulSets
allowVolumeExpansion: true
reclaimPolicy: Retain  # Use 'Retain' for important data
EOF

# Update the StatefulSet to use the new StorageClass
kubectl patch statefulset <statefulset-name> -n <namespace> --type=json \
  -p='[{"op": "replace", "path": "/spec/volumeClaimTemplates/0/spec/storageClassName", "value":"<new-storage-class-name>"}]'
```

### 2. Fix PVC Template Configuration

If volumeClaimTemplates has issues:

```bash
# Edit the StatefulSet to fix volumeClaimTemplates
kubectl edit statefulset <statefulset-name> -n <namespace>
```

Example of correct volumeClaimTemplates:
```yaml
volumeClaimTemplates:
- metadata:
    name: data  # This becomes part of the PVC name (<name>-<statefulset-name>-<ordinal>)
  spec:
    accessModes: ["ReadWriteOnce"]
    storageClassName: <storage-class-name>
    resources:
      requests:
        storage: 10Gi
```

### 3. Fix Zone/Topology Issues

To ensure pods and volumes are in the same zone:

```bash
# Create StorageClass with WaitForFirstConsumer binding mode
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: <storage-class-name>
provisioner: <provisioner>
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp2  # Example for AWS EBS
EOF

# Configure pod affinity based on zone
kubectl edit statefulset <statefulset-name> -n <namespace>
```

Add node affinity to the StatefulSet spec:
```yaml
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                - <zone-1>
                - <zone-2>
```

### 4. Fix PV Reclaim Policy

To prevent data loss when PVCs are deleted:

```bash
# Update the StorageClass to use 'Retain' reclaim policy
kubectl patch storageclass <storage-class-name> \
  -p '{"reclaimPolicy": "Retain"}'

# For existing PVs, update reclaim policy manually
kubectl patch pv <pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

### 5. Handle Orphaned PVCs

If orphaned PVCs exist after StatefulSet deletion:

```bash
# List orphaned PVCs
kubectl get pvc -n <namespace> | grep <statefulset-name>

# Manually delete orphaned PVCs if they're no longer needed
kubectl delete pvc <pvc-name> -n <namespace>

# To preserve data, create a backup before deleting
# Example for AWS EBS volumes
PV_NAME=$(kubectl get pvc <pvc-name> -n <namespace> -o jsonpath='{.spec.volumeName}')
VOLUME_ID=$(kubectl get pv $PV_NAME -o jsonpath='{.spec.awsElasticBlockStore.volumeID}' | cut -d'/' -f4)
aws ec2 create-snapshot --volume-id $VOLUME_ID --description "Backup of StatefulSet PV before deletion"
```

### 6. Fix Finalizer Issues

If PVCs are stuck in terminating state due to finalizers:

```bash
# Check for finalizers
kubectl get pvc <pvc-name> -n <namespace> -o jsonpath='{.metadata.finalizers}'

# Remove finalizers
kubectl patch pvc <pvc-name> -n <namespace> --type=json \
  -p='[{"op": "remove", "path": "/metadata.finalizers"}]'
```

### 7. Fix Volume Expansion Issues

If volumes need to be expanded:

```bash
# Ensure StorageClass has allowVolumeExpansion: true
kubectl patch storageclass <storage-class-name> \
  -p '{"allowVolumeExpansion": true}'

# Edit PVC to request more storage
kubectl edit pvc <pvc-name> -n <namespace>
```

Update the storage request:
```yaml
spec:
  resources:
    requests:
      storage: 20Gi  # Increase from previous value
```

### 8. Configure Parallel Pod Management

For faster StatefulSet operations without affecting storage:

```bash
# Update StatefulSet to use parallel pod management
kubectl patch statefulset <statefulset-name> -n <namespace> --type=json \
  -p='[{"op": "replace", "path": "/spec/podManagementPolicy", "value":"Parallel"}]'
```

Note: This doesn't affect storage ordering but allows pods to start in parallel.

### 9. Implement Orderly Scale Down

For StatefulSets with persistent storage, implement proper scale down:

```bash
# Scale down one pod at a time and verify data migration
kubectl scale statefulset <statefulset-name> -n <namespace> --replicas=$(($(kubectl get statefulset <statefulset-name> -n <namespace> -o jsonpath='{.spec.replicas}')-1))

# Wait for clean shutdown and data migration before further scale down
```

### 10. Recreate StatefulSet Preserving PVCs

In extreme cases, recreate the StatefulSet while preserving PVCs:

```bash
# Export current StatefulSet without cluster-specific fields
kubectl get statefulset <statefulset-name> -n <namespace> -o yaml > statefulset.yaml

# Edit the YAML to remove status and other managed fields
# Then delete the StatefulSet without deleting PVCs
kubectl delete statefulset <statefulset-name> -n <namespace> --cascade=false

# Apply the edited StatefulSet
kubectl apply -f statefulset.yaml
```

## Prevention

1. **Use WaitForFirstConsumer**: Configure StorageClasses with volumeBindingMode: WaitForFirstConsumer
2. **Backup Strategy**: Implement regular backups of StatefulSet data
3. **Testing**: Test StatefulSet storage operations in non-production environments
4. **Monitoring**: Monitor PV/PVC creation and binding for StatefulSets
5. **Resource Planning**: Plan for storage capacity needs in advance
6. **Documentation**: Document StatefulSet storage configuration and behavior
7. **Volume Snapshots**: Use VolumeSnapshots for data protection
8. **Retain Reclaim Policy**: Use "Retain" reclaimPolicy for critical data
9. **Maintenance Planning**: Plan maintenance operations carefully for StatefulSets
10. **Upgrade Testing**: Test StatefulSet behavior during Kubernetes upgrades

## Related Runbooks

* [PVC Stuck in Pending](./pvc-pending.md)
* [Storage Class Issues](./storage-class-issues.md)
* [Volume Mount Problems](./volume-mount-problems.md)
* [CSI Driver Issues](./csi-driver-issues.md)
* [StatefulSet Issues](../workloads/statefulset-issues.md)