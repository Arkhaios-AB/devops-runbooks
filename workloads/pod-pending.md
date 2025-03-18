# Troubleshooting Pod Stuck in Pending

## Symptoms

* Pods remain in `Pending` state for an extended period
* `kubectl get pods` shows pod status as `Pending`
* Deployments or StatefulSets not reaching desired replica count
* Events showing scheduling failures
* Jobs or CronJobs not starting

## Possible Causes

1. **Insufficient Resources**: Not enough CPU, memory, or GPU resources on nodes
2. **Volume Mounting Issues**: PVC not bound or unavailable
3. **Node Selector Constraints**: Pod requires nodes with specific labels that don't exist
4. **Taints and Tolerations**: Node taints preventing pod scheduling
5. **Affinity/Anti-Affinity Rules**: Pod affinity/anti-affinity requirements can't be satisfied
6. **Pod Priority**: Lower priority pods preempted or can't be scheduled
7. **Resource Quotas**: Namespace resource quotas reached
8. **PodDisruptionBudget Conflicts**: PDBs preventing rescheduling
9. **ImagePullBackOff**: Container image cannot be pulled
10. **Network Plugin Issues**: CNI not configured correctly

## Diagnosis Steps

### 1. Check Pod Status and Events

```bash
# Get pod status
kubectl get pod <pod-name> -n <namespace>

# Check pod details
kubectl describe pod <pod-name> -n <namespace>

# Look at events for the pod
kubectl get events --sort-by=.metadata.creationTimestamp -n <namespace> | grep <pod-name>
```

### 2. Check Resource Availability

```bash
# Check node resource usage
kubectl get nodes --sort-by=.status.capacity.cpu
kubectl get nodes --sort-by=.status.capacity.memory

# Check detailed node resource allocation
kubectl describe node <node-name> | grep -A 10 "Allocated resources"
```

### 3. Check Storage/PVC Issues

```bash
# Check if PVCs are bound
kubectl get pvc -n <namespace>

# Check PV availability
kubectl get pv

# Describe the PVC for details
kubectl describe pvc <pvc-name> -n <namespace>
```

### 4. Check Node Selectors, Taints, and Affinity

```bash
# Extract node selector from pod
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.nodeSelector}'

# Check pod affinity settings
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.affinity}'

# List node taints
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints

# Check if nodes match required labels
kubectl get nodes --show-labels | grep <required-label>
```

### 5. Check Quotas and Limits

```bash
# Check namespace resource quotas
kubectl get resourcequota -n <namespace>

# Check detailed quota usage
kubectl describe resourcequota -n <namespace>

# Check limit ranges
kubectl get limitrange -n <namespace>
```

## Resolution Steps

### 1. Fix Resource Constraints

If the pod is pending due to insufficient resources:

```bash
# Option 1: Scale up the cluster to add more nodes
# (This depends on your environment: cloud provider, on-prem, etc.)

# Option 2: Modify pod resource requests to be smaller
kubectl edit deployment <deployment-name> -n <namespace>
```

Example resource adjustment:
```yaml
resources:
  requests:
    memory: "128Mi"  # Reduce from higher value
    cpu: "100m"      # Reduce from higher value
```

### 2. Fix PVC Issues

If PVCs are not bound:

```bash
# Create missing storage class if needed
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs  # Adjust based on your environment
parameters:
  type: gp2
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF

# If using static provisioning, create the missing PV
kubectl apply -f pv-definition.yaml
```

### 3. Fix Node Selector Issues

If node selectors can't be satisfied:

```bash
# Option 1: Add the required label to a node
kubectl label node <node-name> <key>=<value>

# Option 2: Change the pod's node selector
kubectl edit deployment <deployment-name> -n <namespace>
# Remove or modify the nodeSelector field
```

### 4. Fix Taint Issues

If node taints are preventing scheduling:

```bash
# Option 1: Remove the taint from a node
kubectl taint node <node-name> <taint-key>-

# Option 2: Add tolerations to the pod
kubectl edit deployment <deployment-name> -n <namespace>
```

Example toleration to add:
```yaml
tolerations:
- key: "key1"
  operator: "Exists"
  effect: "NoSchedule"
```

### 5. Fix Affinity/Anti-Affinity Issues

If affinity rules are too strict:

```bash
# Edit the deployment to adjust affinity settings
kubectl edit deployment <deployment-name> -n <namespace>
```

Consider relaxing affinity to preferred instead of required:
```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:  # instead of requiredDuringSchedulingIgnoredDuringExecution
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - <app-name>
        topologyKey: kubernetes.io/hostname
```

### 6. Fix Quota Issues

If resource quotas are preventing scheduling:

```bash
# Increase resource quota
kubectl edit resourcequota <quota-name> -n <namespace>

# Or create/apply a new quota
kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: <namespace>
spec:
  hard:
    pods: "20"
    requests.cpu: "4"
    requests.memory: 4Gi
    limits.cpu: "8"
    limits.memory: 8Gi
EOF
```

### 7. Check for ImagePullBackOff

If the pod is stuck in a combination of Pending and ImagePullBackOff:

```bash
# Fix image pull secrets
kubectl create secret docker-registry regcred \
  --docker-server=<your-registry> \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=<email> \
  -n <namespace>

# Add image pull secret to the deployment
kubectl patch serviceaccount default -n <namespace> \
  -p '{"imagePullSecrets": [{"name": "regcred"}]}'
```

### 8. Fix Network Plugin Issues

If CNI issues are causing pending pods:

```bash
# Check CNI pods health
kubectl get pods -n kube-system | grep cni

# Restart CNI pods if needed
kubectl rollout restart daemonset <cni-daemonset> -n kube-system
```

## Prevention

1. **Resource Planning**: Set appropriate resource requests
2. **Quota Management**: Configure appropriate namespace quotas
3. **Monitoring**: Monitor capacity and utilization regularly
4. **Node Groups**: Use node groups for specialized workloads
5. **Autoscaling**: Implement cluster autoscaling
6. **Testing**: Test new workload resource requirements before production
7. **Documentation**: Document resource requirements and constraints
8. **Taints and Labels**: Plan node taints and labels carefully
9. **Affinity Design**: Design pod affinity rules with flexibility
10. **Regular Audits**: Regularly audit scheduling constraints

## Related Runbooks

* [Node Not Ready](../cluster/node-not-ready.md)
* [PVC Stuck in Pending](../storage/pvc-pending.md)
* [Pod Stuck in ContainerCreating](./pod-container-creating.md)
* [Pod ImagePullBackOff](./pod-imagepullbackoff.md)
* [Resource Quota Issues](../resources/resource-quota-issues.md)