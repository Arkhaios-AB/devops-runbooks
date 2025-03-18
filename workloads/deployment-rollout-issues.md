# Troubleshooting Deployment Rollout Issues

## Symptoms

* Deployment stuck in rolling update
* `kubectl rollout status` shows that rollout is not progressing
* Deployment showing partial completion (e.g., "3/5 replicas updated")
* New pods not getting created or terminating correctly
* Rollback operations failing
* Old pods not terminating while new pods are created
* Deployments with "ProgressDeadlineExceeded" condition

## Possible Causes

1. **Pod Startup Issues**: New pods failing health checks or having startup problems
2. **Resource Constraints**: Insufficient resources to schedule new pods
3. **Pod Disruption Budget**: PDB blocking pod termination
4. **Readiness Probe Failures**: New pods failing readiness probes
5. **Rollout Strategy Issues**: Problematic rollout strategy configuration
6. **Node Scheduling Problems**: Nodes unable to accept new pods
7. **Volume Issues**: Problems with persistent volumes during rollout
8. **Progress Deadline Too Short**: Deployment progress deadline too short for application startup
9. **Update Triggers**: Unnecessary pod updates due to improper triggers
10. **Init Container Problems**: Init containers failing in new pod versions

## Diagnosis Steps

### 1. Check Deployment Status

```bash
# Get deployment status
kubectl get deployment <deployment-name> -n <namespace>

# Check rollout status
kubectl rollout status deployment/<deployment-name> -n <namespace>

# Check deployment conditions
kubectl get deployment <deployment-name> -n <namespace> -o jsonpath='{.status.conditions}'
```

### 2. Check ReplicaSets

```bash
# List ReplicaSets for the deployment
kubectl get rs -n <namespace> -l app=<app-label>

# Check details of the new ReplicaSet
kubectl describe rs <new-replicaset> -n <namespace>
```

### 3. Check Pod Status and Events

```bash
# List pods for the deployment
kubectl get pods -n <namespace> -l app=<app-label>

# Check events for problematic pods
kubectl describe pod <pod-name> -n <namespace>

# Check logs for new pods
kubectl logs <pod-name> -n <namespace>
```

### 4. Check Resource Availability

```bash
# Check node resource allocation
kubectl describe nodes | grep -A 5 "Allocated resources"

# Check namespace resource quotas
kubectl get resourcequota -n <namespace>
```

### 5. Check Pod Disruption Budgets

```bash
# List PDBs in the namespace
kubectl get poddisruptionbudgets -n <namespace>

# Check if PDB is blocking rollout
kubectl describe poddisruptionbudget <pdb-name> -n <namespace>
```

### 6. Check Rollout Strategy

```bash
# Check deployment rollout strategy
kubectl get deployment <deployment-name> -n <namespace> -o jsonpath='{.spec.strategy}'
```

## Resolution Steps

### 1. Fix Pod Startup Issues

If new pods are failing to start:

```bash
# Check pod logs
kubectl logs <pod-name> -n <namespace>

# Fix application issues and update the deployment
kubectl edit deployment <deployment-name> -n <namespace>
```

### 2. Adjust Readiness Probe Settings

If readiness probes are failing or timeout is too short:

```bash
# Edit deployment readiness probe settings
kubectl edit deployment <deployment-name> -n <namespace>
```

Example of more lenient readiness probe:
```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30  # Increase delay
  timeoutSeconds: 5
  periodSeconds: 10
  failureThreshold: 3
```

### 3. Fix Resource Constraints

If the cluster lacks resources for new pods:

```bash
# Scale down other non-critical deployments
kubectl scale deployment <other-deployment> --replicas=1 -n <namespace>

# Or modify resource requests in the deployment
kubectl edit deployment <deployment-name> -n <namespace>
```

Example resource adjustment:
```yaml
resources:
  requests:
    memory: "128Mi"  # Decrease from higher value
    cpu: "100m"      # Decrease from higher value
```

### 4. Modify Pod Disruption Budget

If PDB is too restrictive:

```bash
# Edit PDB to be more permissive
kubectl edit poddisruptionbudget <pdb-name> -n <namespace>
```

Example of more permissive PDB:
```yaml
spec:
  maxUnavailable: 1  # Allow at least one pod to be unavailable
  # or
  minAvailable: 50%  # Ensure at least 50% of pods are available
```

### 5. Adjust Rollout Strategy

If the rollout strategy is problematic:

```bash
# Edit deployment strategy
kubectl edit deployment <deployment-name> -n <namespace>
```

Example of a more conservative strategy:
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # Only create 1 extra pod at a time
    maxUnavailable: 0  # Don't allow any unavailable pods
```

Or for a more aggressive strategy:
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%       # Create up to 25% extra pods
    maxUnavailable: 25% # Allow up to 25% pods to be unavailable
```

### 6. Extend Progress Deadline

If the deployment exceeds its progress deadline:

```bash
# Edit progress deadline
kubectl patch deployment <deployment-name> -n <namespace> \
  -p '{"spec":{"progressDeadlineSeconds":600}}'  # Increase to 10 minutes
```

### 7. Force Rollout or Rollback

For stuck rollouts:

```bash
# Restart the rollout
kubectl rollout restart deployment/<deployment-name> -n <namespace>

# Or rollback to a previous revision
kubectl rollout undo deployment/<deployment-name> -n <namespace>

# To a specific revision
kubectl rollout undo deployment/<deployment-name> -n <namespace> --to-revision=<revision>
```

### 8. Fix Update Triggers

If pods are being recreated unnecessarily:

```bash
# Edit deployment to fix update triggers
kubectl edit deployment <deployment-name> -n <namespace>
```

Example of fixing triggers:
```yaml
spec:
  template:
    metadata:
      annotations:
        # Remove automatic triggers like timestamps or random values
        # kubernetes.io/change-cause: "Changed at Thu Aug 24 14:30:21 UTC 2023"
```

### 9. Use Blue-Green Deployment for Critical Services

For critical services, consider blue-green instead of rolling updates:

```bash
# Create a new deployment with a different name
kubectl apply -f new-deployment.yaml

# Once new deployment is ready, switch service to it
kubectl patch service <service-name> -n <namespace> \
  -p '{"spec":{"selector":{"app":"<new-app-label>"}}}'
```

### 10. Fix Init Container Issues

If init containers are failing in new pods:

```bash
# Check init container logs
kubectl logs <pod-name> -c <init-container-name> -n <namespace>

# Fix init container configuration
kubectl edit deployment <deployment-name> -n <namespace>
```

## Prevention

1. **Proactive Testing**: Test deployments in non-production environments
2. **Resource Planning**: Ensure adequate resources for both old and new pods
3. **Proper Probes**: Configure appropriate liveness and readiness probes
4. **Conservative Strategy**: Use conservative rollout strategies for critical services
5. **PDB Configuration**: Configure PDBs to allow for successful rollouts
6. **Monitoring**: Monitor deployment rollouts and set alerts for stuck rollouts
7. **Canary Deployments**: Use canary deployments for high-risk changes
8. **Document Rollout Procedures**: Document standard procedures for deployments
9. **Version Control**: Maintain version control for deployment configurations
10. **Rollback Planning**: Have clear rollback procedures for failed deployments

## Related Runbooks

* [Pod CrashLoopBackOff](./pod-crashloopbackoff.md)
* [Pod OOMKilled](./pod-oomkilled.md)
* [Pod Stuck in Pending](./pod-pending.md)
* [Pod Init Container Errors](./pod-init-container-errors.md)
* [Resource Quota Issues](../resources/resource-quota-issues.md)
* [StatefulSet Issues](./statefulset-issues.md)