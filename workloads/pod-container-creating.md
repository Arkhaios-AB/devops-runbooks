# Troubleshooting Pod Stuck in ContainerCreating

## Symptoms

* Pods remain in `ContainerCreating` state for an extended period
* `kubectl get pods` shows pod status as `ContainerCreating`
* Deployments or StatefulSets not reaching ready state
* Events showing container creation failures
* Container runtime errors in kubelet logs

## Possible Causes

1. **Volume Mount Issues**: Problems with PVCs, ConfigMaps, or Secrets
2. **Container Image Issues**: Image pull failures or invalid images
3. **Container Runtime Problems**: Docker, containerd, or CRI-O issues
4. **Node Resource Constraints**: Node resource exhaustion
5. **Network Plugin Issues**: CNI setup problems
6. **Security Constraints**: SELinux, AppArmor, or seccomp blocking container creation
7. **Node File System Issues**: Read-only filesystem or corrupt files
8. **Init Container Failures**: Init containers failing to complete
9. **Kubelet Problems**: Kubelet unable to communicate with container runtime
10. **Storage Driver Issues**: Storage driver conflicts or configuration problems

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

### 2. Check Volume Issues

```bash
# Check if PVCs are bound
kubectl get pvc -n <namespace>

# Check PV status
kubectl get pv | grep <pvc-name>

# Verify ConfigMaps exist
kubectl get configmap -n <namespace> | grep <configmap-name>

# Verify Secrets exist
kubectl get secret -n <namespace> | grep <secret-name>
```

### 3. Check Container Image Issues

```bash
# Check if image exists in container registry
# (Method depends on your registry)

# For Docker Hub
docker pull <image-name>:<tag>

# For private registries, check image pull secrets
kubectl get pod <pod-name> -n <namespace> -o yaml | grep imagePullSecrets -A 3
```

### 4. Check Node and Kubelet Status

```bash
# Get node where pod is scheduled
NODE=$(kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.nodeName}')

# Check node status
kubectl describe node $NODE

# SSH to the node and check kubelet logs
ssh user@$NODE
sudo journalctl -u kubelet | grep <pod-name>
```

### 5. Check Container Runtime

SSH to the node and check container runtime status:

```bash
# For Docker
sudo systemctl status docker
sudo docker info

# For containerd
sudo systemctl status containerd
sudo crictl info

# For CRI-O
sudo systemctl status crio
sudo crictl info
```

### 6. Check Network Plugin

```bash
# Check CNI configuration
ls -la /etc/cni/net.d/

# Check CNI plugin pods
kubectl get pods -n kube-system | grep -E 'calico|flannel|weave|cilium'
```

## Resolution Steps

### 1. Fix Volume Mount Issues

If PVC is not bound:

```bash
# Create missing PV if using static provisioning
kubectl apply -f pv-definition.yaml

# Check storage class for dynamic provisioning
kubectl get storageclass

# Create default storage class if needed
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs  # Adjust for your environment
parameters:
  type: gp2
reclaimPolicy: Delete
allowVolumeExpansion: true
EOF
```

If ConfigMap or Secret is missing:

```bash
# Create missing ConfigMap
kubectl create configmap <configmap-name> --from-file=<path> -n <namespace>

# Create missing Secret
kubectl create secret generic <secret-name> --from-literal=key=value -n <namespace>
```

### 2. Fix Container Image Issues

If image pull is failing:

```bash
# Create or update image pull secret
kubectl create secret docker-registry regcred \
  --docker-server=<your-registry> \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=<email> \
  -n <namespace> \
  --dry-run=client -o yaml | kubectl apply -f -

# Add image pull secret to the pod or service account
kubectl patch serviceaccount <service-account> -n <namespace> \
  -p '{"imagePullSecrets": [{"name": "regcred"}]}'

# Verify image path and tag are correct
kubectl edit deployment <deployment-name> -n <namespace>
# Edit the image path if needed
```

### 3. Fix Container Runtime Issues

SSH to the node and:

```bash
# Restart container runtime
# For Docker
sudo systemctl restart docker

# For containerd
sudo systemctl restart containerd

# For CRI-O
sudo systemctl restart crio

# Clean up stale containers
sudo crictl rmp $(sudo crictl pods --name <pod-name> -q)
```

### 4. Fix Resource Constraints

If node is out of resources:

```bash
# Free up resources by removing unused containers and images
sudo crictl rmi --prune

# Evict non-critical pods from node
kubectl drain <node-name> --ignore-daemonsets --delete-local-data=true \
  --force=true --grace-period=120 --timeout=300s \
  --selector 'app!=critical-app'
```

### 5. Fix Network Plugin Issues

```bash
# Restart CNI pods
kubectl rollout restart daemonset <cni-daemonset> -n kube-system

# Reinstall CNI plugins if necessary
# Example for Calico
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

### 6. Fix Security Constraints

SSH to the node and:

```bash
# Temporarily disable SELinux to test if it's the issue
sudo setenforce 0

# If SELinux is the issue, configure it properly for containers
sudo setsebool -P container_manage_cgroup 1
```

### 7. Fix Kubelet Issues

```bash
# Restart kubelet
sudo systemctl restart kubelet

# If necessary, reset kubelet state
sudo rm -rf /var/lib/kubelet/pods/<pod-uid>
sudo systemctl restart kubelet
```

### 8. Delete and Recreate Pod

If all else fails:

```bash
# Force delete the stuck pod
kubectl delete pod <pod-name> -n <namespace> --force --grace-period=0

# If created by controller, controller will recreate it
# Otherwise, recreate manually
kubectl apply -f pod-definition.yaml
```

## Prevention

1. **Pre-pull Images**: Pre-pull commonly used images on nodes
2. **Resource Planning**: Set appropriate resource requests and limits
3. **Health Checks**: Implement proper liveness and readiness probes
4. **Storage Testing**: Test storage classes and volume provisioning
5. **Image Strategy**: Use specific image tags or digests, not "latest"
6. **Security Context**: Configure appropriate security contexts for pods
7. **Node Maintenance**: Regularly maintain nodes and container runtime
8. **CNI Monitoring**: Monitor CNI plugins health
9. **Documentation**: Document common container creation issues and fixes
10. **Storage Classes**: Set up default storage classes for dynamic provisioning

## Related Runbooks

* [Pod Stuck in Pending](./pod-pending.md)
* [Pod ImagePullBackOff](./pod-imagepullbackoff.md)
* [PVC Stuck in Pending](../storage/pvc-pending.md)
* [Volume Mount Problems](../storage/volume-mount-problems.md)
* [Node Not Ready](../cluster/node-not-ready.md)
* [Kubelet Issues](../cluster/kubelet-issues.md)
* [CNI Plugin Issues](../networking/cni-plugin-issues.md)