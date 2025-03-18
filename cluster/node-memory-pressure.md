# Troubleshooting Node Memory Pressure

## Symptoms

* Node shows `MemoryPressure=True` condition
* Pods on the node may be evicted
* New pods may not be scheduled on the node
* System processes or container runtime becoming unstable
* Kubelet logs showing memory pressure events
* `kubectl describe node` shows memory pressure condition

## Possible Causes

1. **High Pod Memory Usage**: Pods consuming more memory than expected
2. **Memory Leaks**: Applications with memory leaks running on the node
3. **System Services**: System services or daemons consuming excessive memory
4. **Insufficient Node Resources**: Node has insufficient memory for its workload
5. **Incorrect Resource Reservations**: Node reserved memory configured incorrectly
6. **Kernel Memory Issues**: Kernel memory consumption problems
7. **Container Runtime Overhead**: High memory usage by container runtime
8. **Memory Fragmentation**: Memory fragmentation preventing allocation of large blocks

## Diagnosis Steps

### 1. Check Node Conditions

```bash
# Check node status
kubectl get nodes

# Check memory pressure condition
kubectl describe node <node-name> | grep MemoryPressure

# Check node memory allocation
kubectl describe node <node-name> | grep -A 10 Allocated
```

### 2. Check Node Memory Usage

SSH into the affected node and check memory usage:

```bash
# Check overall memory usage
free -h

# Check memory usage by processes
top -o %MEM

# Check memory usage in detail
cat /proc/meminfo

# Check if swap is being used
swapon --show
```

### 3. Check Pod Memory Usage

```bash
# Check pod memory usage
kubectl top pods --all-namespaces --sort-by=memory

# Filter for pods on the affected node
kubectl top pods --all-namespaces --sort-by=memory | grep <node-name>

# Check cgroup memory stats
cat /sys/fs/cgroup/memory/memory.stat
```

### 4. Check Kubelet Logs

```bash
# Check kubelet logs for memory pressure events
sudo journalctl -u kubelet | grep -i "memory pressure"
```

### 5. Check System Process Memory Usage

```bash
# Check memory usage by process
ps aux --sort=-%mem | head -n 20

# Check memory usage by service
systemd-cgtop
```

## Resolution Steps

### 1. Evict Non-Critical Pods

If immediate relief is needed:

```bash
# Identify the highest memory consuming pods on the node
kubectl top pods --all-namespaces --sort-by=memory | grep <node-name>

# Evict a high-memory pod
kubectl delete pod <pod-name> -n <namespace>
```

### 2. Cordon the Node

To prevent new pods from being scheduled:

```bash
kubectl cordon <node-name>
```

### 3. Drain the Node (if necessary)

If the issue requires node maintenance:

```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

### 4. Adjust Pod Resource Limits

For pods with memory issues:

```bash
# Edit the deployment to adjust memory limits
kubectl edit deployment <deployment-name> -n <namespace>
```

Example memory adjustment:

```yaml
resources:
  limits:
    memory: "1Gi"  # Adjust as needed
  requests:
    memory: "512Mi"  # Adjust as needed
```

### 5. Fix System Services

On the node, identify and fix system services consuming excessive memory:

```bash
# Restart problematic services
sudo systemctl restart <service-name>

# Adjust service configuration to limit memory usage
sudo vi /etc/systemd/system/<service-name>.service
# Add: MemoryLimit=1G to the [Service] section
sudo systemctl daemon-reload
sudo systemctl restart <service-name>
```

### 6. Clear Caches

If cache is consuming too much memory:

```bash
# Drop caches
sudo sh -c "sync; echo 3 > /proc/sys/vm/drop_caches"
```

### 7. Restart Container Runtime

If the container runtime is consuming excessive memory:

```bash
# For Docker
sudo systemctl restart docker

# For containerd
sudo systemctl restart containerd
```

### 8. Increase Node Memory

If the node consistently experiences memory pressure:

* Scale up the node memory (cloud provider console or physical hardware)
* Replace the node with a larger instance type
* Adjust memory reservation for system services

```bash
# Adjust kubelet configuration to reserve more memory for system
sudo vi /var/lib/kubelet/config.yaml
# Modify or add:
# systemReserved:
#   memory: "1Gi"
sudo systemctl restart kubelet
```

### 9. Uncordon the Node

After resolving the issue:

```bash
kubectl uncordon <node-name>
```

## Prevention

1. **Set Appropriate Memory Limits**: Configure appropriate memory limits for all pods
2. **Memory Monitoring**: Implement memory usage monitoring and alerting
3. **Regular Node Maintenance**: Schedule regular node maintenance to clear caches and restart services
4. **Implement Resource Quotas**: Use namespace resource quotas to limit memory usage
5. **Use Cluster Autoscaling**: Implement autoscaling to handle varying memory loads
6. **Consider Memory QoS**: Configure memory quality of service settings in kubelet
7. **Memory Overcommit Settings**: Adjust memory overcommit settings for your workload
8. **Optimize Application Memory Usage**: Work with developers to optimize application memory consumption

## Related Runbooks

* [Node Not Ready](./node-not-ready.md)
* [Node CPU Pressure](./node-cpu-pressure.md)
* [Node Disk Pressure](./node-disk-pressure.md)
* [Pod OOMKilled](../workloads/pod-oomkilled.md)
* [Kubelet Issues](./kubelet-issues.md)