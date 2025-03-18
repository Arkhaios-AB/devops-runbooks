# Troubleshooting PVC Stuck in Pending

## Symptoms

* Persistent Volume Claims (PVCs) remain in `Pending` state
* Pods that depend on these PVCs are stuck in `ContainerCreating` or `Pending` state
* Error events show volume provisioning failures
* Storage operations cannot complete

## Possible Causes

1. **Missing or Invalid StorageClass**: The specified StorageClass doesn't exist or is invalid
2. **Storage Provisioner Issues**: CSI driver or provisioner is not functioning correctly
3. **Storage Capacity Issues**: Not enough capacity in the storage backend
4. **Permissions/RBAC Issues**: Service account lacks permissions to provision storage
5. **Misconfigured PVC**: Incorrect access mode, capacity, or selectors in PVC definition
6. **Cloud Provider Quota Limits**: Reached quota limits on your cloud provider
7. **Zone/Topology Constraints**: Storage and pods in different availability zones
8. **Existing PV Cannot be Bound**: No matching PV available for static provisioning

## Diagnosis Steps

### 1. Check PVC Status and Events

```bash
# Check PVC status
kubectl get pvc -n <namespace>

# Get detailed information about the PVC
kubectl describe pvc <pvc-name> -n <namespace>

# Check events related to the PVC
kubectl get events -n <namespace> | grep <pvc-name>
```

### 2. Check StorageClass Configuration

```bash
# List available StorageClasses
kubectl get storageclass

# Get details of the StorageClass used by the PVC
kubectl describe storageclass <storage-class-name>
```

### 3. Check Storage Provisioner

```bash
# Check CSI driver pods if applicable
kubectl get pods -n kube-system -l app=<csi-driver-name>

# Check logs of provisioner pods
kubectl logs -n kube-system <provisioner-pod-name>
```

### 4. Check Existing PVs

```bash
# List available PVs
kubectl get pv

# Check if there are matching PVs for static provisioning
kubectl get pv -o wide | grep <storage-class-name>
```

## Resolution Steps

### 1. Fix StorageClass Issues

If the StorageClass doesn't exist or is invalid:

```bash
# Create the required StorageClass
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: <storage-class-name>
provisioner: <provisioner>
parameters:
  type: gp2  # Example for AWS EBS
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF
```

### 2. Fix Provisioner Issues

If the storage provisioner is not functioning:

```bash
# Restart provisioner pods
kubectl rollout restart deployment <provisioner-deployment> -n kube-system

# For cloud providers, check CSI driver installation
kubectl apply -f <cloud-provider-csi-driver-manifest>
```

### 3. Fix PVC Configuration

If the PVC configuration is incorrect:

```bash
# Update the PVC definition
kubectl edit pvc <pvc-name> -n <namespace>
```

Example of a proper PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: example-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
```

### 4. Fix Permission Issues

If there are RBAC issues:

```bash
# Create or update the necessary RBAC rules
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: storage-provisioner-role
rules:
- apiGroups: [""]
  resources: ["persistentvolumes"]
  verbs: ["get", "list", "watch", "create", "delete"]
- apiGroups: [""]
  resources: ["persistentvolumeclaims"]
  verbs: ["get", "list", "watch", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: storage-provisioner-binding
subjects:
- kind: ServiceAccount
  name: <provisioner-service-account>
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: storage-provisioner-role
  apiGroup: rbac.authorization.k8s.io
EOF
```

### 5. Address Capacity Issues

If there's not enough storage capacity:

* Increase quota limits in your cloud provider console
* Free up space by deleting unused PVs/PVCs
* Provision additional storage in your backend

### 6. Fix Zone/Topology Issues

If there are topology constraints:

```yaml
# Add node affinity to ensure pods land in the right zone
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - <availability-zone>
```

## Prevention

1. **Default StorageClass**: Set up a default StorageClass for the cluster
2. **Storage Capacity Planning**: Regularly monitor and plan for storage capacity
3. **Monitoring**: Set up alerts for storage provisioning failures
4. **Use WaitForFirstConsumer**: Set `volumeBindingMode: WaitForFirstConsumer` in StorageClass
5. **Regular Testing**: Regularly test storage provisioning as part of CI/CD pipelines
6. **Storage Quotas**: Implement storage resource quotas for namespaces
7. **Documentation**: Document storage classes and their capabilities for developers

## Related Runbooks

* [Storage Class Issues](./storage-class-issues.md)
* [CSI Driver Issues](./csi-driver-issues.md)
* [Volume Mount Problems](./volume-mount-problems.md)
* [Pod Stuck in ContainerCreating](../workloads/pod-container-creating.md)
* [StatefulSet Storage Issues](./statefulset-storage-issues.md)