# Troubleshooting StatefulSet Issues

## Symptoms

* StatefulSet pods not launching in sequence
* StatefulSet stuck during scaling or updates
* Pods created without proper ordinal suffixes
* Persistent volumes not being created or attached
* Headless service not resolving individual pod DNS entries
* Pod identity or state issues during updates
* StatefulSet update operations failing
* Pods recreated with new persistent volumes despite PVC retention policy

## Possible Causes

1. **Storage Provisioning Issues**: Problems with dynamic volume provisioning
2. **Headless Service Problems**: Issues with the headless service configuration
3. **Pod Management Policy**: Incorrect pod management policy for the workload
4. **Update Strategy Issues**: Problematic update strategy configuration
5. **PVC Template Problems**: Errors in the volumeClaimTemplate
6. **Storage Class Issues**: StorageClass not configured properly
7. **Pod Spec Problems**: Issues with pod specification
8. **StatefulSet Controller Issues**: StatefulSet controller not functioning correctly
9. **Volume Binding Mode**: Incorrect volume binding mode in StorageClass
10. **Zone Distribution**: Pods and volumes in different availability zones

## Diagnosis Steps

### 1. Check StatefulSet Status

```bash
# Get StatefulSet status
kubectl get statefulset <statefulset-name> -n <namespace>

# Check detailed StatefulSet information
kubectl describe statefulset <statefulset-name> -n <namespace>

# Check StatefulSet events
kubectl get events -n <namespace> | grep <statefulset-name>
```

### 2. Check Pod Status and Ordering

```bash
# List pods and their creation timestamps
kubectl get pods -n <namespace> -l app=<app-label> --sort-by=.metadata.creationTimestamp

# Check individual pod details
kubectl describe pod <pod-name> -n <namespace>
```

### 3. Check PVCs and PVs

```bash
# List PVCs created by the StatefulSet
kubectl get pvc -n <namespace> -l app=<app-label>

# Check PVC details
kubectl describe pvc <pvc-name> -n <namespace>

# List PVs bound to the PVCs
kubectl get pv | grep <pvc-name>
```

### 4. Check Headless Service

```bash
# Verify headless service exists
kubectl get svc <headless-service-name> -n <namespace>

# Check service details
kubectl describe svc <headless-service-name> -n <namespace>

# Check endpoints
kubectl get endpoints <headless-service-name> -n <namespace>
```

### 5. Test DNS Resolution

```bash
# Create a debug pod
kubectl run debug-dns --image=busybox:1.28 -n <namespace> -- sleep 3600

# Test DNS resolution for individual StatefulSet pods
kubectl exec -it debug-dns -n <namespace> -- nslookup <pod-name>.<headless-service-name>.<namespace>.svc.cluster.local
```

### 6. Check StorageClass Configuration

```bash
# List StorageClasses
kubectl get storageclass

# Check details of the StorageClass used by the StatefulSet
kubectl describe storageclass <storage-class-name>
```

## Resolution Steps

### 1. Fix Storage Provisioning Issues

If PVCs are stuck in Pending state:

```bash
# Create the required StorageClass if missing
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: <storage-class-name>
provisioner: <provisioner>
parameters:
  type: gp2  # Example for AWS EBS
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF
```

### 2. Fix Headless Service

If headless service is misconfigured:

```bash
# Create or update the headless service
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: <headless-service-name>
  namespace: <namespace>
  labels:
    app: <app-label>
spec:
  clusterIP: None
  selector:
    app: <app-label>
  ports:
  - port: 80
    name: web
EOF
```

### 3. Fix Update Strategy

If updates are stuck or problematic:

```bash
# Edit StatefulSet update strategy
kubectl edit statefulset <statefulset-name> -n <namespace>
```

Example strategies:
```yaml
# For ordered updates (default)
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    partition: 0  # Start updating from this ordinal index

# For parallel updates
updateStrategy:
  type: OnDelete  # Update only when pods are manually deleted
```

### 4. Fix Pod Management Policy

If pods are not being created or deleted in the expected order:

```bash
# Edit StatefulSet pod management policy
kubectl edit statefulset <statefulset-name> -n <namespace>
```

Example policies:
```yaml
# For sequential pod management (default)
podManagementPolicy: OrderedReady

# For parallel pod management
podManagementPolicy: Parallel
```

### 5. Fix PVC Template

If PVC templates are causing issues:

```bash
# Edit StatefulSet volumeClaimTemplates
kubectl edit statefulset <statefulset-name> -n <namespace>
```

Example of proper volumeClaimTemplate:
```yaml
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    accessModes: [ "ReadWriteOnce" ]
    storageClassName: "<storage-class-name>"
    resources:
      requests:
        storage: 10Gi
```

### 6. Fix Volume Binding Mode

If volumes are not binding correctly:

```bash
# Patch the StorageClass to use the correct binding mode
kubectl patch storageclass <storage-class-name> \
  -p '{"volumeBindingMode":"WaitForFirstConsumer"}'
```

### 7. Handle PVC Deletion for Scaling Down

When scaling down StatefulSets, PVCs are not automatically deleted:

```bash
# Manually delete PVCs for removed pods (if desired)
kubectl delete pvc <pvc-name> -n <namespace>
```

### 8. Fix Zone Affinity Issues

To ensure pods and their volumes are in the same zone:

```bash
# Edit StatefulSet to add pod affinity based on PV zone
kubectl edit statefulset <statefulset-name> -n <namespace>
```

Example affinity configuration:
```yaml
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

### 9. Restart StatefulSet Controller (in extreme cases)

If the StatefulSet controller appears to be malfunctioning:

```bash
# Restart kube-controller-manager in a kubeadm cluster
sudo mv /etc/kubernetes/manifests/kube-controller-manager.yaml /tmp/
sleep 5
sudo mv /tmp/kube-controller-manager.yaml /etc/kubernetes/manifests/
```

### 10. Recreate StatefulSet as Last Resort

If the StatefulSet is completely stuck:

```bash
# Export the current StatefulSet definition
kubectl get statefulset <statefulset-name> -n <namespace> -o yaml > statefulset.yaml

# Edit the YAML file to remove status and other generated fields
# Then delete and recreate (WARNING: Be careful with PVCs)
kubectl delete statefulset <statefulset-name> -n <namespace> --cascade=false  # Keeps pods and PVCs
kubectl apply -f statefulset.yaml
```

## Prevention

1. **Use Appropriate Storage Class**: Select the right StorageClass for your workload
2. **Pod Disruption Budgets**: Implement PDBs for StatefulSets to prevent unsafe scaling
3. **Regular Testing**: Test StatefulSet scaling and updates in non-production
4. **Backup Strategy**: Implement regular backups of StatefulSet data
5. **Monitoring**: Monitor StatefulSet status and PVC creation/binding
6. **Documentation**: Document StatefulSet configuration and expected behavior
7. **Zone Awareness**: Configure zone awareness for multi-zone clusters
8. **Resource Planning**: Ensure adequate resources for StatefulSet pods
9. **StatefulSet Design**: Properly design StatefulSets for workload requirements
10. **Update Strategy**: Choose the appropriate update strategy for your application

## Related Runbooks

* [PVC Stuck in Pending](../storage/pvc-pending.md)
* [Storage Class Issues](../storage/storage-class-issues.md)
* [Volume Mount Problems](../storage/volume-mount-problems.md)
* [StatefulSet Storage Issues](../storage/statefulset-storage-issues.md)
* [Deployment Rollout Issues](./deployment-rollout-issues.md)
* [Service Not Accessible](../networking/service-not-accessible.md)
* [DNS Resolution Problems](../networking/dns-resolution-problems.md)