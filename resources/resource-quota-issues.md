# Troubleshooting Resource Quota Issues

## Symptoms

* Pod creation failures with messages like "exceeded quota"
* Deployments stuck without reaching desired replicas
* Error messages about exceeding CPU, memory, or pod count limits
* New resources (PVCs, Services, etc.) cannot be created in a namespace
* Requests to the API server failing with quota-related errors
* Events showing resource quota exceeded
* Scale-up operations failing due to quota restrictions

## Possible Causes

1. **Namespace Quota Exhausted**: Resource limits reached in namespace
2. **Missing Resource Requests/Limits**: Required fields missing in pod specs
3. **LimitRange Conflicts**: Conflicts with LimitRange policy
4. **Over-allocation**: Resources over-allocated to individual pods
5. **Insufficient Cluster Resources**: Cluster lacks resources to satisfy quotas
6. **Quota Scope Mismatch**: Resource quota scopes not aligning with pod specs
7. **Hidden Resource Consumption**: Resources consumed by system objects or terminating pods
8. **Incorrect Quota Configuration**: Resource quotas misconfigured
9. **Multiple Quota Objects**: Conflicting quota settings in multiple objects
10. **Job/CronJob Accumulation**: Completed jobs accumulating and consuming quota

## Diagnosis Steps

### 1. Check Resource Quota Status

```bash
# List resource quotas in a namespace
kubectl get resourcequota -n <namespace>

# Check detailed quota usage
kubectl describe resourcequota <quota-name> -n <namespace>
```

### 2. Check LimitRange Configuration

```bash
# List LimitRange objects
kubectl get limitrange -n <namespace>

# Check detailed LimitRange settings
kubectl describe limitrange <limitrange-name> -n <namespace>
```

### 3. Check Pod Resource Requirements

```bash
# Check resource requests and limits across all pods
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].resources}{"\n"}{end}'

# Sum up resources across a namespace
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.resources.requests.cpu}{"\n"}{end}{end}' | awk '{s+=$1} END {print s}'
```

### 4. Check Failed Deployments/Pods

```bash
# Check deployments not reaching desired replicas
kubectl get deployments -n <namespace>

# Check recent events for quota-related failures
kubectl get events -n <namespace> | grep -i quota
```

### 5. Check Cluster Resource Capacity

```bash
# Check node capacity
kubectl describe nodes | grep -A 5 "Capacity"

# Check allocatable resources
kubectl describe nodes | grep -A 5 "Allocatable"
```

### 6. Check for Terminating Resources

```bash
# Check for pods stuck in Terminating state
kubectl get pods -n <namespace> | grep Terminating

# Check for PVCs in Terminating state
kubectl get pvc -n <namespace> | grep Terminating
```

## Resolution Steps

### 1. Increase Resource Quota

If the quota is too restrictive:

```bash
# Update ResourceQuota with higher limits
kubectl edit resourcequota <quota-name> -n <namespace>
```

Example of increased quota:
```yaml
spec:
  hard:
    pods: "20"
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
```

### 2. Fix Missing Resource Requests/Limits

If pods are missing required fields:

```bash
# Edit deployment to add resource requests/limits
kubectl edit deployment <deployment-name> -n <namespace>
```

Example resource configuration:
```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "200m"
```

### 3. Adjust LimitRange

If LimitRange is causing issues:

```bash
# Edit LimitRange to be compatible with ResourceQuota
kubectl edit limitrange <limitrange-name> -n <namespace>
```

Example adjustment:
```yaml
spec:
  limits:
  - type: Container
    default:
      memory: "256Mi"
      cpu: "200m"
    defaultRequest:
      memory: "128Mi"
      cpu: "100m"
    max:
      memory: "1Gi"
      cpu: "1"
    min:
      memory: "64Mi"
      cpu: "50m"
```

### 4. Optimize Resource Allocation

If resources are over-allocated:

```bash
# Adjust resource requests to be more efficient
kubectl edit deployment <deployment-name> -n <namespace>
```

Example of more efficient allocation:
```yaml
resources:
  requests:
    # Request only what's needed, not "just in case" allocations
    memory: "128Mi"  # Reduced from higher value
    cpu: "100m"      # Reduced from higher value
```

### 5. Clean Up Unused Resources

If quota is consumed by unused resources:

```bash
# Delete unused deployments
kubectl delete deployment <unused-deployment> -n <namespace>

# Remove completed jobs
kubectl delete jobs -n <namespace> --field-selector status.successful=1

# Delete failed pods
kubectl delete pods -n <namespace> --field-selector status.phase=Failed
```

### 6. Fix Stuck Terminating Resources

If resources are stuck in Terminating state:

```bash
# Force delete stuck pods
kubectl delete pod <pod-name> -n <namespace> --force --grace-period=0

# Force delete stuck PVCs
kubectl patch pvc <pvc-name> -n <namespace> -p '{"metadata":{"finalizers":null}}'
```

### 7. Create Additional Namespaces

If a single namespace is too constrained:

```bash
# Create a new namespace with appropriate quotas
kubectl create namespace <new-namespace>

# Apply resource quota to the new namespace
kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: <new-namespace>
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
EOF
```

### 8. Use Multiple Quota Scopes

If different types of workloads need different quotas:

```bash
# Create scoped resource quotas
kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: besteffort-quota
  namespace: <namespace>
spec:
  hard:
    pods: "5"
  scopes:
  - BestEffort
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: non-besteffort-quota
  namespace: <namespace>
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: 8Gi
  scopes:
  - NotBestEffort
EOF
```

### 9. Fix Job/CronJob Cleanup

If completed jobs are accumulating:

```bash
# Configure job cleanup
kubectl edit cronjob <cronjob-name> -n <namespace>
```

Add job history limits:
```yaml
spec:
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
```

### 10. Implement Vertical Pod Autoscaler

For efficient resource usage:

```bash
# Install Vertical Pod Autoscaler
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/vertical-pod-autoscaler-0.9.2/vpa-v1-crd.yaml
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/vertical-pod-autoscaler-0.9.2/vpa-rbac.yaml
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/vertical-pod-autoscaler-0.9.2/vpa-v1-deployment.yaml

# Create VPA for a deployment
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

1. **Resource Planning**: Plan namespace resource quotas based on workload requirements
2. **Default Quotas**: Set default quotas for all namespaces
3. **Monitoring**: Monitor quota usage and set alerts for high utilization
4. **Standardized Requests/Limits**: Establish standards for resource requests and limits
5. **Documentation**: Document quota policies and procedures
6. **Resource Efficiency**: Regularly review and optimize resource usage
7. **Implement LimitRanges**: Use LimitRanges to prevent over-allocation
8. **Hierarchical Quotas**: Consider using hierarchical namespace controllers for nested quotas
9. **Quota Enforcement**: Enforce quotas at deployment time via CI/CD
10. **Regular Cleanup**: Implement regular cleanup of completed jobs and failed pods

## Related Runbooks

* [LimitRange Issues](./limit-range-issues.md)
* [Pod Stuck in Pending](../workloads/pod-pending.md)
* [Pod OOMKilled](../workloads/pod-oomkilled.md)
* [HPA Issues](./hpa-issues.md)
* [VPA Issues](./vpa-issues.md)
* [Deployment Rollout Issues](../workloads/deployment-rollout-issues.md)