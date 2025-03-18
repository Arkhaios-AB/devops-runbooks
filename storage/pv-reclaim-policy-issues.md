# Troubleshooting PV Reclaim Policy Issues

## Symptoms

* Unexpected PV deletion when PVC is deleted
* PVs remain in Released state and don't get reused
* Storage resources not being reclaimed after PVC deletion
* Data loss after workload deletion or migration
* Orphaned PVs consuming storage resources
* PVs stuck in Failed state after reclaim attempt
* Unable to reuse existing storage for new PVCs
* PVs remaining when they should be deleted
* Storage space exhaustion due to unreclaimed PVs
* Billing for unused storage resources

## Possible Causes

1. **Incorrect Reclaim Policy**: PV or StorageClass configured with wrong reclaim policy
2. **Storage Provisioner Issues**: Problems with storage provisioner handling reclamation
3. **Finalizer Problems**: Finalizers preventing proper PV reclamation
4. **Manual PV Manipulation**: Direct modification of PVs outside Kubernetes
5. **Storage Backend Issues**: Storage backend failing to delete or recycle volumes
6. **Permission Problems**: Insufficient permissions to delete storage in backend
7. **Kubernetes Version Issues**: Behavior changes between Kubernetes versions
8. **PV Protection Conflicts**: PV protection preventing reclamation
9. **StatefulSet PV Management**: Issues with StatefulSet PV handling
10. **Cloud Provider API Limits**: Throttling or errors from cloud provider API

## Diagnosis Steps

### 1. Check PV Reclaim Policies

```bash
# List all PVs with their reclaim policies
kubectl get pv -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,RECLAIM:.spec.persistentVolumeReclaimPolicy

# Check the reclaim policy of a specific PV
kubectl get pv <pv-name> -o jsonpath='{.spec.persistentVolumeReclaimPolicy}'
```

### 2. Check StorageClass Default Reclaim Policy

```bash
# List StorageClasses and their reclaim policies
kubectl get storageclass -o custom-columns=NAME:.metadata.name,RECLAIM:.reclaimPolicy

# Check specific StorageClass reclaim policy
kubectl get storageclass <storage-class-name> -o jsonpath='{.reclaimPolicy}'
```

### 3. Check PV Status and Events

```bash
# Get detailed PV information
kubectl describe pv <pv-name>

# Check events related to the PV
kubectl get events | grep <pv-name>
```

### 4. Check Storage Provisioner Logs

```bash
# Identify and check provisioner pods
kubectl get pods -n kube-system -l app=<provisioner-label>

# Check provisioner logs
kubectl logs -n kube-system <provisioner-pod-name>
```

### 5. Check for Finalizers

```bash
# Check for finalizers on the PV
kubectl get pv <pv-name> -o jsonpath='{.metadata.finalizers}'
```

## Resolution Steps

### 1. Fix Incorrect Reclaim Policy on PV

If individual PV has the wrong reclaim policy:

```bash
# Update PV reclaim policy (e.g., from Delete to Retain)
kubectl patch pv <pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'

# Or from Retain to Delete (be cautious - data will be deleted)
kubectl patch pv <pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Delete"}}'
```

### 2. Fix StorageClass Default Reclaim Policy

To change the default for new PVs:

```bash
# Update StorageClass reclaim policy
kubectl patch storageclass <storage-class-name> -p '{"reclaimPolicy": "Retain"}'
```

Create a new StorageClass with desired policy:
```bash
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: <storage-class-name>
provisioner: <provisioner>
parameters:
  # Provisioner-specific parameters
  type: gp2  # Example for AWS EBS
reclaimPolicy: Retain  # Or Delete or Recycle (deprecated)
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
EOF
```

### 3. Fix PVs Stuck in Released State

If PVs are stuck in Released state with Delete policy:

```bash
# Check PV status
kubectl get pv <pv-name>

# Remove finalizers from PV
kubectl patch pv <pv-name> --type=json -p='[{"op": "remove", "path": "/metadata.finalizers"}]'
```

For cloud providers, manually verify and delete the backend resource if needed:

AWS:
```bash
# Get EBS volume ID
VOLUME_ID=$(kubectl get pv <pv-name> -o jsonpath='{.spec.awsElasticBlockStore.volumeID}' | cut -d'/' -f4)
# Check volume status
aws ec2 describe-volumes --volume-ids $VOLUME_ID
# Manually delete if stuck
aws ec2 delete-volume --volume-id $VOLUME_ID
```

GCP:
```bash
# Get PD name
DISK_NAME=$(kubectl get pv <pv-name> -o jsonpath='{.spec.gcePersistentDisk.pdName}')
# Delete the disk
gcloud compute disks delete $DISK_NAME --zone=<zone>
```

