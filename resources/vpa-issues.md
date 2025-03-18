# Troubleshooting VPA (Vertical Pod Autoscaler) Issues

## Symptoms

* VPA not recommending resource changes
* VPA recommending inappropriate resource limits
* Pods not being updated with new resource recommendations
* VPA components crashing or showing errors
* Frequent pod restarts due to VPA updates
* OOMKilled pods despite VPA configuration
* Resource recommendations inconsistent or fluctuating
* VPA in "Auto" mode not applying changes
* Error events related to VPA admission controller
* High CPU or memory usage by VPA components

## Possible Causes

1. **VPA Not Properly Installed**: Missing VPA components or incorrect installation
2. **Incorrect VPA Configuration**: Misconfigured VPA resource
3. **Update Mode Issues**: Wrong updatePolicy mode for the workload
4. **Insufficient History**: Not enough resource usage data for recommendations
5. **Resource Constraints**: Cluster resource limitations preventing scaling
6. **Conflicts with HPA**: Conflicts between Vertical and Horizontal Pod Autoscaler
7. **Admission Controller Issues**: VPA admission controller not functioning
8. **Metrics Server Problems**: Metrics not being collected correctly
9. **LimitRange/ResourceQuota Conflicts**: Conflicts with namespace constraints
10. **Pod Selector Mismatch**: VPA targeting wrong pods

## Diagnosis Steps

### 1. Check VPA Installation

```bash
# Check VPA components
kubectl get pods -n kube-system | grep vpa
kubectl get deployment -n kube-system | grep vpa

# Check CRDs
kubectl get crd | grep autoscaling.k8s.io
```

### 2. Check VPA Resources

```bash
# List VPA resources
kubectl get vpa --all-namespaces

# Check specific VPA details
kubectl describe vpa <vpa-name> -n <namespace>

# Get VPA in YAML format
kubectl get vpa <vpa-name> -n <namespace> -o yaml
```

### 3. Check VPA Recommendations

```bash
# Get detailed VPA recommendation status
kubectl describe vpa <vpa-name> -n <namespace> | grep -A 20 "Recommendation"
```

### 4. Check VPA Component Logs

```bash
# Check logs for VPA components
kubectl logs -n kube-system deployment/vpa-recommender
kubectl logs -n kube-system deployment/vpa-updater
kubectl logs -n kube-system deployment/vpa-admission-controller
```

### 5. Check Target Pod Resources

```bash
# Check current pod resources
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[*].resources}'

# Check pod metrics
kubectl top pod <pod-name> -n <namespace>
```

### 6. Check for Conflicting HPA

```bash
# Check if HPA exists for the same workload
kubectl get hpa -n <namespace> | grep <deployment-name>
```

### 7. Check Metrics Server

```bash
# Check metrics-server status
kubectl get pods -n kube-system | grep metrics-server

# Check metrics-server logs
kubectl logs -n kube-system -l k8s-app=metrics-server
```

## Resolution Steps

### 1. Fix VPA Installation

If VPA components are missing or misconfigured:

```bash
# Install or reinstall VPA
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/vertical-pod-autoscaler-0.10.0/vpa-v1-crd.yaml
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/vertical-pod-autoscaler-0.10.0/vpa-rbac.yaml
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/vertical-pod-autoscaler-0.10.0/vpa-v1-deployment.yaml
```

### 2. Fix VPA Configuration

If VPA is misconfigured:

```bash
# Create or update VPA resource
kubectl apply -f - <<EOF
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: <vpa-name>
  namespace: <namespace>
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: <deployment-name>
  updatePolicy:
    updateMode: "Auto"  # Or "Off", "Initial", "Recreate"
  resourcePolicy:
    containerPolicies:
    - containerName: '*'
      minAllowed:
        cpu: "50m"
        memory: "64Mi"
      maxAllowed:
        cpu: "4"
        memory: "8Gi"
      controlledResources: ["cpu", "memory"]
EOF
```

### 3. Fix Update Mode

Choose the appropriate update mode based on your needs:

```bash
# Update VPA with appropriate mode
kubectl patch vpa <vpa-name> -n <namespace> --type=merge \
  -p '{"spec":{"updatePolicy":{"updateMode":"Auto"}}}'
```

Available update modes:
- `Off`: VPA only provides recommendations, doesn't apply changes
- `Initial`: VPA applies recommendations only at pod creation time
- `Auto`: VPA automatically applies recommendations by evicting and recreating pods
- `Recreate`: VPA applies recommendations by recreating pods, but never evicts them

### 4. Wait for Sufficient History

If there's insufficient history:

