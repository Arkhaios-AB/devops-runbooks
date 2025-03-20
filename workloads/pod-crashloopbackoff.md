# Troubleshooting Pod CrashLoopBackOff

## Symptoms

* Pods are restarting repeatedly
* `kubectl get pods` shows pods in the `CrashLoopBackOff` state
* `kubectl describe pod <pod-name>` shows a high number of restarts
* Container starts and exits quickly, then Kubernetes attempts to restart it

## Possible Causes

1. **Application Error**: The application inside the container is crashing
2. **Resource Constraints**: Pod doesn't have enough CPU or memory resources
3. **Liveness Probe Failures**: Misconfigured liveness probe causing container restarts
4. **Configuration Issues**: Missing or incorrect environment variables, config maps, or secrets
5. **Dependency Issues**: The pod depends on a service that is not available
6. **Permission Problems**: Container lacks permissions to access required resources
7. **Init Container Failures**: Init containers failing, preventing main containers from starting properly

## Diagnosis Steps

### 1. Check Pod Status and Details

```bash
# Get basic pod information
kubectl get pods -n <namespace>

# Get detailed information about the pod
kubectl describe pod <pod-name> -n <namespace>

# Check events related to the pod
kubectl get events --sort-by=.metadata.creationTimestamp -n <namespace> | grep <pod-name>
```

### 2. Check Pod Logs

```bash
# Check logs from the current instance of the container
kubectl logs <pod-name> -n <namespace>

# Check logs from the previous instance of the container
kubectl logs <pod-name> -n <namespace> --previous

# If the pod has multiple containers, specify the container name
kubectl logs <pod-name> -c <container-name> -n <namespace>
```

### 3. Check Resource Usage

```bash
# Check pod resource usage
kubectl top pod <pod-name> -n <namespace>

# Check node resource usage where the pod is running
kubectl top node <node-name>
```

### 4. Check Configuration

```bash
# Check environment variables
kubectl exec <pod-name> -n <namespace> -- env

# Check mounted ConfigMaps and Secrets
kubectl describe pod <pod-name> -n <namespace> | grep -A10 "Mounts:"
```

## Resolution Steps

### 1. Fix Application Errors

If the logs show application errors:

* Fix the code issue that's causing the crash
* Build and push a new container image
* Update the deployment to use the new image:

```bash
kubectl set image deployment/<deployment-name> <container-name>=<new-image> -n <namespace>
```

### 2. Fix Resource Constraints

If the pod is being terminated due to OOM (Out of Memory) or resource constraints:

```bash
# Edit the deployment to increase resource limits
kubectl edit deployment <deployment-name> -n <namespace>
```

Example resource adjustment in the pod spec:

```yaml
resources:
  limits:
    memory: "512Mi"
    cpu: "500m"
  requests:
    memory: "256Mi"
    cpu: "250m"
```

### 3. Fix Liveness Probe

If the liveness probe is failing:

```bash
# Edit the deployment to fix the liveness probe
kubectl edit deployment <deployment-name> -n <namespace>
```

Example liveness probe adjustment:

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30  # Increase this value
  periodSeconds: 10
  failureThreshold: 3
```

### 4. Fix Configuration Issues

If environment variables or mounted files are missing:

```bash
# Edit the deployment to fix environment variables or volumes
kubectl edit deployment <deployment-name> -n <namespace>
```

### 5. Debug Within a Temporary Pod

Sometimes it's helpful to debug in a similar environment:

```bash
# Run a temporary pod with the same image
kubectl run debug-pod --image=<same-image-as-crashing-pod> -n <namespace> --rm -it -- /bin/sh
```

## Prevention

1. **Implement Proper Testing**: Test applications thoroughly before deployment
2. **Set Appropriate Resource Limits**: Configure memory and CPU limits/requests based on application needs
3. **Use Readiness Probes**: Implement readiness probes to ensure dependencies are available
4. **Implement Graceful Shutdown**: Ensure applications handle termination signals properly
5. **Add Health Checks**: Implement proper health checks for your applications
6. **Log Management**: Implement comprehensive logging to quickly identify issues
7. **Set Appropriate Restart Policies**: Configure appropriate restart policies for your workloads

## Related Runbooks

* [Pod OOMKilled](./pod-oomkilled.md)
* [Pod Not Ready](./pod-not-ready.md)
* [Pod Init Container Errors](./pod-init-container-errors.md)
* [Deployment Rollout Issues](./deployment-rollout-issues.md)
* [Resource Quota Issues](../resources/resource-quota-issues.md)