# Troubleshooting Volume Mount Problems

## Symptoms

* Pods stuck in `ContainerCreating` state with volume-related errors
* Permission denied errors when accessing mounted volumes
* Applications reporting missing files or directories
* Unable to write to mounted volumes
* I/O errors in container logs
* Read-only filesystem errors
* Slow I/O performance on mounted volumes
* Pods crash when attempting to access mounted volumes
* Events showing `MountVolume.SetUp failed`

## Possible Causes

1. **PVC Not Bound**: Persistent Volume Claim not bound to a Persistent Volume
2. **Permission Issues**: Wrong ownership or permissions on mounted volumes
3. **Path Issues**: Incorrect mount paths or non-existent directories
4. **Filesystem Errors**: Corrupted filesystem on the volume
5. **Volume Type Issues**: Issues specific to volume type (NFS, EBS, etc.)
6. **Node Problems**: Node-level storage or mounting problems
7. **Security Context**: Pod security context preventing proper volume access
8. **HostPath Issues**: Problems with hostPath volumes on specific nodes
9. **Mount Propagation**: Incorrect mount propagation settings
10. **Stale Mounts**: Previous mounts not properly cleaned up

## Diagnosis Steps

### 1. Check Pod Status and Events

```bash
# Get pod status
kubectl get pod <pod-name> -n <namespace>

# Check pod events
kubectl describe pod <pod-name> -n <namespace>

# Check logs for volume-related errors
kubectl logs <pod-name> -n <namespace>
```

### 2. Check PVC and PV Status

```bash
# List all PVCs
kubectl get pvc -n <namespace>

# Check specific PVC
kubectl describe pvc <pvc-name> -n <namespace>

# Check bound PV
kubectl get pv | grep <pvc-name>
kubectl describe pv <pv-name>
```

### 3. Check Volume Mounts in Pod Spec

```bash
# Check volume mounts configuration
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A 10 volumeMounts
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A 15 volumes
```

### 4. Check Storage Class

```bash
# List storage classes
kubectl get storageclass

# Check details of storage class
kubectl describe storageclass <storage-class-name>
```

### 5. Check Node Status

```bash
# Find node where pod is running
NODE=$(kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.nodeName}')

# Check node status
kubectl describe node $NODE
```

### 6. Check Kubelet Logs on Node

SSH to the node and check logs:

```bash
# Check kubelet logs for mount issues
sudo journalctl -u kubelet | grep -i mount | grep <pod-uid>
sudo journalctl -u kubelet | grep -i volume | grep <pod-uid>
```

### 7. Inspect the Volume on the Node

SSH to the node where the pod is scheduled and check:

```bash
# Find the mounted volume
sudo findmnt | grep <pv-name>

# Check mount permissions and ownership
ls -la /var/lib/kubelet/pods/<pod-uid>/volumes/
```

## Resolution Steps

### 1. Fix PVC Binding Issues

If the PVC is stuck in Pending:

```bash
# Check why PVC is not binding
kubectl describe pvc <pvc-name> -n <namespace>

# Create the required storage class if missing
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

### 2. Fix Permission Issues

If the pod has permission errors:

```bash
# Add security context to pod spec
kubectl edit deployment <deployment-name> -n <namespace>
```

Example security context:
```yaml
securityContext:
  fsGroup: 1000  # Set to appropriate group ID
  runAsUser: 1000  # Set to appropriate user ID
```

Or for a temporary fix, use an init container:

```yaml
initContainers:
- name: fix-permissions
  image: busybox
  command: ['sh', '-c', 'chown -R 1000:1000 /data']
  volumeMounts:
  - name: data-volume
    mountPath: /data
  securityContext:
    runAsUser: 0  # Run as root to change permissions
```

### 3. Fix Mount Path Issues

If the mount path is incorrect:

```bash
# Edit deployment to fix mount paths
kubectl edit deployment <deployment-name> -n <namespace>
```

Make sure the mount path exists in the container:
```yaml
volumeMounts:
- name: data-volume
  mountPath: /path/that/exists/in/container
