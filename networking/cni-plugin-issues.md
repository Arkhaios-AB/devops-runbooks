# Troubleshooting CNI Plugin Issues

## Symptoms

* Pods stuck in `ContainerCreating` with network-related errors
* Intermittent network connectivity between pods
* DNS resolution failures
* Pod IP assignment failures
* Network interface missing in pods
* MTU-related issues or packet fragmentation
* Nodes reporting "Node Not Ready" due to network issues
* NetworkPolicy not being enforced
* Performance degradation in pod-to-pod communication
* Multiple default routes or routing conflicts

## Possible Causes

1. **CNI Plugin Installation Issues**: Incorrect or incomplete CNI installation
2. **CNI Configuration Errors**: Misconfigured CNI plugin settings
3. **Pod CIDR Conflicts**: Overlapping IP ranges with nodes, services, or external networks
4. **CNI Binary Issues**: Missing, corrupt, or incompatible CNI binaries
5. **Network Interface Problems**: Host network interface issues affecting CNI
6. **Kubelet Configuration**: Kubelet not configured correctly for CNI
7. **MTU Mismatches**: Inconsistent MTU settings across the network path
8. **Resource Limitations**: CNI plugin pods hitting resource limits
9. **Version Incompatibility**: CNI plugin version incompatible with Kubernetes version
10. **CNI Plugin Crashes**: CNI daemon processes crashing or restarting

## Diagnosis Steps

### 1. Check CNI Pod Status

```bash
# Check CNI plugin pods (specific command depends on CNI used)
kubectl get pods -n kube-system -l k8s-app=calico-node  # For Calico
kubectl get pods -n kube-system -l k8s-app=cilium  # For Cilium
kubectl get pods -n kube-system -l name=weave-net  # For Weave
kubectl get pods -n kube-system -l app=flannel  # For Flannel

# Check CNI pod logs
kubectl logs -n kube-system <cni-pod-name>
```

### 2. Check CNI Installation on Nodes

SSH to a problematic node and check:

```bash
# Check CNI binaries
ls -la /opt/cni/bin/

# Check CNI configuration
ls -la /etc/cni/net.d/
cat /etc/cni/net.d/*

# Check kubelet configuration for CNI settings
cat /var/lib/kubelet/config.yaml | grep -i cni
```

### 3. Check Pod Network Issues

```bash
# Get pod with networking issues
kubectl describe pod <pod-name> -n <namespace>

# Check if pod has IP assigned
kubectl get pod <pod-name> -n <namespace> -o wide

# Try to create a test pod
kubectl run test-pod --image=nicolaka/netshoot
kubectl exec -it test-pod -- ip a
kubectl exec -it test-pod -- ip route
```

### 4. Check MTU Settings

```bash
# Check MTU on host
ip link show | grep mtu

# Check MTU in CNI configuration
cat /etc/cni/net.d/* | grep -i mtu

# Check MTU inside pod
kubectl exec -it test-pod -- ip link show | grep mtu
```

### 5. Check Network Connectivity

```bash
# Test connectivity between pods
kubectl exec -it test-pod -- ping <other-pod-ip>

# Test DNS resolution
kubectl exec -it test-pod -- nslookup kubernetes.default.svc.cluster.local

# Trace network path
kubectl exec -it test-pod -- traceroute <other-pod-ip>
```

### 6. Check Kubelet Logs

SSH to the node and check kubelet logs:

```bash
sudo journalctl -u kubelet | grep -i cni
sudo journalctl -u kubelet | grep -i network
```

## Resolution Steps

### 1. Fix CNI Installation Issues

If CNI plugin is not properly installed:

```bash
# Re-install CNI plugin (example for Calico)
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# For Cilium
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.9/install/kubernetes/quick-install.yaml

# For Weave
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

# For Flannel
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

### 2. Fix CNI Configuration

If CNI configuration is incorrect:

```bash
# Backup current configuration
sudo cp -r /etc/cni/net.d/ /etc/cni/net.d.bak

