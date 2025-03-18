# Troubleshooting Node Disk Pressure

## Symptoms

* Node shows `DiskPressure=True` condition
* Pods may be evicted from the node
* `kubectl describe node` shows a DiskPressure condition
* Kubelet logs showing disk pressure warnings or errors
* Container runtime reporting errors related to storage
* Errors creating or writing to files in pods
* Node becomes unstable or goes into NotReady state

## Possible Causes

1. **Container Log Accumulation**: Log files growing too large
2. **Unused Container Images**: Accumulation of unused container images
3. **Orphaned Containers**: Containers not cleaned up properly
4. **Temporary Storage Exhaustion**: /tmp directory filling up
5. **EmptyDir Volumes**: Excessive use of emptyDir volumes
6. **System Logs**: System logs consuming excessive space
7. **Node Root Partition Full**: Root partition running out of space
8. **Node Ephemeral Storage Full**: Ephemeral storage partition filling up
9. **Incorrect Garbage Collection**: Improper kubelet garbage collection settings
10. **Slow Disk I/O**: Disk I/O bottlenecks causing operation backlog

## Diagnosis Steps

### 1. Check Node Disk Pressure

```bash
# Check node conditions
kubectl get nodes

# Check disk pressure condition
kubectl describe node <node-name> | grep DiskPressure

# Check allocatable disk resources
kubectl describe node <node-name> | grep -A 5 Allocatable
```

### 2. Check Disk Usage on Node

SSH into the affected node and check disk usage:

```bash
# Check overall disk usage
df -h

# Check inodes usage (can also cause disk pressure)
df -i

# Find large directories
sudo du -h --max-depth=1 /var | sort -hr | head -10
sudo du -h --max-depth=1 /etc | sort -hr | head -10
```

### 3. Check Container and Image Usage

```bash
# For Docker
docker system df -v

# For containerd
sudo crictl info | grep -A 20 storage

# List all images
sudo crictl images

# List containers
sudo crictl ps -a
```

### 4. Check Log Directory Size

```bash
# Check log directory size
sudo du -sh /var/log

# Check journal logs size
sudo journalctl --disk-usage
```

### 5. Check Kubelet Settings

```bash
# Check kubelet config for garbage collection settings
sudo cat /var/lib/kubelet/config.yaml | grep -A 10 "imageGC"
```

## Resolution Steps

### 1. Clean Up Container Images

```bash
# For Docker
docker system prune -a -f

# For containerd
sudo crictl rmi --prune
```

### 2. Clean Up Logs

```bash
# Rotate and compress logs
sudo logrotate -f /etc/logrotate.conf

# Truncate large log files
sudo truncate -s 0 /var/log/large-log-file.log

# Clean up journal logs
sudo journalctl --vacuum-time=1d
sudo journalctl --vacuum-size=500M
```

### 3. Clean Up Unused Containers

```bash
# For Docker
docker rm $(docker ps -aq -f status=exited)

# For containerd
sudo crictl rm $(sudo crictl ps -a -q --state exited)
```

### 4. Free Up Root Directory Space

```bash
# Remove temporary files
sudo rm -rf /tmp/* /var/tmp/*

# Clean package cache (Ubuntu/Debian)
sudo apt clean

# Clean package cache (CentOS/RHEL)
sudo yum clean all
```

### 5. Expand Disk Space (if possible)

For cloud provider nodes:

* Resize the disk in the cloud provider console
* Expand the filesystem to use the new space

```bash
# Example for AWS EBS volume after resizing in console
sudo growpart /dev/xvda 1
sudo resize2fs /dev/xvda1
```

### 6. Adjust Kubelet Garbage Collection Settings

Edit the kubelet configuration to adjust garbage collection thresholds:

```bash
sudo vi /var/lib/kubelet/config.yaml
```

Example settings:

```yaml
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
evictionHard:
  imagefs.available: 10%
  nodefs.available: 5%
  nodefs.inodesFree: 5%
```

Then restart kubelet:

```bash
sudo systemctl restart kubelet
```

### 7. Configure Pod Log Rotation

For containerized applications, ensure they use log rotation:

```yaml
# Add this to container spec
volumeMounts:
- name: varlog
  mountPath: /var/log
volumes:
- name: varlog
  emptyDir: {}
```

### 8. Restrict EmptyDir Volume Sizes

Set size limits on emptyDir volumes:

```yaml
volumes:
- name: cache-volume
  emptyDir:
    sizeLimit: 500Mi
```

### 9. Uncordon the Node

After resolving the issue:

```bash
kubectl uncordon <node-name>
```

## Prevention

1. **Monitor Disk Usage**: Set up monitoring and alerting for disk usage
2. **Configure Log Rotation**: Implement log rotation for all applications
3. **Configure Garbage Collection**: Set appropriate garbage collection thresholds in kubelet
4. **Regular Cleanup**: Schedule regular cleanup of unused containers and images
5. **Limit EmptyDir Sizes**: Set size limits on emptyDir volumes in pod specs
6. **Use Persistent Volumes**: Use persistent volumes for data that needs to survive pod restarts
7. **Resource Quotas**: Implement ephemeral storage resource quotas for namespaces
8. **Configure Pod Log Limits**: Set log size limits for pods
9. **Regular System Maintenance**: Schedule regular system maintenance for nodes

## Related Runbooks

* [Node Not Ready](./node-not-ready.md)
* [Node Memory Pressure](./node-memory-pressure.md)
* [Node CPU Pressure](./node-cpu-pressure.md)
* [Kubelet Issues](./kubelet-issues.md)
* [Storage Capacity Issues](../storage/storage-capacity-issues.md)