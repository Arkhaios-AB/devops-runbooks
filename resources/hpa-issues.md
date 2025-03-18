# Troubleshooting Horizontal Pod Autoscaler (HPA) Issues

## Symptoms

* Pods do not scale up despite increased load
* Pods do not scale down after load decreases
* HPA status shows "unknown" metrics
* HPA is configured but has no effect
* Scaling happens too slowly or too aggressively
* HPA shows errors in `kubectl describe hpa` output

## Possible Causes

1. **Metrics Server Issues**: Metrics Server not deployed or not working correctly
2. **Custom Metrics Issues**: Problems with Prometheus Adapter or other custom metrics sources
3. **Resource Limits Not Defined**: Container resource requests/limits not properly configured
4. **Target Resource Issues**: Target deployment, statefulset, or other resource has issues
5. **Misconfigured HPA**: Incorrect HPA settings (min/max replicas, target values)
6. **Authorization Issues**: HPA lacks permissions to access metrics
7. **Pod Disruption Budget Conflicts**: PDBs preventing scale down
8. **Cluster Autoscaler Issues**: Cluster not providing enough nodes for scale up

## Diagnosis Steps

### 1. Check HPA Status

```bash
# List all HPAs
kubectl get hpa -A

# Get detailed information about the HPA
kubectl describe hpa <hpa-name> -n <namespace>

# Check HPA current metrics status
kubectl get hpa <hpa-name> -n <namespace> -o yaml
```

### 2. Check Metrics Server

```bash
# Check if metrics-server is running
kubectl get pods -n kube-system | grep metrics-server

# Check metrics-server logs
kubectl logs -n kube-system $(kubectl get pods -n kube-system | grep metrics-server | awk '{print $1}')

# Check if node metrics are available
kubectl top nodes

# Check if pod metrics are available
kubectl top pods -n <namespace>
```

### 3. Check Target Workload

```bash
# Check the target resource
kubectl describe deployment <deployment-name> -n <namespace>

# Check pod resource configuration
kubectl get pods -n <namespace> -l <selector> -o jsonpath='{.items[0].spec.containers[0].resources}'
```

### 4. Check Events

```bash
# Check events related to HPA
kubectl get events -n <namespace> | grep <hpa-name>

# Check events related to the target resource
kubectl get events -n <namespace> | grep <deployment-name>
```

## Resolution Steps

### 1. Fix Metrics Server Issues

If Metrics Server is not installed or not working:

```bash
# Install metrics-server (if not installed)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Restart metrics-server (if not working)
kubectl rollout restart deployment metrics-server -n kube-system
```

For metrics-server configuration issues, edit the deployment:

```bash
kubectl edit deployment metrics-server -n kube-system
```

Common fixes include adding these arguments:

```yaml
args:
- --kubelet-insecure-tls
- --kubelet-preferred-address-types=InternalIP
```

### 2. Fix Custom Metrics Issues

If using custom metrics with Prometheus:

```bash
# Check Prometheus adapter installation
kubectl get pods -n monitoring | grep adapter

# Fix Prometheus adapter configuration
kubectl edit configmap adapter-config -n monitoring
```

### 3. Fix Resource Configuration

If container resources are not properly defined:

```bash
# Edit the deployment to add resource requests/limits
kubectl edit deployment <deployment-name> -n <namespace>
```

Example resource configuration:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

### 4. Fix HPA Configuration

If HPA settings are incorrect:

```bash
# Edit the HPA
kubectl edit hpa <hpa-name> -n <namespace>
```

Example HPA configuration:

```yaml
spec:
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

For a complete recreation:

```bash
kubectl delete hpa <hpa-name> -n <namespace>
kubectl autoscale deployment <deployment-name> --cpu-percent=70 --min=2 --max=10 -n <namespace>
```

### 5. Fix RBAC Issues

If there are permission issues:

```yaml
# Create necessary RBAC rules
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:hpa-metrics
rules:
- apiGroups: ["metrics.k8s.io"]
  resources: ["pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: hpa-metrics-binding
subjects:
- kind: ServiceAccount
  name: horizontal-pod-autoscaler
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: system:hpa-metrics
  apiGroup: rbac.authorization.k8s.io
EOF
```

### 6. Resolve Pod Disruption Budget Conflicts

If PDBs are preventing scaling:

```bash
# Check PDBs
kubectl get pdb -n <namespace>

# Modify PDB to allow scaling
kubectl edit pdb <pdb-name> -n <namespace>
```

Example PDB adjustment:

```yaml
spec:
  minAvailable: 1  # or use maxUnavailable instead
```

### 7. Adjust Scaling Parameters

For more responsive scaling:

```yaml
# Edit HPA to add behavior rules
spec:
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
```

## Prevention

1. **Monitor HPA Status**: Set up monitoring for HPA metrics and scaling events
2. **Proper Resource Configuration**: Ensure all deployments have appropriate resource requests/limits
3. **HPA Testing**: Regularly test HPA scaling during non-peak hours
4. **Use Stabilization Windows**: Configure appropriate stabilization windows to prevent thrashing
5. **Document Scaling Policies**: Document expected scaling behavior for each application
6. **Cluster Capacity Planning**: Ensure the cluster has enough resources for maximum scale
7. **Use Multiple Metrics**: Configure HPAs with multiple metrics for more robust scaling

## Related Runbooks

* [VPA Issues](./vpa-issues.md)
* [Resource Quota Issues](./resource-quota-issues.md)
* [Limit Range Issues](./limit-range-issues.md)
* [Pod OOMKilled](../workloads/pod-oomkilled.md)
* [Cluster Autoscaling Issues](../cluster/autoscaling-issues.md)