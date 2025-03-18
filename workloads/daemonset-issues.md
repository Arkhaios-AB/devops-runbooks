# Troubleshooting DaemonSet Issues

## Symptoms

* DaemonSet pods not running on all expected nodes
* DaemonSet pods stuck in Pending, CrashLoopBackOff, or other non-Ready states
* DaemonSet not respecting node selectors or taints
* DaemonSet updates not progressing
* Unexpected pod restarts across multiple nodes
* Nodes showing uneven distribution of DaemonSet pods
* DaemonSet pods showing "MatchNodeSelector" or "TaintToleration" scheduling errors

## Possible Causes

1. **Node Selector Mismatch**: DaemonSet node selector doesn't match node labels
2. **Taint Issues**: DaemonSet lacks necessary tolerations for node taints
3. **Resource Constraints**: Nodes don't have enough resources for DaemonSet pods
4. **Update Strategy Problems**: Incorrect update strategy configuration
5. **Pod Spec Issues**: Problems with pod template specification
6. **Admission Controller Rejection**: Pods rejected by admission controllers
7. **Container Image Issues**: Problems pulling or running container images
8. **Volume Mount Problems**: Issues with required volume mounts
9. **Controller Issues**: DaemonSet controller not functioning correctly
10. **Initialization Errors**: Init containers failing to complete

## Diagnosis Steps

### 1. Check DaemonSet Status

```bash
# Get DaemonSet status
kubectl get daemonset <daemonset-name> -n <namespace>

# Check detailed DaemonSet information
kubectl describe daemonset <daemonset-name> -n <namespace>

# Check events related to the DaemonSet
kubectl get events -n <namespace> | grep <daemonset-name>
```

### 2. Check Node and Pod Distribution

```bash
# List all nodes
kubectl get nodes

# Check which nodes are running DaemonSet pods
kubectl get pods -n <namespace> -l name=<daemonset-label> -o wide

# Compare nodes and DaemonSet pods to identify missing nodes
```

### 3. Check Node Selectors and Taints

```bash
# Check DaemonSet node selector
kubectl get daemonset <daemonset-name> -n <namespace> -o jsonpath='{.spec.template.spec.nodeSelector}'

# List node labels to compare with node selector
kubectl get nodes --show-labels

# Check node taints
kubectl describe nodes | grep -A5 Taints:
```

### 4. Check Pod Status

```bash
# List DaemonSet pods and their status
kubectl get pods -n <namespace> -l name=<daemonset-label>

# Check details of problematic pods
kubectl describe pod <pod-name> -n <namespace>

# Check pod logs
kubectl logs <pod-name> -n <namespace>
```

### 5. Check Resource Constraints

```bash
# Check node resource allocation
kubectl describe nodes | grep -A 10 "Allocated resources"

# Compare with DaemonSet pod resource requests
kubectl get daemonset <daemonset-name> -n <namespace> -o jsonpath='{.spec.template.spec.containers[0].resources}'
```

### 6. Check Update Strategy

```bash
# Check DaemonSet update strategy
kubectl get daemonset <daemonset-name> -n <namespace> -o jsonpath='{.spec.updateStrategy}'
```

## Resolution Steps

### 1. Fix Node Selector Issues

If the DaemonSet node selector doesn't match node labels:

```bash
# Check current node selector
kubectl get daemonset <daemonset-name> -n <namespace> -o jsonpath='{.spec.template.spec.nodeSelector}'

# Option 1: Update node labels to match selector
kubectl label node <node-name> <key>=<value>

# Option 2: Update DaemonSet node selector
kubectl patch daemonset <daemonset-name> -n <namespace> \
  --type=json \
  -p='[{"op": "replace", "path": "/spec/template/spec/nodeSelector", "value": {"<key>": "<value>"}}]'
```

### 2. Fix Taint Toleration Issues

If node taints are preventing DaemonSet pods from running:

```bash
# Add necessary tolerations to the DaemonSet
kubectl patch daemonset <daemonset-name> -n <namespace> --type=json \
  -p='[{"op": "add", "path": "/spec/template/spec/tolerations", "value": [{"key": "<taint-key>", "operator": "Exists", "effect": "<taint-effect>"}]}]'
```

Example for common taints:
```yaml
tolerations:
- key: "node-role.kubernetes.io/master"
  operator: "Exists"
  effect: "NoSchedule"
- key: "node.kubernetes.io/not-ready"
  operator: "Exists"
  effect: "NoExecute"
```

