# Troubleshooting etcd Issues

## Symptoms

* API server operations becoming slow or timing out
* Kubernetes API server logs showing connection issues to etcd
* Increased latency for API requests
* Error messages in etcd logs
* Failed etcd health checks
* Cluster-wide degradation of performance
* Unable to create or update resources
* `kubectl` commands stalling or returning errors
* etcd member disconnections or quorum loss

## Possible Causes

1. **Disk I/O Issues**: Slow disk performance affecting etcd
2. **Network Problems**: Network latency or connectivity issues
3. **Resource Constraints**: CPU or memory pressure on etcd nodes
4. **Quorum Loss**: Not enough etcd members available to form quorum
5. **Database Size**: etcd database grown too large
6. **Certificate Issues**: Expired or invalid TLS certificates
7. **Version Incompatibility**: Version mismatch between etcd members
8. **Backup Failure**: Failed or corrupt etcd backups
9. **Split-Brain**: Network partition causing split-brain scenario
10. **Corrupted Data**: Database corruption from unexpected shutdowns

## Diagnosis Steps

### 1. Check etcd Health and Status

```bash
# For kubeadm clusters, use kubectl to check etcd pods
kubectl get pods -n kube-system -l component=etcd

# SSH into a control plane node and check etcd status
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list
  
# Check etcd health
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health
```

### 2. Check etcd Logs

```bash
# For kubeadm clusters
kubectl logs -n kube-system etcd-<hostname>

# For systemd-managed etcd
sudo journalctl -u etcd
```

### 3. Check etcd Metrics

```bash
# Get etcd metrics
curl -k -L https://127.0.0.1:2379/metrics
```

### 4. Check etcd Database Size

```bash
# Check DB size
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status -w table
```

### 5. Check Disk Performance

```bash
# Check disk I/O
sudo iostat -x 1 10

# Check disk latency
sudo dd if=/dev/zero of=/var/lib/etcd/test bs=512 count=1000 oflag=dsync
```

### 6. Check Network Connectivity

```bash
# Check network connectivity between etcd nodes
ping <other-etcd-node-ip>

# Check network latency
traceroute <other-etcd-node-ip>
```

## Resolution Steps

### 1. Fix Disk I/O Issues

If disk performance is poor:

```bash
# Move etcd data to faster storage
sudo systemctl stop etcd  # or stop the etcd pods
sudo rsync -avz /var/lib/etcd /path/to/faster/storage/
```

Update etcd configuration to use the new path:

```bash
# For kubeadm
sudo vi /etc/kubernetes/manifests/etcd.yaml
# Update volumes and volumeMounts to point to new location

# For non-kubeadm
sudo vi /etc/etcd/etcd.conf
# Update data-dir parameter
```

### 2. Fix Resource Constraints

Increase resources for etcd:

```bash
# For kubeadm
sudo vi /etc/kubernetes/manifests/etcd.yaml
```

Adjust resource limits:
```yaml
resources:
  requests:
    cpu: "200m"
    memory: "512Mi"
  limits:
    cpu: "2"
    memory: "4Gi"
```

### 3. Compact and Defragment etcd Database

```bash
# Get current revision
rev=$(sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status -w json | grep -o '"revision":[0-9]*' | grep -o '[0-9]*')

# Compact etcd to reclaim space
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  compact $rev

# Defragment to reclaim space
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  defrag
```

### 4. Recover from Quorum Loss

If quorum is lost, you'll need to recover from backup:

```bash
# Stop etcd on all nodes
sudo systemctl stop etcd  # or remove etcd pods

# Restore from backup
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot restore /path/to/backup/snapshot \
  --data-dir=/var/lib/etcd-backup

# Update etcd to use restored data
sudo mv /var/lib/etcd-backup /var/lib/etcd

# Start etcd on all nodes
sudo systemctl start etcd  # or recreate etcd pods
```

### 5. Fix Certificate Issues

If certificates have expired:

```bash
# For kubeadm clusters
sudo kubeadm certs renew all

# For non-kubeadm clusters, regenerate etcd certificates
# This depends on your certificate management system
```

### 6. Replace Failed etcd Member

If a member has failed:

```bash
# Remove the failed member
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member remove <member-id>

# Add a new member
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member add <new-member-name> --peer-urls=https://<new-member-ip>:2380
```

### 7. Configure Proper Quotas

Set appropriate quotas to prevent etcd database from growing too large:

```bash
# In etcd configuration, add quota-backend-bytes
# For kubeadm
sudo vi /etc/kubernetes/manifests/etcd.yaml
# Add to command:
# --quota-backend-bytes=8589934592  # 8GB
```

## Prevention

1. **Regular Backups**: Schedule regular etcd backups
2. **Monitoring**: Implement monitoring for etcd performance and health
3. **Use Fast Storage**: Use SSD or faster storage for etcd data
4. **Proper Sizing**: Allocate sufficient resources for etcd
5. **Regular Compaction**: Schedule regular compaction and defragmentation
6. **Certificate Management**: Monitor certificate expiration and renew before they expire
7. **Multi-Node etcd**: Run etcd as a cluster with at least 3 nodes
8. **Network Quality**: Ensure low-latency network between etcd nodes
9. **Isolated Workload**: Run etcd on dedicated nodes where possible
10. **Regular Testing**: Test etcd backup and recovery procedures regularly

## Related Runbooks

* [API Server Unavailable](./api-server-unavailable.md)
* [Certificate Issues](./certificate-issues.md)
* [Node Not Ready](./node-not-ready.md)
* [Cluster Backup and Restore](../security/cluster-backup-restore.md)
* [Kubelet Issues](./kubelet-issues.md)