```bash
# Check recommendation age
kubectl describe vpa <vpa-name> -n <namespace> | grep "Last recommendation"

# Force workload to generate more metrics
kubectl run load-generator --image=busybox -- /bin/sh -c "while true; do wget -q -O- http://<service-name>.<namespace>; sleep 0.1; done"
```

Allow VPA to collect data for at least 30 minutes for stable recommendations.

### 5. Fix Resource Constraints

If cluster resources are preventing scaling:

```bash
# Check node resource allocation
kubectl describe nodes | grep -A 5 "Allocated resources"

# Add nodes if necessary (cloud provider specific)
# For example, in AWS EKS:
eksctl scale nodegroup --cluster=<cluster-name> --nodes=<desired-nodes> --name=<nodegroup-name>
```

### 6. Fix HPA and VPA Conflicts

Configure VPA and HPA to work together:

```bash
# Configure VPA to only control memory
kubectl apply -f - <<EOF
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: <vpa-name>
  namespace: <namespace>
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: <deployment-name>
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: '*'
      controlledResources: ["memory"]  # Only control memory, let HPA handle CPU
EOF

# Configure HPA for CPU scaling
kubectl apply -f - <<EOF
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: <hpa-name>
  namespace: <namespace>
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: <deployment-name>
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
EOF
```

### 7. Fix Admission Controller Issues

If VPA admission controller is not working:

```bash
# Check admission controller status
kubectl get pods -n kube-system | grep vpa-admission-controller

# Restart the admission controller
kubectl rollout restart deployment/vpa-admission-controller -n kube-system

# Check if webhook is registered
kubectl get validatingwebhookconfiguration | grep vpa

# Check webhook details
kubectl describe validatingwebhookconfiguration vpa-webhook-config
```

### 8. Fix Metrics Server Issues

If metrics-server is not providing data:

```bash
# Restart metrics-server
kubectl rollout restart deployment metrics-server -n kube-system

# Check if metrics are available
kubectl top pods --all-namespaces
```

### 9. Fix LimitRange and ResourceQuota Conflicts

If LimitRange or ResourceQuota is blocking VPA:

```bash
# Modify LimitRange to allow VPA range
kubectl apply -f - <<EOF
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: <namespace>
spec:
  limits:
  - type: Container
    max:
      cpu: "8"       # Ensure max is high enough for VPA
      memory: "16Gi"
    min:
      cpu: "50m"     # Ensure min is low enough for VPA
      memory: "64Mi"
EOF

# Update ResourceQuota to allow VPA range
kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: <namespace>
spec:
  hard:
    requests.cpu: "20"        # Ensure high enough for VPA increases
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
EOF
```

### 10. Fix Pod Selector Mismatch

If VPA is targeting wrong pods:

```bash
# Check pod labels
kubectl get pods -n <namespace> --show-labels

# Update VPA targetRef
kubectl edit vpa <vpa-name> -n <namespace>
```

Fix the targetRef:
```yaml
targetRef:
  apiVersion: "apps/v1"
  kind: Deployment  # Make sure kind is correct (Deployment, StatefulSet, etc.)
  name: <correct-deployment-name>
```

### 11. Force VPA Update

For testing or immediate application of recommendations:

```bash
# Delete pods to force recreation with new recommendations
kubectl delete pods -n <namespace> -l <pod-selector>
```

For VPA in "Auto" mode, pods will be recreated automatically.

## Prevention

1. **Start with Off Mode**: Begin with VPA in "Off" mode to review recommendations
2. **Gradual Changes**: Use Initial mode before Auto for critical workloads
3. **Set Min/Max Bounds**: Configure minAllowed and maxAllowed to prevent extreme changes
4. **Monitor Recommendations**: Regularly review VPA recommendations
5. **Restart Schedule**: Schedule restarts during low-traffic periods
6. **Proper HPA Integration**: Coordinate VPA and HPA settings
7. **Resource Monitoring**: Implement monitoring for actual resource usage
8. **Testing**: Test VPA in non-production before enabling in production
9. **Backup Configurations**: Keep backups of pre-VPA resource configurations
10. **Documentation**: Document VPA policies and expected behavior

## Related Runbooks

* [HPA Issues](./hpa-issues.md)
* [Resource Quota Issues](./resource-quota-issues.md)
* [Limit Range Issues](./limit-range-issues.md)
* [Pod OOMKilled](../workloads/pod-oomkilled.md)
* [Pod CrashLoopBackOff](../workloads/pod-crashloopbackoff.md)
* [Deployment Rollout Issues](../workloads/deployment-rollout-issues.md)