### 3. Fix Resource Constraints

If nodes don't have enough resources:

```bash
# Update DaemonSet to reduce resource requests
kubectl edit daemonset <daemonset-name> -n <namespace>
```

Example resource adjustment:
```yaml
resources:
  limits:
    memory: "256Mi"
    cpu: "200m"
  requests:
    memory: "128Mi"
    cpu: "100m"
```

### 4. Fix Update Strategy

If updates are not progressing correctly:

```bash
# Update the DaemonSet update strategy
kubectl patch daemonset <daemonset-name> -n <namespace> \
  --type=json \
  -p='[{"op": "replace", "path": "/spec/updateStrategy", "value": {"type": "RollingUpdate", "rollingUpdate": {"maxUnavailable": 1}}}]'
```

Available update strategies:
- `RollingUpdate`: Updates pods one at a time
- `OnDelete`: Updates pods only when they are manually deleted

### 5. Fix Pod Spec Issues

If there are issues with the pod template:

```bash
# Edit the DaemonSet pod template
kubectl edit daemonset <daemonset-name> -n <namespace>
```

Common issues to check:
- Incorrect container image name or tag
- Missing environment variables
- Incorrect command or arguments
- Faulty health check configurations
- Missing volume definitions

### 6. Fix Admission Controller Rejections

If admission controllers are rejecting pods:

```bash
# Check for webhook configurations
kubectl get validatingwebhookconfiguration,mutatingwebhookconfiguration

# Modify the webhook to exclude the namespace or add exemptions
kubectl edit validatingwebhookconfiguration <webhook-name>
```

### 7. Fix Container Image Issues

If there are issues with pulling images:

```bash
# Update image pull secrets if needed
kubectl edit daemonset <daemonset-name> -n <namespace>
```

Add image pull secrets to pod spec:
```yaml
imagePullSecrets:
- name: regcred
```

### 8. Restart DaemonSet Controller (in extreme cases)

If the DaemonSet controller appears to be malfunctioning:

```bash
# Restart kube-controller-manager in a kubeadm cluster
sudo mv /etc/kubernetes/manifests/kube-controller-manager.yaml /tmp/
sleep 5
sudo mv /tmp/kube-controller-manager.yaml /etc/kubernetes/manifests/
```

### 9. Delete and Recreate DaemonSet

As a last resort:

```bash
# Export the current DaemonSet definition
kubectl get daemonset <daemonset-name> -n <namespace> -o yaml > daemonset.yaml

# Edit the YAML file to remove status and other generated fields
# Then delete and recreate
kubectl delete daemonset <daemonset-name> -n <namespace>
kubectl apply -f daemonset.yaml
```

### 10. Fix Scheduling Issues Across Zones

For multi-zone clusters with zone-specific issues:

```bash
# Update DaemonSet to add affinity rules
kubectl edit daemonset <daemonset-name> -n <namespace>
```

Example affinity to exclude a problematic zone:
```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: topology.kubernetes.io/zone
          operator: NotIn
          values:
          - problematic-zone
```

## Prevention

1. **Proper Tolerations**: Include tolerations for common taints (node-role.kubernetes.io/master, node.kubernetes.io/not-ready)
2. **Resource Planning**: Set appropriate resource requests for DaemonSet pods
3. **Testing**: Test DaemonSet deployments on representative node types
4. **Monitoring**: Monitor DaemonSet status and distribution
5. **Update Strategy**: Choose the appropriate update strategy for your workload
6. **Node Labeling Strategy**: Implement a consistent node labeling strategy
7. **Pod Priority**: Set appropriate pod priority for critical DaemonSets
8. **Documentation**: Document DaemonSet configurations and expected behavior
9. **Health Checks**: Implement proper liveness and readiness probes
10. **Version Compatibility**: Ensure DaemonSet is compatible with all node versions

## Related Runbooks

* [Node Not Ready](../cluster/node-not-ready.md)
* [Pod Stuck in Pending](./pod-pending.md)
* [Pod CrashLoopBackOff](./pod-crashloopbackoff.md)
* [Pod ImagePullBackOff](./pod-imagepullbackoff.md)
* [Volume Mount Problems](../storage/volume-mount-problems.md)
* [CNI Plugin Issues](../networking/cni-plugin-issues.md)
* [Admission Controller Issues](../cluster/admission-controller-issues.md)