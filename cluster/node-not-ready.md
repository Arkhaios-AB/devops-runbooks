# Troubleshooting Node Not Ready

## Symptoms

* Node status shows `NotReady` when running `kubectl get nodes`
* Pods scheduled on the node may be stuck in `Pending` state or evicted
* New pods are not scheduled on the affected node
* `kubectl describe node <node-name>` shows error conditions

## Possible Causes

1. **Kubelet Issues**: Kubelet service not running or encountering errors
2. **Network Problems**: Node cannot communicate with the control plane/API server
3. **Resource Exhaustion**: Node is experiencing memory, CPU, or disk pressure
4. **Certificate Issues**: Expired or invalid kubelet certificates
5. **Container Runtime Issues**: Docker, containerd, or CRI-O runtime problems
6. **Cloud Provider Issues**: Issues with underlying cloud infrastructure
7. **Node Cordoned**: Node was manually cordoned for maintenance
8. **CNI Problems**: Container Network Interface plugin issues

## Diagnosis Steps

### 1. Check Node Status and Conditions

```bash
# Get node status
kubectl get nodes

# Get detailed node information
kubectl describe node <node-name>

# Check node conditions
kubectl get node <node-name> -o jsonpath='{.status.conditions[*].type}{"\t"}{.status.conditions[*].status}{"\n"}'
```

Look for specific conditions like `Ready`, `MemoryPressure`, `DiskPressure`, `NetworkUnavailable`, etc.

### 2. Check Kubelet Status

SSH into the affected node and check the kubelet service:

```bash
# Check kubelet service status
sudo systemctl status kubelet

# Check kubelet logs
sudo journalctl -u kubelet -n 100 --no-pager
```

### 3. Check System Resources

```bash
# Check disk space
df -h

# Check memory usage
free -m

# Check CPU usage
top

# Check system logs
dmesg | tail -n 100
```

### 4. Check Container Runtime

```bash
# For Docker
sudo systemctl status docker
sudo journalctl -u docker -n 100 --no-pager

# For containerd
sudo systemctl status containerd
sudo journalctl -u containerd -n 100 --no-pager

# For CRI-O
sudo systemctl status crio
sudo journalctl -u crio -n 100 --no-pager
```

### 5. Check Network Connectivity

```bash
# Test connectivity to API server
curl -k https://<api-server-address>:6443/healthz

# Check CNI plugins
ls -la /etc/cni/net.d/
cat /etc/cni/net.d/<network-config-file>
```

## Resolution Steps

### 1. Fix Kubelet Issues

If kubelet service is failing:

```bash
# Restart kubelet service
sudo systemctl restart kubelet

# If configuration is corrupted, check and repair kubelet config
sudo vi /etc/kubernetes/kubelet.conf
```

### 2. Fix Resource Exhaustion

If the node has disk pressure:

```bash
# Clean up container images
sudo crictl rmi --prune

# Clean up logs
sudo journalctl --vacuum-size=500M
```

If the node has memory pressure:

```bash
# Identify and restart or terminate high memory processes
sudo systemctl restart <high-memory-service>
```

### 3. Fix Certificate Issues

If certificates are expired:

```bash
# For kubeadm clusters, renew certificates
sudo kubeadm certs renew all

# Restart kubelet after renewing certificates
sudo systemctl restart kubelet
```

### 4. Fix Container Runtime Issues

If container runtime is failing:

```bash
# Restart container runtime
# For Docker
sudo systemctl restart docker

# For containerd
sudo systemctl restart containerd

# For CRI-O
sudo systemctl restart crio
```

### 5. Fix CNI Issues

If CNI plugins are misconfigured:

```bash
# Reinstall CNI plugins
# Example for Calico
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Example for Flannel
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

### 6. Uncordon Node

If the node was cordoned:

```bash
kubectl uncordon <node-name>
```

## Prevention

1. **Implement Node Monitoring**: Set up monitoring for node resources (CPU, memory, disk)
2. **Resource Quotas**: Implement pod resource quotas to prevent overloading nodes
3. **Certificate Monitoring**: Set up alerts for expiring certificates
4. **Regular Maintenance**: Schedule regular node maintenance and updates
5. **Use Node Autoscaling**: Implement node autoscaling for dynamic workloads
6. **Keep Container Runtime Updated**: Regularly update container runtime components
7. **CNI Redundancy**: Ensure CNI configuration is robust and redundant

## Related Runbooks

* [Node Memory Pressure](./node-memory-pressure.md)
* [Node Disk Pressure](./node-disk-pressure.md)
* [Node CPU Pressure](./node-cpu-pressure.md)
* [Kubelet Issues](./kubelet-issues.md)
* [Certificate Issues](./certificate-issues.md)
* [CNI Plugin Issues](../networking/cni-plugin-issues.md)
* [Pod Not Ready](../workloads/pod-not-ready.md)