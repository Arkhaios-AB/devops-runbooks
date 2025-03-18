# Troubleshooting Pod OOMKilled

## Symptoms

* Pods suddenly terminate and restart
* Container status shows "OOMKilled" in pod description
* Container exit code is 137
* Application logs suddenly cut off
* Container restart count increases
* Error messages in kubelet logs about killing containers

## Possible Causes

1. **Insufficient Memory Limits**: Container memory limits set too low
2. **Memory Leaks**: Application has memory leaks
3. **Memory Spikes**: Sudden spikes in memory usage
4. **JVM Configuration**: Incorrect Java heap size settings
5. **Node Memory Pressure**: Node running out of memory
6. **Competing Workloads**: Multiple high-memory workloads on same node
7. **Incorrect Resource Requests**: Memory requests set too low
8. **Cgroup Enforcement**: Container exceeding cgroup memory limits
9. **Application Bug**: Application code issues leading to high memory usage
10. **Kubernetes Overhead**: System overhead not accounted for in limits

## Diagnosis Steps

### 1. Confirm OOMKilled Status

```bash
# Check pod status
kubectl get pod <pod-name> -n <namespace>

# Check pod description for OOMKilled
kubectl describe pod <pod-name> -n <namespace> | grep -A 10 "Last State"
```

Look for:
```
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
```

### 2. Check Container Memory Limits

```bash
# Check pod memory limits
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[0].resources}'
```

### 3. Check Memory Usage Before OOM

```bash
# Check memory usage metrics if available
kubectl top pod <pod-name> -n <namespace>

# If using Prometheus, query for memory usage before OOM event
# Example query: container_memory_usage_bytes{pod="<pod-name>"}
```

### 4. Check Node Memory Status

```bash
# Check node memory status
kubectl describe node <node-name> | grep -A 5 "Allocated resources"

# Check memory pressure conditions
kubectl describe node <node-name> | grep MemoryPressure
```

### 5. Check Application Logs

```bash
# Get logs from the pod before it was killed
kubectl logs <pod-name> -n <namespace> --previous

# Check for memory-related error messages
kubectl logs <pod-name> -n <namespace> --previous | grep -i "memory"
```

### 6. Check Kubelet Logs

SSH to the node and check kubelet logs:

```bash
sudo journalctl -u kubelet | grep -i "killed process" | grep <pod-name>
```

## Resolution Steps

### 1. Increase Memory Limits

If memory limits are too low:

```bash
# Edit deployment to increase memory limits
kubectl edit deployment <deployment-name> -n <namespace>
```

Example memory settings:
```yaml
resources:
  limits:
    memory: "1Gi"  # Increase based on actual usage
  requests:
    memory: "512Mi"  # Set to about 50% of limits
```

### 2. Fix Application Memory Leaks

For applications with memory leaks:

* Add memory profiling to the application
* Identify and fix memory leaks in the code
* Implement proper resource cleanup
* Consider using a memory profiler like jmap for Java applications

### 3. Configure JVM Memory Settings

For Java applications:

```yaml
env:
- name: JAVA_OPTS
  value: "-Xmx512m -Xms256m"  # Adjust based on container limits
```

Make sure the JVM max heap size (-Xmx) is set lower than the container memory limit.

### 4. Adjust Requests to Match Workload

If memory requests are too low or too high:

```bash
# Analyze actual memory usage patterns
# Then edit deployment with appropriate values
kubectl edit deployment <deployment-name> -n <namespace>
```

Set memory requests to typical usage and limits to peak usage plus buffer.

### 5. Isolate High-Memory Workloads

Use node selectors or taints to separate high-memory workloads:

```yaml
nodeSelector:
  memory-tier: high

# And on the node:
kubectl label node <node-name> memory-tier=high
```

### 6. Implement Memory Circuit Breakers

For applications that support memory circuit breakers:

```yaml
env:
- name: MAX_CACHE_SIZE
  value: "100MB"
- name: ENABLE_CIRCUIT_BREAKER
  value: "true"
```

### 7. Horizontal Scaling

If possible, scale horizontally instead of vertically:

```bash
# Increase replicas while decreasing per-pod memory
kubectl scale deployment <deployment-name> --replicas=3 -n <namespace>
```

### 8. Implement Memory Monitoring

Set up proper monitoring and alerting for memory usage:

* Prometheus + Grafana for visualizing memory trends
* Alerts for pods approaching memory limits
* Trend analysis to predict OOM events

### 9. Node Memory Management

If node memory pressure is the issue:

```bash
# Spread workloads across more nodes
kubectl taint node <node-name> workload-type=memory:NoSchedule

# Optimize system reserved memory
# Edit kubelet configuration to reserve more memory for system
```

### 10. Use Vertical Pod Autoscaler

Consider implementing Vertical Pod Autoscaler to automatically adjust resources:

```bash
# Install VPA
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/vertical-pod-autoscaler-0.9.2/vpa-v1-crd.yaml
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/vertical-pod-autoscaler-0.9.2/vpa-rbac.yaml
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/vertical-pod-autoscaler-0.9.2/vpa-v1-deployment.yaml

# Create VPA for your deployment
kubectl apply -f - <<EOF
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: <deployment-name>-vpa
  namespace: <namespace>
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: <deployment-name>
  updatePolicy:
    updateMode: "Auto"
EOF
```

## Prevention

1. **Memory Profiling**: Regularly profile applications for memory usage
2. **Resource Tuning**: Properly tune memory requests and limits
3. **Monitoring**: Implement memory usage monitoring and alerting
4. **Graceful Degradation**: Design applications to degrade gracefully under memory pressure
5. **Load Testing**: Test applications under memory pressure before production
6. **Memory Analysis**: Regularly analyze memory usage patterns
7. **Container Optimization**: Optimize container images and runtime
8. **Documentation**: Document memory requirements and OOM troubleshooting
9. **Resource Quotas**: Implement namespace resource quotas
10. **Caching Strategy**: Implement efficient caching strategies with memory limits

## Related Runbooks

* [Node Memory Pressure](../cluster/node-memory-pressure.md)
* [Pod CrashLoopBackOff](./pod-crashloopbackoff.md)
* [Resource Quota Issues](../resources/resource-quota-issues.md)
* [VPA Issues](../resources/vpa-issues.md)
* [Limit Range Issues](../resources/limit-range-issues.md)