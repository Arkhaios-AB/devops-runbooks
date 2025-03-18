# Troubleshooting Kubelet Issues

## Symptoms

* Node showing `NotReady` status
* Container creation failures
* Pods stuck in `ContainerCreating` state
* Node stops reporting to the API server
* Kubelet service showing as failed or inactive
* High resource utilization by kubelet process
* Logs showing errors in `/var/log/kubelet.log` or via `journalctl`
* Pods on the node being evicted
* Network issues with pods on the node

## Possible Causes

1. **Kubelet Service Issues**: Kubelet service not running or crashing
2. **Configuration Errors**: Invalid kubelet configuration
3. **Certificate Problems**: Expired or invalid certificates
4. **API Server Connectivity**: Kubelet unable to reach the API server
5. **Container Runtime Issues**: Problems with Docker, containerd, or CRI-O
6. **Resource Exhaustion**: Node running out of memory, CPU, or disk space
7. **Network Plugin Issues**: CNI plugin problems
8. **Kernel Issues**: Kernel parameters not properly set
9. **File System Issues**: Issues with `/var/lib/kubelet` or `/var/log`
10. **SELinux/AppArmor Conflicts**: Security modules blocking kubelet operations

## Diagnosis Steps

### 1. Check Kubelet Service Status

```bash
# Check kubelet service status
sudo systemctl status kubelet

# Check if kubelet is running
ps aux | grep kubelet
```

### 2. Check Kubelet Logs

```bash
# Check kubelet logs via journalctl
sudo journalctl -u kubelet -n 100 --no-pager

# Check for specific error patterns
sudo journalctl -u kubelet | grep -i error
sudo journalctl -u kubelet | grep -i "unable to"
```

### 3. Check Kubelet Configuration

```bash
# Check kubelet configuration
sudo cat /var/lib/kubelet/config.yaml

# Check kubelet startup arguments
ps aux | grep kubelet | grep "\--"
sudo cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```

### 4. Check API Server Connectivity

```bash
# Test connectivity to API server
curl -k https://<apiserver-ip>:6443/healthz

# Check kubelet connectivity with node status
kubectl get node <node-name> -o yaml
```

### 5. Check Certificate Status

```bash
# Check certificate expiration
sudo openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -dates

# Check certificate is valid for the right hostname
sudo openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -text | grep Subject
```

### 6. Check Container Runtime

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

### 7. Check Node Resources

```bash
# Check memory usage
free -m

# Check disk space
df -h

# Check for zombie processes
ps aux | grep Z
```

## Resolution Steps

### 1. Restart Kubelet Service

If kubelet service is not running or has stalled:

```bash
# Restart kubelet service
sudo systemctl restart kubelet

# Enable kubelet to start on boot
sudo systemctl enable kubelet
```

### 2. Fix Kubelet Configuration

If configuration is incorrect:

```bash
# Edit kubelet configuration
sudo vi /var/lib/kubelet/config.yaml

# Apply changes and restart
sudo systemctl restart kubelet
```

Common configuration issues to check:
- `staticPodPath`: Should point to the correct directory
- `cgroupDriver`: Should match container runtime (usually `systemd` or `cgroupfs`)
- `clusterDNS`: Should point to correct cluster DNS service
- `resolvConf`: Should point to a valid resolv.conf file

### 3. Renew Certificates

If certificates have expired:

```bash
# For kubeadm clusters
sudo kubeadm certs renew kubelet-client

# For non-kubeadm clusters, regenerate kubelet certificates
# This depends on your certificate management system
```

### 4. Fix Container Runtime Issues

If the container runtime is failing:

```bash
# Restart container runtime
# For Docker
sudo systemctl restart docker

# For containerd
sudo systemctl restart containerd

# For CRI-O
sudo systemctl restart crio
```

### 5. Address Resource Exhaustion

If the node is out of resources:

```bash
# Clear disk space
sudo rm -rf /var/log/*.gz
sudo journalctl --vacuum-size=500M

# Clear unused images
sudo crictl rmi --prune
```

### 6. Reset Kubelet State

As a last resort, if kubelet state is corrupt:

```bash
# Stop kubelet
sudo systemctl stop kubelet

# Remove kubelet state
sudo rm -rf /var/lib/kubelet/pods/*

# Start kubelet
sudo systemctl start kubelet
```

### 7. Fix Network Plugin Issues

If CNI plugins are misconfigured:

```bash
# Check CNI configuration
ls -la /etc/cni/net.d/

# Reinstall CNI plugins if needed
# Example for Calico
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

### 8. Fix SELinux/AppArmor Issues

If security modules are blocking kubelet:

```bash
# Temporarily set SELinux to permissive
sudo setenforce 0

# For a permanent fix, configure SELinux policies properly
sudo vi /etc/selinux/config
```

## Prevention

1. **Regular Monitoring**: Set up monitoring for kubelet service status
2. **Certificate Management**: Monitor certificate expiration and renew before they expire
3. **Resource Monitoring**: Monitor node resources and set alerts for resource exhaustion
4. **Configuration Management**: Use configuration management tools to ensure consistent kubelet configuration
5. **Regular Updates**: Keep kubelet and Kubernetes components updated
6. **Backup Configuration**: Keep backups of working kubelet configuration
7. **Log Rotation**: Configure proper log rotation for kubelet logs
8. **Security Module Configuration**: Properly configure SELinux/AppArmor for Kubernetes
9. **Regular Testing**: Regularly test node failure and recovery scenarios
10. **Documentation**: Document node-specific configuration and troubleshooting steps

## Related Runbooks

* [Node Not Ready](./node-not-ready.md)
* [Certificate Issues](./certificate-issues.md)
* [Node Memory Pressure](./node-memory-pressure.md)
* [Node Disk Pressure](./node-disk-pressure.md)
* [Pod Stuck in ContainerCreating](../workloads/pod-container-creating.md)
* [CNI Plugin Issues](../networking/cni-plugin-issues.md)