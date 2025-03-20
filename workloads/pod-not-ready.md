# Pod Not Ready Troubleshooting

When a pod is not transitioning to the Ready state, follow these steps to diagnose and resolve the issue.

## Symptoms

- Pod status shows "Not Ready"
- Pod is running but the readiness probe is failing
- Service cannot route traffic to the pod

## Investigation Steps

### 1. Check Pod Status and Events

```bash
# Get detailed pod information
kubectl describe pod <pod-name> -n <namespace>
```

Look for:
- Events section for errors
- Readiness probe configuration and failures
- Container status

### 2. Check Pod Logs

```bash
# Check logs for the specific container
kubectl logs <pod-name> -c <container-name> -n <namespace>

# If the container has restarted, check previous logs
kubectl logs <pod-name> -c <container-name> -n <namespace> --previous
```

### 3. Examine Readiness Probe Configuration

Verify the readiness probe configuration is correct:
- Endpoint exists and is accessible
- Timeout values are appropriate
- Port is correct

### 4. Check Resource Constraints

```bash
# Check node resource usage
kubectl top nodes

# Check pod resource usage
kubectl top pod <pod-name> -n <namespace>
```

Ensure the pod has sufficient resources to start properly.

### 5. Verify Network Connectivity

```bash
# Execute commands inside the pod
kubectl exec -it <pod-name> -n <namespace> -- <command>

# Example: Test endpoint that readiness probe uses
kubectl exec -it <pod-name> -n <namespace> -- curl -v localhost:<port>/health
```

### 6. Check Dependencies

Verify that any services the pod depends on (databases, APIs, etc.) are available.

## Common Causes and Solutions

### Readiness Probe Misconfiguration

**Problem**: Probe is checking the wrong port, path, or has incorrect timing parameters.

**Solution**: 
- Update probe configuration to use correct endpoint and parameters
- Ensure the health check endpoint is implemented correctly

### Application Not Starting Properly

**Problem**: Application inside the container is failing to initialize.

**Solution**:
- Check application logs for errors
- Verify environment variables and configurations
- Ensure startup dependencies are available

### Resource Constraints

**Problem**: Pod doesn't have enough CPU or memory to start properly.

**Solution**:
- Increase resource limits/requests
- Check for resource contention on the node

### External Dependencies Unavailable

**Problem**: Application can't connect to required external services.

**Solution**:
- Verify connectivity to dependent services
- Check for network policies that might be blocking traffic

### Container Image Issues

**Problem**: Container image has problems or application was built incorrectly.

**Solution**:
- Validate container image works locally
- Check for recent changes to the build process

## Prevention

- Implement comprehensive readiness probes
- Set appropriate resource requests/limits
- Document application dependencies
- Use dependency health checks in startup procedures

## Related Runbooks

- [Pod Stuck in Pending](./pod-pending.md)
- [Pod CrashLoopBackOff](./pod-crashloopbackoff.md)
- [Pod Init Container Errors](./pod-init-container-errors.md)
- [Node Not Ready](../cluster/node-not-ready.md)
- [Resource Quota Issues](../resources/resource-quota-issues.md)