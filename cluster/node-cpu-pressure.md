# Troubleshooting Node CPU Pressure

## Symptoms

* Node becomes slow or unresponsive
* High system load average reported in monitoring
* Pod scheduling delays due to CPU constraints
* Increased application latency
* Kubelet may report CPU throttling in logs
* `kubectl top nodes` shows high CPU utilization

## Possible Causes

1. **High Pod CPU Usage**: One or more pods consuming excessive CPU resources
2. **CPU-Intensive System Processes**: System daemons or services consuming high CPU
3. **Insufficient Node Resources**: Node has inadequate CPU capacity for its workload
4. **Kernel Issues**: Kernel processes consuming excessive CPU
5. **Container Runtime Overhead**: Container runtime consuming high CPU
6. **Runaway Processes**: Processes caught in infinite loops or similar issues
7. **CPU Throttling**: Excessive CPU throttling due to improper limits
8. **Management Processes**: Monitoring, logging, or management agents using excessive CPU

## Diagnosis Steps

### 1. Check Node CPU Status

```bash
# Check CPU usage across nodes
kubectl top nodes

# Get node details
kubectl describe node <node-name>
```

### 2. Check Pod CPU Usage

```bash
# Check CPU usage by pod, sorted highest first
kubectl top pods --all-namespaces --sort-by=cpu

# Filter for pods on the affected node
kubectl get pods --all-namespaces -o wide | grep <node-name>
kubectl top pods --all-namespaces | grep <pod-name>
```

### 3. Check System Process CPU Usage

SSH into the affected node and check CPU usage:

```bash
# Check overall CPU usage
top

# Check CPU usage by process, sorted by CPU usage
ps aux --sort=-%cpu | head -n 20

# Check load average
uptime

# Check detailed CPU stats
mpstat -P ALL

# Check for CPU throttling
cat /sys/fs/cgroup/cpu/cpu.stat
```

### 4. Check Kernel and Interrupt Activity

```bash
# Check kernel process activity
vmstat 1 10

# Check interrupts
cat /proc/interrupts

# Check context switches
vmstat -s | grep "context switches"
```

### 5. Check Container Runtime CPU Usage

```bash
# For Docker
sudo systemctl status docker
ps aux | grep docker

# For containerd
sudo systemctl status containerd
ps aux | grep containerd
```

## Resolution Steps

### 1. Isolate CPU-Intensive Pods

If a specific pod is causing problems:

```bash
# Identify CPU-intensive pods on the node
kubectl top pods --all-namespaces --sort-by=cpu | grep <node-name>

# Evict the problematic pod to another node
kubectl delete pod <pod-name> -n <namespace>
```

### 2. Limit CPU-Intensive Applications

Adjust CPU limits for problematic pods:

```bash
# Edit deployment to adjust CPU limits
kubectl edit deployment <deployment-name> -n <namespace>
```

Example CPU limit adjustment:

```yaml
resources:
  limits:
    cpu: "2"  # Adjust as needed
  requests:
    cpu: "500m"  # Adjust as needed
```

### 3. Cordon the Node

To prevent new pods from being scheduled:

```bash
kubectl cordon <node-name>
```

### 4. Drain the Node (if necessary)

If the issue requires node maintenance:

```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

### 5. Fix System Processes

On the node, identify and fix system processes consuming excessive CPU:

```bash
# Restart problematic services
sudo systemctl restart <service-name>

# Adjust service configuration to limit CPU usage
sudo vi /etc/systemd/system/<service-name>.service
# Add: CPUQuota=50% to the [Service] section
sudo systemctl daemon-reload
sudo systemctl restart <service-name>
```

### 6. Restart Container Runtime

If the container runtime is using excessive CPU:

```bash
# For Docker
sudo systemctl restart docker

# For containerd
sudo systemctl restart containerd
```

### 7. Kill Runaway Processes

For processes stuck in loops:

```bash
# Find the PID of the process
ps aux | grep <process-name>

# Kill the process
sudo kill -9 <pid>
```

### 8. Adjust CPU Throttling Settings

Modify cgroup CPU settings:

```bash
# Check current CPU shares
cat /sys/fs/cgroup/cpu/cpu.shares

# Adjust CPU shares for a specific cgroup
echo 1024 > /sys/fs/cgroup/cpu/system.slice/cpu.shares
```

### 9. Increase Node CPU Capacity

If the node consistently experiences CPU pressure:

* Scale up the node CPU (cloud provider console or physical hardware)
* Replace the node with a larger instance type

### 10. Uncordon the Node

After resolving the issue:

```bash
kubectl uncordon <node-name>
```

## Prevention

1. **Set Appropriate CPU Limits**: Configure appropriate CPU requests and limits for all pods
2. **CPU Monitoring**: Implement CPU usage monitoring and alerting
3. **Use CPU Affinity**: Configure CPU affinity for critical workloads
4. **Implement Resource Quotas**: Use namespace resource quotas to limit CPU usage
5. **Use Cluster Autoscaling**: Implement autoscaling to handle varying CPU loads
6. **Consider CPU Sets**: Use CPU sets to isolate critical workloads
7. **Regular Performance Testing**: Regularly test application performance under load
8. **Optimize System Services**: Minimize CPU usage by system services

## Related Runbooks

* [Node Not Ready](./node-not-ready.md)
* [Node Memory Pressure](./node-memory-pressure.md)
* [Pod CrashLoopBackOff](../workloads/pod-crashloopbackoff.md)
* [Kubelet Issues](./kubelet-issues.md)
* [HPA Issues](../resources/hpa-issues.md)