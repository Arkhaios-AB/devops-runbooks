# Troubleshooting Pod Stuck in Terminating

## Symptoms

* Pods remain in `Terminating` state for an extended period
* `kubectl delete pod` command hangs
* Node draining operations stall due to pods not terminating
* Deployments or StatefulSets unable to complete rollouts
* Cluster upgrades stall due to pods not terminating

## Possible Causes

1. **Container Process Issues**: Container process not responding to SIGTERM
2. **Volume Unmounting Problems**: Volumes cannot be unmounted
3. **Finalizer Blocking**: Pod has finalizers preventing deletion
4. **Node Communication Issues**: Kubelet cannot communicate with API server
5. **Container Runtime Problems**: Docker, containerd, or CRI-O issues
6. **Node Kernel Issues**: Kernel-level process handling problems
7. **Stuck API Operations**: Kubernetes API operations stuck
8. **PodDisruptionBudget Conflicts**: PDB preventing pod termination
9. **Resource Busy**: Resources in use by other processes
10. **Node Failure**: Node is down or unreachable

## Diagnosis Steps

### 1. Check Pod Status and Finalizers

```bash
# Get pod details
kubectl describe pod <pod-name> -n <namespace>

# Check finalizers on the pod
kubectl get pod <pod-name> -n <namespace> -o json | jq '.metadata.finalizers'
```

### 2. Check Node Status

```bash
# Get node where pod is scheduled
NODE=$(kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.nodeName}')

# Check node status
kubectl describe node $NODE
```

### 3. Check Kubelet Logs

SSH to the node and check kubelet logs:

```bash
sudo journalctl -u kubelet | grep <pod-name>
```

### 4. Check Container Runtime Status

SSH to the node and check container runtime:

```bash
# For Docker
sudo docker ps | grep <container-name>
sudo docker inspect <container-id>

# For containerd
sudo crictl ps | grep <pod-name>
sudo crictl inspect <container-id>

# For CRI-O
sudo crictl ps | grep <pod-name>
sudo crictl inspect <container-id>
```

### 5. Check Volume Status

SSH to the node and check mounted volumes:

```bash
# List mounted volumes
mount | grep <volume-path>

# Check if volumes are in use
sudo lsof <volume-path>
```

### 6. Check PodDisruptionBudgets

```bash
# List PDBs in the namespace
kubectl get poddisruptionbudgets -n <namespace>

# Check if PDB is blocking termination
kubectl describe poddisruptionbudget <pdb-name> -n <namespace>
```

## Resolution Steps

### 1. Remove Finalizers

If finalizers are preventing deletion:

```bash
# Get pod details in JSON format
kubectl get pod <pod-name> -n <namespace> -o json > pod.json

# Edit the JSON to remove finalizers
# Change: "finalizers": ["example.com/finalizer"]
# To: "finalizers": []
vi pod.json

# Update the pod without validation
kubectl replace --raw "/api/v1/namespaces/<namespace>/pods/<pod-name>/status" -f pod.json
```

### 2. Force Delete the Pod

If the pod won't terminate normally:

```bash
# Force delete the pod
kubectl delete pod <pod-name> -n <namespace> --force --grace-period=0
```

### 3. Manually Clean Up Container Runtime

If the container runtime still shows the container:

SSH to the node and:

```bash
# For Docker
sudo docker rm -f <container-id>

# For containerd
sudo crictl rm -f <container-id>

# For CRI-O
sudo crictl rm -f <container-id>
```

### 4. Restart Container Runtime

If the container runtime is unresponsive:

```bash
# For Docker
sudo systemctl restart docker

# For containerd
sudo systemctl restart containerd

# For CRI-O
sudo systemctl restart crio
```

### 5. Fix Volume Unmount Issues

SSH to the node and:

```bash
# Check processes using the volume
sudo lsof <volume-path>

# Terminate processes using the volume
sudo kill -9 <pid>

# Force unmount if necessary
sudo umount -f <volume-path>

# If device is busy, use lazy unmount
sudo umount -l <volume-path>
```

### 6. Restart Kubelet

If kubelet is stuck:

```bash
sudo systemctl restart kubelet
```

### 7. Cordon and Drain the Node

If multiple pods are stuck on the same node:

```bash
# Cordon the node
kubectl cordon $NODE

# Drain the node (skip stuck pods)
kubectl drain $NODE --ignore-daemonsets --delete-local-data --force
```

### 8. Restart API Server

If API server is stuck (extreme measure):

For kubeadm clusters:
```bash
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sleep 5
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
```

### 9. Reboot the Node

As a last resort:

```bash
# Reboot the node
sudo reboot
```

## Prevention

1. **Graceful Termination**: Ensure applications handle SIGTERM properly
2. **Termination Grace Period**: Set appropriate termination grace periods
3. **Resource Cleanup**: Implement proper resource cleanup in applications
4. **Finalizer Management**: Carefully manage custom finalizers
5. **PDB Configuration**: Configure PDBs to allow pod termination when needed
6. **Container Health Checks**: Implement proper liveness and readiness probes
7. **Pod Disruption Planning**: Plan for pod disruptions in application design
8. **Node Maintenance**: Regularly maintain nodes and check for issues
9. **Monitoring**: Monitor pod termination events and duration
10. **Documentation**: Document procedures for handling stuck pods

## Related Runbooks

* [Node Not Ready](../cluster/node-not-ready.md)
* [Kubelet Issues](../cluster/kubelet-issues.md)
* [Volume Mount Problems](../storage/volume-mount-problems.md)
* [API Server Unavailable](../cluster/api-server-unavailable.md)
* [Deployment Rollout Issues](./deployment-rollout-issues.md)