```

### 4. Fix Filesystem Errors

If there are filesystem errors:

```bash
# Create a debug pod with the same volume
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: volume-debug
  namespace: <namespace>
spec:
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: <pvc-name>
  containers:
  - name: debug
    image: busybox
    command: ['sleep', '3600']
    volumeMounts:
    - name: data-volume
      mountPath: /data
    securityContext:
      privileged: true  # Be careful with this!
EOF

# Then check and fix filesystem
kubectl exec -it volume-debug -n <namespace> -- fsck -y /dev/sdX
```

### 5. Fix Volume Type Specific Issues

For NFS volumes:

```bash
# Check NFS server access
kubectl exec -it volume-debug -n <namespace> -- showmount -e <nfs-server>

# Test NFS mount manually
kubectl exec -it volume-debug -n <namespace> -- mount -t nfs <nfs-server>:<export-path> /mnt
```

For cloud provider volumes, check cloud provider console for issues.

### 6. Fix Stale Mounts

If there are stale mounts on the node:

SSH to the node and:
```bash
# Find and clean up stale mounts
sudo findmnt -t <filesystem-type> | grep <stale-volume>
sudo umount -f /path/to/stale/mount
```

Then restart kubelet:
```bash
sudo systemctl restart kubelet
```

### 7. Fix HostPath Issues

If using hostPath volumes:

```bash
# Check if the path exists on the node
ssh user@node
ls -la /path/on/host

# Create directory if missing
sudo mkdir -p /path/on/host
sudo chown <user>:<group> /path/on/host
```

Then update pod spec if needed:
```yaml
volumes:
- name: host-volume
  hostPath:
    path: /path/on/host
    type: DirectoryOrCreate  # Will create if doesn't exist
```

### 8. Fix Mount Propagation

If mount propagation is an issue:

```bash
# Edit deployment to set proper mount propagation
kubectl edit deployment <deployment-name> -n <namespace>
```

Example mount propagation:
```yaml
volumeMounts:
- name: data-volume
  mountPath: /data
  mountPropagation: Bidirectional  # Options: None, HostToContainer, Bidirectional
```

### 9. Force Delete and Recreate Pod

If all else fails:

```bash
# Force delete the stuck pod
kubectl delete pod <pod-name> -n <namespace> --force --grace-period=0

# If created by controller, it will be recreated automatically
```

### 10. Data Recovery

If data needs to be recovered:

```bash
# Create a recovery pod
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: data-recovery
  namespace: <namespace>
spec:
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: <pvc-name>
  - name: backup-volume
    emptyDir: {}
  containers:
  - name: recovery
    image: busybox
    command: ['sh', '-c', 'cp -a /data/* /backup/ && chmod -R 777 /backup']
    volumeMounts:
    - name: data-volume
      mountPath: /data
    - name: backup-volume
      mountPath: /backup
EOF

# Then copy data out
kubectl cp <namespace>/data-recovery:/backup /local/backup/path
```

## Prevention

1. **Use Storage Classes**: Define proper storage classes for your environment
2. **Consistent Security Context**: Use consistent security context settings
3. **Testing**: Test volume mounts with non-critical workloads first
4. **Monitoring**: Monitor volume usage and performance
5. **Regular Maintenance**: Schedule regular maintenance for filesystems
6. **Documentation**: Document volume requirements for applications
7. **Backup Strategy**: Implement regular volume backups
8. **Access Controls**: Implement proper access controls for volumes
9. **Initialization**: Use init containers to prepare volume mounts
10. **Resource Limits**: Set appropriate storage resource limits

## Related Runbooks

* [PVC Stuck in Pending](./pvc-pending.md)
* [Storage Class Issues](./storage-class-issues.md)
* [Pod Stuck in ContainerCreating](../workloads/pod-container-creating.md)
* [StatefulSet Storage Issues](./statefulset-storage-issues.md)
* [CSI Driver Issues](./csi-driver-issues.md)
* [Node Disk Pressure](../cluster/node-disk-pressure.md)