# Troubleshooting LimitRange Issues

## Symptoms

* Pod creation failures with messages about exceeding LimitRange
* Error messages about missing resource requests or limits
* Pods being rejected during admission
* Containers getting unexpected resource defaults
* OOMKilled pods due to incorrect memory limits
* CPU throttling due to incorrect CPU limits
* LimitRange constraints blocking pod deployments
* Containers unable to start with larger/smaller resource settings than expected
* Inconsistent resource assignment across pods
* Resource request/limit validation errors

## Possible Causes

1. **Misconfigured LimitRange**: LimitRange with incorrect constraints
2. **Missing Resource Specifications**: Pod spec missing required resource requests/limits
3. **LimitRange/ResourceQuota Conflicts**: Conflicts between LimitRange and ResourceQuota
4. **Default Limit Issues**: Default limits too low for application requirements
5. **Min/Max Constraints**: Application requirements outside min/max constraints
6. **Inconsistent LimitRanges**: Different LimitRanges across namespaces causing confusion
7. **Type Mismatch**: LimitRange applied to wrong resource type
8. **Multiple LimitRanges**: Overlapping or conflicting LimitRanges in same namespace
9. **LimitRange Enforcement Issues**: LimitRange not enforced correctly by admission controller
10. **Change in Resource Requirements**: Application resource usage has changed

## Diagnosis Steps

### 1. Check LimitRange Configuration

```bash
# List LimitRanges in the namespace
kubectl get limitrange -n <namespace>

# Check specific LimitRange details
kubectl describe limitrange <limitrange-name> -n <namespace>

# Get LimitRange in YAML format
kubectl get limitrange <limitrange-name> -n <namespace> -o yaml
```

### 2. Check Pod Resource Configuration

```bash
# Examine pod resource specifications
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[*].resources}'

# Check deployment resource specifications
kubectl get deployment <deployment-name> -n <namespace> -o jsonpath='{.spec.template.spec.containers[*].resources}'
```

### 3. Check Pod Creation Errors

```bash
# Check error events
kubectl get events -n <namespace> | grep <pod-name>
kubectl get events -n <namespace> | grep -i limitrange
```

### 4. Check for ResourceQuota Conflicts

```bash
# List ResourceQuotas in the namespace
kubectl get resourcequota -n <namespace>

# Check ResourceQuota details
kubectl describe resourcequota <quota-name> -n <namespace>
```

### 5. Check Container Resource Usage

```bash
# Check actual resource usage
kubectl top pod <pod-name> -n <namespace>

# For detailed resource usage
kubectl describe pod <pod-name> -n <namespace>
```

## Resolution Steps

### 1. Fix Misconfigured LimitRange

If LimitRange is incorrectly configured:

```bash
# Edit the LimitRange
kubectl edit limitrange <limitrange-name> -n <namespace>
```

Example of a properly configured LimitRange:
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: <namespace>
spec:
  limits:
  - type: Container
    default:          # Default limits if not specified
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:   # Default requests if not specified
      cpu: "100m"
      memory: "256Mi"
    max:              # Maximum allowed
      cpu: "2"
      memory: "2Gi"
    min:              # Minimum allowed
      cpu: "50m"
      memory: "64Mi"
  - type: PersistentVolumeClaim
    max:
      storage: "50Gi"
    min:
      storage: "1Gi"
```

### 2. Fix Pod Resource Specifications

If pod resources don't meet LimitRange requirements:

```bash
# Edit deployment to adjust resources
kubectl edit deployment <deployment-name> -n <namespace>
```

Example of proper resource configuration:
```yaml
resources:
  requests:
    memory: "256Mi"  # Must be >= min and <= max
    cpu: "100m"
  limits:
    memory: "512Mi"  # Must be >= min and <= max
    cpu: "500m"
```

### 3. Fix Default Resource Settings

If default resource settings are problematic:

```bash
# Update LimitRange with appropriate defaults
kubectl apply -f - <<EOF
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: <namespace>
spec:
  limits:
  - type: Container
    default:
      cpu: "1"         # Increase default CPU limit
      memory: "1Gi"    # Increase default memory limit
    defaultRequest:
      cpu: "200m"      # Increase default CPU request
      memory: "512Mi"  # Increase default memory request
    max:
      cpu: "4"
      memory: "4Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