# Remove invalid configuration
sudo rm -f /etc/cni/net.d/*.conflist

# Apply correct configuration
kubectl apply -f correct-cni-config.yaml

# Restart kubelet
sudo systemctl restart kubelet
```

### 3. Fix Pod CIDR Conflicts

If pod CIDR ranges are conflicting:

```bash
# Check current pod CIDR configuration
kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}'

# Update CNI configuration with non-overlapping CIDR
# This varies by CNI plugin, example for Calico:
kubectl edit configmap calico-config -n kube-system
# Change CALICO_IPV4POOL_CIDR to non-overlapping range

# Restart CNI pods
kubectl rollout restart daemonset calico-node -n kube-system
```

### 4. Fix MTU Issues

If MTU settings are causing problems:

```bash
# Edit CNI config to set correct MTU
kubectl edit configmap calico-config -n kube-system  # For Calico
# Set appropriate MTU value (usually 1500 or 9000 if jumbo frames are enabled)
# Or 1450 for overlay networks

# Restart CNI pods
kubectl rollout restart daemonset calico-node -n kube-system
```

### 5. Fix CNI Binary Issues

If CNI binaries are missing or corrupt:

```bash
# SSH to the node
ssh user@node

# Check CNI binaries
ls -la /opt/cni/bin/

# Install CNI binaries if missing
sudo mkdir -p /opt/cni/bin
curl -L "https://github.com/containernetworking/plugins/releases/download/v0.9.1/cni-plugins-linux-amd64-v0.9.1.tgz" | sudo tar -C /opt/cni/bin -xz

# Set proper permissions
sudo chmod +x /opt/cni/bin/*
```

### 6. Fix Kubelet Configuration

If kubelet is not configured correctly for CNI:

```bash
# Edit kubelet configuration
sudo vi /var/lib/kubelet/config.yaml

# Ensure CNI settings are correct
# networkPlugin: cni
# cniDir: /etc/cni/net.d
# cniBinDir: /opt/cni/bin

# Restart kubelet
sudo systemctl restart kubelet
```

### 7. Fix Resource Limitations

If CNI pods are hitting resource limits:

```bash
# Increase resource limits for CNI pods
kubectl edit daemonset <cni-daemonset> -n kube-system
```

Example resource adjustment:
```yaml
resources:
  limits:
    cpu: "500m"
    memory: "512Mi"
  requests:
    cpu: "250m"
    memory: "256Mi"
```

### 8. Fix Version Incompatibility

If CNI plugin version is incompatible:

```bash
# Check CNI plugin version
kubectl describe pod <cni-pod-name> -n kube-system | grep Image

# Update CNI plugin to compatible version
kubectl apply -f https://docs.projectcalico.org/v3.17/manifests/calico.yaml  # Specific version example
```

### 9. Reset Networking on Problematic Node

As a last resort, reset networking on problematic node:

```bash
# SSH to the node
ssh user@node

# Stop kubelet
sudo systemctl stop kubelet

# Remove CNI configuration
sudo rm -rf /etc/cni/net.d/*

# Remove CNI interfaces
sudo ip link delete <cni-interface-name>  # e.g., cali*, flannel*

# Restart kubelet
sudo systemctl start kubelet
```

### 10. Node Reboot

If all else fails:

```bash
# Drain node
kubectl drain <node-name> --ignore-daemonsets --delete-local-data

# Reboot node
sudo reboot

# Uncordon node after reboot
kubectl uncordon <node-name>
```

## Prevention

1. **Standard CNI Installation**: Use standardized CNI installation procedures
2. **CNI Version Compatibility**: Ensure CNI version is compatible with Kubernetes version
3. **CIDR Planning**: Plan pod and service CIDRs to avoid overlaps
4. **MTU Configuration**: Standardize MTU settings across network
5. **Monitoring**: Monitor CNI pod health and network connectivity
6. **Resource Allocation**: Allocate sufficient resources for CNI components
7. **Regular Updates**: Keep CNI plugins updated with security patches
8. **Documentation**: Document network architecture and CNI configuration
9. **Testing**: Test CNI changes in non-production environment first
10. **Backup Configuration**: Maintain backups of CNI configuration

## Related Runbooks

* [Pod Stuck in ContainerCreating](../workloads/pod-container-creating.md)
* [DNS Resolution Problems](./dns-resolution-problems.md)
* [Service Not Accessible](./service-not-accessible.md)
* [Network Policy Issues](./network-policy-issues.md)
* [Pod-to-Pod Communication Issues](./pod-to-pod-communication.md)
* [Node Not Ready](../cluster/node-not-ready.md)
* [Kubelet Issues](../cluster/kubelet-issues.md)