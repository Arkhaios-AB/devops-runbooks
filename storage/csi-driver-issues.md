# Troubleshooting CSI Driver Issues

## Symptoms

* Persistent Volume Claims (PVCs) stuck in `Pending` state
* Errors about failing to provision volumes
* CSI driver pods in CrashLoopBackOff or Error state
* "Failed to attach volume" or "Failed to mount volume" errors
* Volume attachment timeout errors
* CSI node plugin registration failures
* Storage features like snapshots or volume expansion not working
* Unexpected volume detachment
* Volume corruption or data access issues
* CSI metrics showing abnormal values

## Possible Causes

1. **CSI Driver Installation Issues**: Missing or incorrect CSI driver components
2. **CSI Node Plugin Problems**: CSI node plugin registration failures
3. **Storage Backend Issues**: Problems with storage backend (cloud provider, NAS, etc.)
4. **Credential Problems**: Incorrect or missing credentials for storage backend
5. **Version Incompatibility**: CSI driver version incompatible with Kubernetes version
6. **Resource Constraints**: CSI driver pods hitting resource limits
7. **Node Kernel Issues**: Missing kernel modules or unsupported kernel version
8. **StorageClass Configuration**: Incorrect StorageClass parameters
9. **CSI Driver Configuration**: Misconfigured CSI driver settings
10. **Permissions Problems**: Insufficient permissions for CSI components

## Diagnosis Steps

### 1. Check CSI Driver Pod Status

```bash
# Find and check CSI driver pods
kubectl get pods -n kube-system | grep csi
kubectl get pods -A | grep csi

# Check detailed pod status
kubectl describe pod <csi-pod-name> -n <namespace>

# Check CSI driver pod logs
kubectl logs <csi-pod-name> -n <namespace>
kubectl logs <csi-pod-name> -c <container-name> -n <namespace>  # For multi-container pods
```

### 2. Check CSI Driver Installation

```bash
# Check CSI driver CRDs
kubectl get crd | grep csi

# Check CSI driver StorageClasses
kubectl get storageclass | grep <csi-provisioner-name>

# Check CSI driver deployments/daemonsets
kubectl get deployment,daemonset -A | grep csi
```

### 3. Check Volume Attachment

```bash
# Check volume attachments
kubectl get volumeattachments

# Check persistent volumes
kubectl get pv
kubectl describe pv <pv-name>

# Check persistent volume claims
kubectl get pvc -A
kubectl describe pvc <pvc-name> -n <namespace>
```

### 4. Check CSI Registration

```bash
# Check CSI driver registration
kubectl get csidrivers
kubectl describe csidriver <driver-name>

# Check node CSI information
kubectl describe node <node-name> | grep csi
```

### 5. Check Storage Backend Status

For cloud providers:

```bash
# AWS
aws ec2 describe-volumes --volume-ids <volume-id>

# GCP
gcloud compute disks describe <disk-name> --zone=<zone>

# Azure
az disk show --name <disk-name> --resource-group <resource-group>
```

For on-premises storage, check the respective storage management interfaces.

## Resolution Steps

### 1. Fix CSI Driver Installation

If CSI driver is missing or incorrectly installed:

```bash
# Install or update CSI driver
# For AWS EBS CSI Driver
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"

# For GCP PD CSI Driver
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gcp-compute-persistent-disk-csi-driver/master/deploy/kubernetes/overlays/stable/driver.yaml

# For Azure Disk CSI Driver
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/master/deploy/csi-azuredisk-driver.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/master/deploy/csi-azuredisk-controller.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/master/deploy/csi-azuredisk-node.yaml
```

### 2. Fix CSI Node Plugin Issues

If node plugin registration fails:

```bash
# Restart CSI node plugin daemonset
kubectl rollout restart daemonset <csi-node-daemonset-name> -n <namespace>

# Check for missing kernel modules on node
ssh <node> lsmod | grep <required-module>

# Load missing kernel module if needed
ssh <node> sudo modprobe <required-module>
```