Azure:
```bash
# Get disk details
DISK_URI=$(kubectl get pv <pv-name> -o jsonpath='{.spec.azureDisk.diskURI}')
# Delete the disk
az disk delete --ids $DISK_URI --yes
```

### 4. Handle Orphaned PVs

For PVs that should be deleted but remain:

```bash
# For PVs with Delete policy that are still around:
kubectl delete pv <pv-name>

# For PVs with Retain policy that are no longer needed:
kubectl delete pv <pv-name>
```

Then clean up backend storage if needed using cloud provider tools.

### 5. Manually Reclaim PVs with Retain Policy

For Released PVs with Retain policy that you want to reuse:

```bash
# Edit the PV to clear claimRef
kubectl edit pv <pv-name>
```

Remove or clear the `claimRef` section:
```yaml
spec:
  # Remove this entire claimRef section to make the PV Available
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: example-pvc
    namespace: default
    resourceVersion: "123456"
    uid: abcdef-1234-5678-90ab-cdef12345678
```

### 6. Set Up Backup Before Deletion

Before allowing Delete policy to take effect:

```bash
# For AWS EBS:
VOLUME_ID=$(kubectl get pv <pv-name> -o jsonpath='{.spec.awsElasticBlockStore.volumeID}' | cut -d'/' -f4)
aws ec2 create-snapshot --volume-id $VOLUME_ID --description "Backup before PV deletion"

# For GCP PD:
DISK_NAME=$(kubectl get pv <pv-name> -o jsonpath='{.spec.gcePersistentDisk.pdName}')
gcloud compute disks snapshot $DISK_NAME --zone=<zone> --snapshot-names=$DISK_NAME-snapshot
```

### 7. Fix Permissions for Storage Deletion

If permissions are preventing deletion:

For AWS:
```bash
# Grant necessary permissions for EBS volume management
aws iam attach-role-policy \
  --role-name <node-role-name> \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy
```

For GCP:
```bash
# Grant necessary permissions for disk management
gcloud projects add-iam-policy-binding <project-id> \
  --member serviceAccount:<service-account> \
  --role roles/compute.storageAdmin
```

For Azure:
```bash
# Assign the required role
az role assignment create \
  --assignee <service-principal-id> \
  --role "Contributor" \
  --scope /subscriptions/<subscription-id>/resourceGroups/<resource-group>
```

### 8. Fix PV Protection Issues

If PV protection is preventing deletion:

```bash
# Check for PV protection finalizer
kubectl get pv <pv-name> -o jsonpath='{.metadata.finalizers}'

# Remove the kubernetes.io/pv-protection finalizer if present
kubectl patch pv <pv-name> --type=json \
  -p='[{"op": "remove", "path": "/metadata.finalizers"}]'
```

### 9. Handling StatefulSet PV Reclamation

For StatefulSets, handle PVs carefully:

```bash
# Use deployments with PVCs instead of StatefulSets if reclamation is needed
# Or set cascading deletion to false when removing StatefulSets
kubectl delete statefulset <statefulset-name> --cascade=false
```

Then manually handle PVCs as needed.

### 10. Fix Cloud Provider API Rate Limiting

If cloud APIs are rate-limited during mass deletion:

```bash
# Delete PVs in batches rather than all at once
kubectl get pv | grep Released | head -5 | awk '{print $1}' | xargs kubectl delete pv
# Wait a few minutes then run again for the next batch
```

## Prevention

1. **Default to Retain**: Configure StorageClasses with reclaimPolicy: Retain for critical data
2. **Document Policies**: Clearly document reclaim policies for all storage classes
3. **Regular Auditing**: Regularly audit Released PVs and reclaim as needed
4. **Backup Strategy**: Implement backups before deleting PVCs
5. **Testing**: Test PV reclamation behavior in non-production
6. **Standardize StorageClasses**: Use consistently configured StorageClasses
7. **IAM Roles**: Ensure proper IAM roles and permissions for volume lifecycle management
8. **Volume Snapshots**: Use VolumeSnapshots before deletion for critical data
9. **Monitoring**: Monitor PV states and alert on Released PVs
10. **Retention Labels**: Label PVs that should never be automatically deleted

## Related Runbooks

* [Storage Class Issues](./storage-class-issues.md)
* [PVC Stuck in Pending](./pvc-pending.md)
* [StatefulSet Storage Issues](./statefulset-storage-issues.md)
* [Dynamic Provisioning Issues](./dynamic-provisioning-issues.md)
* [CSI Driver Issues](./csi-driver-issues.md)