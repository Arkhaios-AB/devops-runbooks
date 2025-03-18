# Troubleshooting Storage Class Issues

## Symptoms

* PVCs stuck in `Pending` state
* No default StorageClass available
* Dynamic provisioning of volumes not working
* Error events mentioning "failed to provision volume"
* Storage provisioner errors in logs
* PVs created with incorrect parameters
* Volume expansion failures
* Incorrect volume binding behavior
* Unexpected volume deletion when pods are removed
* Volume provisioning taking too long or timing out

## Possible Causes

1. **Missing StorageClass**: No StorageClass defined in the cluster
2. **No Default StorageClass**: No StorageClass marked as default
3. **Provisioner Issues**: Storage provisioner not running or misconfigured
4. **Wrong Parameters**: Incorrect parameters in StorageClass definition
5. **Cloud Provider Issues**: Problems with cloud provider storage backend
6. **Permission Problems**: Insufficient permissions to provision storage
7. **Resource Constraints**: Resource quotas or limits preventing provisioning
8. **Volume Binding Mode**: Inappropriate volume binding mode
9. **Reclaim Policy**: Incorrect reclaim policy causing unexpected behavior
10. **CSI Driver Issues**: Container Storage Interface driver problems

## Diagnosis Steps

### 1. Check StorageClass Availability

```bash
# List StorageClasses
kubectl get storageclass

# Check if any StorageClass is marked as default
kubectl get storageclass -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}{"\n"}{end}'
```

### 2. Check PVC Status

```bash
# List PVCs and their status
kubectl get pvc -A

# Examine a specific pending PVC
kubectl describe pvc <pvc-name> -n <namespace>
```

### 3. Check Storage Provisioner

```bash
# Check storage provisioner pods
kubectl get pods -n kube-system | grep -E 'provisioner|csi'

# Check logs of provisioner pod
kubectl logs -n kube-system <provisioner-pod-name>
```

### 4. Check StorageClass Parameters

```bash
# Examine StorageClass details
kubectl describe storageclass <storage-class-name>

# Get StorageClass in YAML format
kubectl get storageclass <storage-class-name> -o yaml
```

### 5. Check Cloud Provider Status

For cloud environments, check cloud provider console:
- AWS: Check EBS volume status
- GCP: Check Persistent Disk status
- Azure: Check Managed Disk status

### 6. Check CSI Driver Status

```bash
# List CSI driver pods
kubectl get pods -n kube-system -l app=<csi-driver-name>

# Check CSI driver logs
kubectl logs -n kube-system -l app=<csi-driver-name> -c <csi-driver-container>
```

## Resolution Steps

### 1. Create Missing StorageClass

If StorageClass is missing:

```bash
# Create StorageClass for AWS EBS
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF

# Create StorageClass for GCE PD
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF

# Create StorageClass for Azure Disk
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/azure-disk
parameters:
  storageaccounttype: Standard_LRS
  kind: Managed
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF
```

### 2. Set Default StorageClass

If no default StorageClass is set:

```bash
# Mark a StorageClass as default
kubectl patch storageclass <storage-class-name> -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Remove default annotation from other StorageClasses if needed
kubectl patch storageclass <old-default-sc> -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

### 3. Fix Storage Provisioner

If the provisioner has issues:

```bash
# Restart storage provisioner deployment
kubectl rollout restart deployment <provisioner-deployment> -n kube-system

# For external provisioners, check their specific deployment
kubectl get deployment -A | grep provision
```

### 4. Fix CSI Driver

If CSI driver has problems:

```bash
# Restart CSI driver pods
kubectl rollout restart deployment <csi-controller-deployment> -n kube-system
kubectl rollout restart daemonset <csi-node-daemonset> -n kube-system

# Apply updated CSI driver manifest
kubectl apply -f <updated-csi-manifest>
```

### 5. Fix Volume Binding Mode

If volume binding mode is causing issues:

```bash
# Create new StorageClass with appropriate binding mode
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: <storage-class-name>
provisioner: <provisioner>
parameters:
  # parameters specific to provisioner
volumeBindingMode: WaitForFirstConsumer  # Use this for better scheduling
reclaimPolicy: Delete
allowVolumeExpansion: true
EOF
```

Volume binding modes:
- `Immediate`: Volumes are provisioned immediately when PVC is created
- `WaitForFirstConsumer`: Volumes are provisioned when pod using PVC is scheduled

### 6. Fix Reclaim Policy

If reclaim policy is causing unexpected volume deletion:

```bash
# Create StorageClass with Retain policy
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: retained-storage
provisioner: <provisioner>
parameters:
  # parameters specific to provisioner
reclaimPolicy: Retain  # PVs will not be deleted when PVC is deleted
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
EOF
```

### 7. Fix Cloud Provider Permissions

If permissions are insufficient:

For AWS:
```bash
# Update IAM role with proper permissions for EBS
# Example policy needs:
# - ec2:CreateVolume
# - ec2:DeleteVolume
# - ec2:AttachVolume
# - ec2:DetachVolume
# - ec2:DescribeVolumes
# - ec2:CreateSnapshot
# - ec2:DeleteSnapshot
# - ec2:DescribeSnapshots
```

For GCP:
```bash
# Update IAM role with compute.disks.* permissions
```

For Azure:
```bash
# Ensure managed identity or service principal has:
# - Microsoft.Compute/disks/read
# - Microsoft.Compute/disks/write
# - Microsoft.Compute/disks/delete
```

### 8. Fix Parameters for Specific Cloud Providers

For AWS EBS:
```bash
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iopsPerGB: "3000"
  throughput: "125"
  encrypted: "true"
  kmsKeyId: "<optional-kms-key-id>"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF
```

For GCE PD:
```bash
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  fstype: ext4
  replication-type: none
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF
```

### 9. Enable Volume Expansion

If volume expansion is needed:

```bash
# Create or update StorageClass with expansion enabled
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable-storage
provisioner: <provisioner>
parameters:
  # parameters specific to provisioner
reclaimPolicy: Delete
allowVolumeExpansion: true  # Enable volume expansion
volumeBindingMode: WaitForFirstConsumer
EOF
```

### 10. Fix Zone Issues

For multi-zone clusters, specify allowed zones:

```bash
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: zoned-storage
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  zones: "us-east-1a,us-east-1b"  # Specify allowed zones
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF
```

## Prevention

1. **Default StorageClass**: Always have a default StorageClass configured
2. **Documentation**: Document StorageClass parameters and limitations
3. **Testing**: Test volume provisioning in non-production before deployment
4. **Monitoring**: Monitor provisioner logs and PVC/PV creation
5. **Regular Maintenance**: Keep storage drivers and CSI components updated
6. **Permissions**: Ensure proper IAM/RBAC permissions for provisioners
7. **Resource Planning**: Plan storage capacity and quota requirements
8. **Multi-zone Awareness**: Configure StorageClasses with zone awareness
9. **Backup Strategy**: Implement regular volume backup procedures
10. **Volume Templates**: Create templates for commonly used PVCs

## Related Runbooks

* [PVC Stuck in Pending](./pvc-pending.md)
* [Volume Mount Problems](./volume-mount-problems.md)
* [CSI Driver Issues](./csi-driver-issues.md)
* [StatefulSet Storage Issues](./statefulset-storage-issues.md)
* [Dynamic Provisioning Issues](./dynamic-provisioning-issues.md)
* [PV Reclaim Policy Issues](./pv-reclaim-policy-issues.md)