EOF
```

### 4. Fix LimitRange/ResourceQuota Conflicts

If LimitRange and ResourceQuota are in conflict:

```bash
# Align ResourceQuota with LimitRange
kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: <namespace>
spec:
  hard:
    requests.cpu: "10"         # Must accommodate LimitRange min * pod count
    requests.memory: "10Gi"    # Must accommodate LimitRange min * pod count
    limits.cpu: "20"           # Must accommodate LimitRange max * pod count
    limits.memory: "20Gi"      # Must accommodate LimitRange max * pod count
    pods: "10"
EOF
```

### 5. Handle Min/Max Constraints

If application requires resources outside min/max constraints:

```bash
# Create separate namespace with appropriate LimitRange
kubectl create namespace <new-namespace>

# Create appropriate LimitRange in new namespace
kubectl apply -f - <<EOF
apiVersion: v1
kind: LimitRange
metadata:
  name: high-resource-limits
  namespace: <new-namespace>
spec:
  limits:
  - type: Container
    default:
      cpu: "2"
      memory: "4Gi"
    defaultRequest:
      cpu: "1"
      memory: "2Gi"
    max:
      cpu: "8"
      memory: "16Gi"
    min:
      cpu: "500m"
      memory: "1Gi"
EOF

# Move workload to new namespace
kubectl get deployment <deployment-name> -n <old-namespace> -o yaml | \
  sed "s/namespace: <old-namespace>/namespace: <new-namespace>/" | \
  kubectl apply -f -
```

### 6. Fix Type Mismatch

If LimitRange is applied to wrong resource type:

```bash
# Create LimitRange with correct type
kubectl apply -f - <<EOF
apiVersion: v1
kind: LimitRange
metadata:
  name: pod-limits
  namespace: <namespace>
spec:
  limits:
  - type: Pod  # For pod-level limits
    max:
      cpu: "4"
      memory: "8Gi"
    min:
      cpu: "200m"
      memory: "512Mi"
EOF
```

Valid types include:
- Container
- Pod
- PersistentVolumeClaim
- ConfigMap
- Secret
- StorageClass

### 7. Fix Multiple LimitRanges

If multiple conflicting LimitRanges exist:

```bash
# List all LimitRanges
kubectl get limitrange -n <namespace>

# Delete redundant LimitRanges
kubectl delete limitrange <redundant-limitrange> -n <namespace>

# Or consolidate into a single LimitRange
kubectl apply -f consolidated-limitrange.yaml
```

### 8. Temporarily Override LimitRange

For testing or emergency situations:

```bash
# Temporarily delete LimitRange (USE WITH CAUTION)
kubectl delete limitrange <limitrange-name> -n <namespace>

# Create workload
kubectl apply -f workload.yaml

# Recreate LimitRange
kubectl apply -f limitrange.yaml
```

### 9. Implement Limit Exemptions for Specific Applications

If specific applications need exemptions:

```bash
# Create dedicated namespace without LimitRange
kubectl create namespace <app-namespace>

# Or use ResourceQuota without LimitRange for more flexible limits
kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: flexible-quota
  namespace: <app-namespace>
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
EOF
```

### 10. Fix Inconsistent Resource Assignment

If containers have inconsistent resource assignments:

```bash
# Create a consistent resource template
cat <<EOF > resource-template.yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"
EOF

# Apply template to all deployments
kubectl get deployment -n <namespace> -o name | xargs -I{} kubectl patch {} -n <namespace> --type=strategic --patch="$(cat <<EOF
spec:
  template:
    spec:
      containers:
      - name: *
        $(cat resource-template.yaml)
EOF
)"
```

## Prevention

1. **Standardize LimitRanges**: Use standardized LimitRanges across namespaces
2. **Documentation**: Document LimitRange policies and resource guidelines
3. **Namespace-based Policies**: Use different namespaces for workloads with different resource requirements
4. **Testing**: Test workloads against LimitRanges before production deployment
5. **Resource Monitoring**: Monitor actual resource usage to refine LimitRanges
6. **Resource Templates**: Create templates for common resource configurations
7. **Regular Review**: Regularly review and update LimitRanges based on workload needs
8. **Education**: Educate developers about resource requirements and constraints
9. **Automation**: Automate resource configuration validation in CI/CD
10. **Tooling**: Use tools to visualize and audit resource configurations

## Related Runbooks

* [Resource Quota Issues](./resource-quota-issues.md)
* [Pod Stuck in Pending](../workloads/pod-pending.md)
* [Pod OOMKilled](../workloads/pod-oomkilled.md)
* [HPA Issues](./hpa-issues.md)
* [VPA Issues](./vpa-issues.md)
* [Deployment Rollout Issues](../workloads/deployment-rollout-issues.md)