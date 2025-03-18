# Troubleshooting Pod Init Container Errors

## Symptoms

* Pods stuck in `Init` state
* `kubectl get pods` shows pod status with `Init:N/M`
* Main containers never start
* Init container exit codes indicating failures
* Init container logs showing errors
* Deployments or StatefulSets not reaching ready state

## Possible Causes

1. **Init Container Logic Failures**: Code or script errors in init containers
2. **Dependency Unavailability**: Services or endpoints that init containers depend on are unavailable
3. **Permission Issues**: Init containers lack necessary permissions
4. **Resource Constraints**: Init containers hitting resource limits
5. **Volume Mount Problems**: Issues with volumes needed by init containers
6. **Network Connectivity**: Init containers unable to reach required services
7. **Image Issues**: Problems with init container images
8. **Configuration Errors**: Misconfigured init container parameters
9. **Timeout Issues**: Init container operations timing out
10. **Secret or ConfigMap Issues**: Missing or invalid secrets/configmaps

## Diagnosis Steps

### 1. Check Pod Init Container Status

```bash
# Get pod status
kubectl get pod <pod-name> -n <namespace>

# Check detailed pod status
kubectl describe pod <pod-name> -n <namespace>

# Check events
kubectl get events -n <namespace> | grep <pod-name>
```

### 2. Examine Init Container Logs

```bash
# Get logs from the init container
kubectl logs <pod-name> -c <init-container-name> -n <namespace>

# If init container has restarted, get previous logs
kubectl logs <pod-name> -c <init-container-name> -n <namespace> --previous
```

### 3. Check Init Container Configuration

```bash
# Get pod specification with init containers
kubectl get pod <pod-name> -n <namespace> -o yaml
```

Look for the `initContainers` section and check:
- Command and arguments
- Environment variables
- Volume mounts
- Resource requests/limits

### 4. Check Dependencies

If init container is waiting for a service:

```bash
# Check if required services exist
kubectl get svc <service-name> -n <namespace>

# Check service endpoints
kubectl get endpoints <service-name> -n <namespace>

# Check if DNS resolution works
kubectl run -it --rm debug --image=busybox:1.28 -- nslookup <service-name>.<namespace>.svc.cluster.local
```

### 5. Check Resource Usage

```bash
# If init container is running but slow, check resource usage
kubectl top pod <pod-name> -n <namespace>
```

## Resolution Steps

### 1. Fix Init Container Logic

If the init container script or application has errors:

```bash
# Edit deployment to fix init container logic
kubectl edit deployment <deployment-name> -n <namespace>
```

Example of proper init container command:
```yaml
initContainers:
- name: init-myservice
  image: busybox:1.28
  command: ['sh', '-c', 'until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice; sleep 2; done;']
```

### 2. Fix Dependency Issues

If the init container is waiting for a service that doesn't exist:

```bash
# Create the missing service
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: myservice
  namespace: <namespace>
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
EOF
```

### 3. Fix Permission Issues

If the init container needs specific permissions:

```bash
# Create a ServiceAccount with proper permissions
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: init-container-sa
  namespace: <namespace>
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: init-container-role
  namespace: <namespace>
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: init-container-rolebinding
  namespace: <namespace>
subjects:
- kind: ServiceAccount
  name: init-container-sa
  namespace: <namespace>
roleRef:
  kind: Role
  name: init-container-role
  apiGroup: rbac.authorization.k8s.io
EOF

# Update deployment to use the ServiceAccount
kubectl patch deployment <deployment-name> -n <namespace> \
  -p '{"spec":{"template":{"spec":{"serviceAccountName":"init-container-sa"}}}}'
```

### 4. Fix Resource Constraints

If the init container is hitting resource limits:

```bash
# Edit deployment to increase resource limits
kubectl edit deployment <deployment-name> -n <namespace>
```

Example resource adjustment:
```yaml
initContainers:
- name: init-myservice
  # other fields...
  resources:
    limits:
      memory: "256Mi"
      cpu: "500m"
    requests:
      memory: "128Mi"
      cpu: "100m"
```

### 5. Fix Volume Mount Issues

If volume mounts are causing problems:

```bash
# Check if volumes exist
kubectl get pvc -n <namespace>

# Fix volume configuration in the deployment
kubectl edit deployment <deployment-name> -n <namespace>
```

Example of proper volume mount:
```yaml
initContainers:
- name: init-myservice
  # other fields...
  volumeMounts:
  - name: config-volume
    mountPath: /config
volumes:
- name: config-volume
  configMap:
    name: my-config
```

### 6. Fix Network Connectivity

If network issues are preventing init containers from connecting to services:

```bash
# Check network policy
kubectl get networkpolicy -n <namespace>

# Create or modify network policy if needed
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-init-container
  namespace: <namespace>
spec:
  podSelector:
    matchLabels:
      app: myapp
  egress:
  - to:
    - namespaceSelector: {}
EOF
```

### 7. Debug with Interactive Init Container

For complex issues, create a debugging pod with the same configuration:

```bash
# Create debug pod with interactive shell
kubectl run debug-init --image=<init-container-image> -n <namespace> -it -- /bin/sh

# Then manually run the init container commands to debug
```

### 8. Configure Proper Init Container Order

If init containers must run in a specific order:

```yaml
initContainers:
- name: init-first
  # This runs first
- name: init-second
  # This runs second, after init-first completes successfully
```

### 9. Add Retry Logic

For flaky dependencies, add retry logic to init container scripts:

```yaml
initContainers:
- name: init-with-retry
  image: busybox:1.28
  command: ['sh', '-c', 'for i in $(seq 1 30); do if command-to-check; then exit 0; fi; echo "Attempt $i failed"; sleep 5; done; exit 1']
```

## Prevention

1. **Simplify Init Containers**: Keep init containers simple and focused
2. **Proper Error Handling**: Implement error handling in init container scripts
3. **Health Checks**: Implement proper health checks for dependencies
4. **Timeout Configuration**: Set appropriate timeouts for init operations
5. **Resource Planning**: Set appropriate resource requests and limits
6. **Testing**: Test init container logic in isolation
7. **Documentation**: Document init container purpose and requirements
8. **Monitoring**: Monitor init container success rate and duration
9. **Graceful Dependency Handling**: Design init containers to handle missing dependencies gracefully
10. **Use Init Container Best Practices**: Follow Kubernetes best practices for init containers

## Related Runbooks

* [Pod Stuck in Pending](./pod-pending.md)
* [Pod Stuck in ContainerCreating](./pod-container-creating.md)
* [Pod CrashLoopBackOff](./pod-crashloopbackoff.md)
* [Service Not Accessible](../networking/service-not-accessible.md)
* [DNS Resolution Problems](../networking/dns-resolution-problems.md)
* [Volume Mount Problems](../storage/volume-mount-problems.md)