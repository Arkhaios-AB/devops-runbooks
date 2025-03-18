# Troubleshooting API Server Unavailability

## Symptoms

* `kubectl` commands failing with connection errors
* Error messages such as "The connection to the server was refused" or "Unable to connect to the server"
* Dashboard or other Kubernetes UIs not loading
* Helm or other tools timing out when trying to interact with the cluster
* New pods not being scheduled
* Control plane operators logging connection errors
* Liveness probes failing for API-dependent applications

## Possible Causes

1. **API Server Crash**: kube-apiserver process has crashed or is not running
2. **Resource Exhaustion**: Control plane node out of resources
3. **etcd Issues**: Problems with the etcd datastore
4. **Network Problems**: Network connectivity issues between components
5. **Certificate Issues**: Expired or invalid TLS certificates
6. **Load Balancer Problems**: Issues with load balancer for HA control plane
7. **Authentication Issues**: Problems with authentication providers
8. **Component Incompatibility**: Version mismatch between components
9. **Improper Configuration**: Misconfiguration of API server
10. **Security Group/Firewall Issues**: Blocked access to API server port

## Diagnosis Steps

### 1. Check API Server Status

```bash
# Try a basic API server connection
kubectl get nodes

# Check API server health directly
curl -k https://<control-plane-ip>:6443/healthz

# For managed Kubernetes, check cloud provider status page
```

### 2. Check API Server Pods (for self-managed Kubernetes)

```bash
# SSH into a control plane node
ssh user@<control-plane-ip>

# Check kube-apiserver status
sudo crictl ps | grep kube-apiserver

# Check logs for API server
sudo crictl logs <container-id>
# or
sudo journalctl -u kubelet | grep apiserver
```

### 3. Check API Server Process

```bash
# Check if API server process is running
ps aux | grep kube-apiserver

# Check systemd service status (for non-static pod deployments)
sudo systemctl status kube-apiserver
```

### 4. Check etcd Health

```bash
# Check etcd health
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# Check etcd member list
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list
```

### 5. Check Load Balancer Status (for HA setups)

```bash
# Check if load balancer endpoints are responding
curl -k https://<load-balancer-ip>:6443/healthz

# For cloud-based load balancers, check cloud console status
```

### 6. Check Certificate Validity

```bash
# Check API server certificate expiration
sudo openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates
```

### 7. Check Node Resources

```bash
# Check control plane node resources
free -m
df -h
top
```

## Resolution Steps

### 1. Restart API Server

For kubeadm clusters:
```bash
# Move the static pod manifest away and back (forces a restart)
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sleep 5
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
```

For non-kubeadm clusters:
```bash
# Restart the API server service
sudo systemctl restart kube-apiserver
```

### 2. Fix Resource Exhaustion

If control plane nodes are out of resources:
```bash
# Clear disk space
sudo rm -rf /var/log/*.gz
sudo journalctl --vacuum-size=500M

# If memory is the issue, identify and restart memory-hogging processes
sudo systemctl restart kubelet
```

### 3. Fix etcd Issues

If etcd is unhealthy:
```bash
# Restart etcd
sudo systemctl restart etcd
# or for kubeadm
sudo mv /etc/kubernetes/manifests/etcd.yaml /tmp/
sleep 5
sudo mv /tmp/etcd.yaml /etc/kubernetes/manifests/

# If etcd data is corrupted, restore from backup
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot restore /path/to/backup/snapshot
```

### 4. Fix Certificate Issues

If certificates have expired:
```bash
# For kubeadm clusters
sudo kubeadm certs renew apiserver

# For non-kubeadm clusters, regenerate API server certificates
# This depends on your certificate management system
```

### 5. Fix Network/Firewall Issues

Check firewall rules and security groups:
```bash
# Check local firewall
sudo iptables -L

# Make sure port 6443 is open
sudo iptables -A INPUT -p tcp --dport 6443 -j ACCEPT
sudo iptables -A OUTPUT -p tcp --sport 6443 -j ACCEPT
```

For cloud environments, check security groups and network ACLs in the cloud console.

### 6. Fix Load Balancer Issues

For HA setups with load balancers:
- Check load balancer health checks
- Ensure backend pools include all control plane nodes
- Verify load balancer configuration for TLS settings

### 7. Fix API Server Configuration

If API server configuration is incorrect:
```bash
# Edit API server manifest
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

Common configuration issues to check:
- `--etcd-servers` pointing to correct etcd endpoints
- `--service-cluster-ip-range` matching the service CIDR
- Proper authentication and authorization flags

## Prevention

1. **High Availability**: Deploy multiple API server instances
2. **Resource Planning**: Ensure control plane nodes have adequate resources
3. **Monitoring**: Set up monitoring for API server health and performance
4. **Certificate Management**: Monitor certificate expiration and renew before they expire
5. **Regular Backups**: Implement regular etcd backups
6. **Load Testing**: Periodically test API server under load
7. **Version Compatibility**: Ensure components are compatible when upgrading
8. **Documentation**: Document API server configuration and troubleshooting steps
9. **Audit Logging**: Enable audit logging to diagnose authentication issues
10. **Redundancy**: Set up geographical redundancy for critical clusters

## Related Runbooks

* [etcd Issues](./etcd-issues.md)
* [Certificate Issues](./certificate-issues.md)
* [Node Not Ready](./node-not-ready.md)
* [Kubelet Issues](./kubelet-issues.md)
* [RBAC Permission Problems](../security/rbac-permission-problems.md)