### 3. Fix Credential Issues

If storage backend credentials are incorrect:

```bash
# Update credentials secret
kubectl create secret generic <csi-credentials-secret> \
  --from-literal=key1=value1 \
  --from-literal=key2=value2 \
  -n <namespace> \
  --dry-run=client -o yaml | kubectl apply -f -
```

For cloud providers with IAM roles:

```bash
# For AWS EBS CSI Driver with IRSA (IAM Roles for Service Accounts)
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster <cluster-name> \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --override-existing-serviceaccounts
```

### 4. Fix Resource Constraints

If CSI controller is hitting resource limits:

```bash
# Edit CSI controller deployment to increase resources
kubectl edit deployment <csi-controller-deployment> -n <namespace>
```

Example resource adjustment:
```yaml
resources:
  limits:
    cpu: "500m"
    memory: "512Mi"
  requests:
    cpu: "100m"
    memory: "128Mi"
```

### 5. Fix StorageClass Configuration

If StorageClass is incorrectly configured:

```bash
# Create or update StorageClass with correct parameters
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: <storageclass-name>
provisioner: <csi-driver-name>
parameters:
  # Parameters specific to the CSI driver
  type: gp3  # Example for AWS EBS
  fsType: ext4
volumeBindingMode: WaitForFirstConsumer  # Recommended for most CSI drivers
allowVolumeExpansion: true
EOF
```

### 6. Fix Version Incompatibility

If CSI driver version is incompatible with Kubernetes:

```bash
# Check current version
kubectl describe csidriver <driver-name>

# Update to compatible version
# Example for AWS EBS CSI Driver
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=v1.5.0"
```

### 7. Fix CSI Feature Gates

If CSI features aren't working:

```bash
# Check feature gates in kube-apiserver
kubectl get pods -n kube-system -l component=kube-apiserver -o yaml | grep feature-gates

# Check CSI Snapshotter deployment
kubectl get deployment -n kube-system | grep snapshot
kubectl get crd | grep snapshot
```

If missing, install CSI Snapshotter:
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v4.2.0/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v4.2.0/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v4.2.0/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml
```

### 8. Fix Permissions Issues

If CSI components lack necessary permissions:

```bash
# Create or update RBAC rules
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: csi-provisioner
  namespace: kube-system
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: external-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: csi-provisioner-role
subjects:
  - kind: ServiceAccount
    name: csi-provisioner
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: external-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
EOF
```

### 9. Force Volume Detachment

If volumes are stuck in attaching/detaching state:

```bash
# Force delete VolumeAttachment
kubectl patch volumeattachment <volumeattachment-name> -p '{"metadata":{"finalizers":null}}' --type=merge
```

### 10. Recreate CSI Driver

If all else fails:

```bash
# Uninstall and reinstall CSI driver
# For AWS EBS CSI Driver example
kubectl delete -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"
```

## Prevention

1. **Monitoring**: Monitor CSI driver components health and metrics
2. **Version Compatibility**: Ensure CSI driver is compatible with Kubernetes version
3. **Regular Updates**: Keep CSI drivers updated with security patches
4. **Resource Planning**: Allocate sufficient resources for CSI components
5. **Proper IAM Configuration**: Configure appropriate IAM roles and permissions
6. **Regular Testing**: Test volume provisioning in non-production environments
7. **Backup Strategy**: Implement regular volume backup procedures
8. **Documentation**: Document CSI driver configuration and troubleshooting steps
9. **Health Checks**: Implement health checks for storage backend
10. **Capacity Planning**: Monitor and plan for storage capacity needs

## Related Runbooks

* [PVC Stuck in Pending](./pvc-pending.md)
* [Storage Class Issues](./storage-class-issues.md)
* [Volume Mount Problems](./volume-mount-problems.md)
* [Pod Stuck in ContainerCreating](../workloads/pod-container-creating.md)
* [StatefulSet Issues](../workloads/statefulset-issues.md)