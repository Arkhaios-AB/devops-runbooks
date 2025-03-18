# Kubernetes Cluster Backup and Restore

## Symptoms

* Need to recover from cluster data loss
* Planning disaster recovery procedures
* Preparing for cluster upgrades
* Migrating to a new cluster
* Need to rollback after a failed upgrade
* Recovering from etcd data corruption
* Restoring after accidental deletion of critical resources
* Recovery after control plane failure
* Preparing for security incident recovery
* Compliance requirements for data retention

## Possible Causes

1. **Planned Maintenance**: Upgrades, migrations, or configuration changes
2. **Data Corruption**: etcd database corruption
3. **Accidental Deletion**: Accidentally deleted critical resources
4. **Hardware Failure**: Control plane node failures
5. **Security Incidents**: Compromised clusters requiring restore to a clean state
6. **Network Partitions**: Split-brain scenarios in etcd
7. **Disk Failures**: Storage issues on etcd or control plane nodes
8. **Human Error**: Misconfiguration of critical components
9. **Certificate Expiry**: Expired certificates causing cluster failure
10. **Cloud Provider Issues**: Cloud infrastructure failures

## Diagnosis Steps

### 1. Assess Backup Status

```bash
# Check if regular backups exist
ls -la /path/to/backup/location

# Check backup dates to find the most recent valid backup
find /path/to/backup/location -type f -name "*.db" -o -name "*.snapshot" | sort -r
```

### 2. Verify Cluster Status Before Restore

```bash
# Check cluster component status
kubectl get componentstatuses

# Check node status
kubectl get nodes

# Check etcd cluster health (if accessible)
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health
```

### 3. Verify Backup Integrity

```bash
# For etcd snapshots, check status
sudo ETCDCTL_API=3 etcdctl --write-out=table snapshot status /path/to/backup/snapshot.db

# For resource backups (e.g., with Velero), check backup details
velero backup describe <backup-name>

# For kubeadm clusters, check if all certificate files are present in backup
ls -la /path/to/backup/pki/
```

### 4. Document Current Configuration

Before restoring, document current configuration for reference:

```bash
# Save current cluster configuration
kubectl -n kube-system get configmap kubeadm-config -o yaml > kubeadm-config-before-restore.yaml

# Save current pod information
kubectl get pods -A -o wide > pods-before-restore.txt

# Save current service information
kubectl get svc -A > services-before-restore.txt

# Save current persistent volume information
kubectl get pv,pvc -A > volumes-before-restore.txt
```

## Resolution Steps

### 1. Backup etcd (Before Any Restore)

Always take a backup before attempting to restore, even if restoring from a previous backup:

```bash
# For kubeadm-based clusters
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /path/to/new-backup/etcd-snapshot-$(date +%Y-%m-%d-%H-%M).db
```

### 2. Restore etcd from Backup

#### For kubeadm-based clusters:

```bash
# Stop etcd and other control plane components
sudo -i
cd /etc/kubernetes/manifests/
mv kube-apiserver.yaml kube-controller-manager.yaml kube-scheduler.yaml etcd.yaml /tmp/

# Verify etcd is stopped
docker ps | grep etcd

# Restore snapshot to a temporary directory
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot restore /path/to/backup/snapshot.db \
  --data-dir=/var/lib/etcd-backup

# Backup current etcd data directory
mv /var/lib/etcd /var/lib/etcd.bak

# Replace with restored data
mv /var/lib/etcd-backup /var/lib/etcd
chown -R etcd:etcd /var/lib/etcd

# Restore manifests to restart control plane
mv /tmp/{etcd.yaml,kube-apiserver.yaml,kube-controller-manager.yaml,kube-scheduler.yaml} /etc/kubernetes/manifests/

# Wait for control plane to start
watch crictl ps
```

#### For non-kubeadm clusters:

```bash
# Stop etcd service
sudo systemctl stop etcd

# Restore snapshot to a temporary directory
sudo ETCDCTL_API=3 etcdctl snapshot restore /path/to/backup/snapshot.db \
  --data-dir=/var/lib/etcd-backup \
  --name=<etcd-name> \
  --initial-cluster=<etcd-initial-cluster> \
  --initial-cluster-token=<etcd-initial-cluster-token> \
  --initial-advertise-peer-urls=<etcd-initial-advertise-peer-urls>

# Backup current etcd data directory
sudo mv /var/lib/etcd /var/lib/etcd.bak

# Replace with restored data
sudo mv /var/lib/etcd-backup /var/lib/etcd
sudo chown -R etcd:etcd /var/lib/etcd

# Start etcd service
sudo systemctl start etcd
```

### 3. Restore Kubernetes Resources (Using Velero)

If using Velero for application-level backup:

```bash
# List available backups
velero backup get

# Restore all resources from a backup
velero restore create --from-backup <backup-name>

# Restore specific namespace
velero restore create --from-backup <backup-name> --include-namespaces <namespace>

# Restore specific resources
velero restore create --from-backup <backup-name> --include-resources deployments,services
```

### 4. Restore Using Resource Files

If you have YAML exports of your resources:

```bash
# Apply all resource files from a directory
kubectl apply -f /path/to/resource/files/

# Apply specific high-priority resources first
kubectl apply -f /path/to/resource/files/namespaces/
kubectl apply -f /path/to/resource/files/crds/
kubectl apply -f /path/to/resource/files/rbac/
kubectl apply -f /path/to/resource/files/pv-pvc/
kubectl apply -f /path/to/resource/files/deployments/
```

### 5. Restore Certificates (For kubeadm clusters)

If certificates were lost or corrupted:

```bash
# Restore PKI directory from backup
sudo cp -r /path/to/backup/pki /etc/kubernetes/

# Set correct permissions
sudo chmod 600 /etc/kubernetes/pki/ca.key
sudo chmod 600 /etc/kubernetes/pki/etcd/ca.key

# If certificates are expired, renew them
sudo kubeadm certs renew all
```

### 6. Validate Restored Cluster

```bash
# Check control plane component status
kubectl get componentstatuses

# Check node status
kubectl get nodes

# Check system pods
kubectl get pods -n kube-system

# Check deployed workloads
kubectl get pods -A
```

### 7. Reconfigure Cluster-Specific Settings (If Needed)

If restoring to a different infrastructure:

```bash
# Update cloud provider configs
kubectl edit configmap -n kube-system cloud-config

# Update node taints/labels if needed
kubectl label node <node-name> <key>=<value>
kubectl taint node <node-name> <key>=<value>:<effect>
```

## Prevention

1. **Regular Automated Backups**: Set up scheduled etcd backups
   ```bash
   # Create a cron job for etcd backup
   echo "0 3 * * * root ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
     --cacert=/etc/kubernetes/pki/etcd/ca.crt \
     --cert=/etc/kubernetes/pki/etcd/server.crt \
     --key=/etc/kubernetes/pki/etcd/server.key \
     snapshot save /path/to/backup/etcd-snapshot-\$(date +%Y-%m-%d).db" > /etc/cron.d/etcd-backup
   ```

2. **Resource Export Backups**: Regularly export critical resources
   ```bash
   # Create script for resource backup
   cat <<EOF > /usr/local/bin/k8s-resource-backup.sh
   #!/bin/bash
   BACKUP_DIR="/path/to/backup/resources/\$(date +%Y-%m-%d)"
   mkdir -p \$BACKUP_DIR
   
   # Export all namespaces
   kubectl get namespaces -o yaml > \$BACKUP_DIR/namespaces.yaml
   
   # Export all resources per namespace
   for NS in \$(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}'); do
     mkdir -p \$BACKUP_DIR/\$NS
     for RESOURCE in deployments statefulsets daemonsets configmaps secrets services ingress pv pvc; do
       kubectl get \$RESOURCE -n \$NS -o yaml > \$BACKUP_DIR/\$NS/\$RESOURCE.yaml
     done
   done
   EOF
   
   chmod +x /usr/local/bin/k8s-resource-backup.sh
   echo "0 4 * * * root /usr/local/bin/k8s-resource-backup.sh" > /etc/cron.d/k8s-resource-backup
   ```

3. **Use Velero**: Implement Velero for comprehensive backup solution
   ```bash
   # Install Velero CLI
   wget https://github.com/vmware-tanzu/velero/releases/download/v1.8.0/velero-v1.8.0-linux-amd64.tar.gz
   tar -xvf velero-v1.8.0-linux-amd64.tar.gz
   mv velero-v1.8.0-linux-amd64/velero /usr/local/bin/
   
   # Install Velero in cluster (AWS example)
   velero install \
     --provider aws \
     --plugins velero/velero-plugin-for-aws:v1.3.0 \
     --bucket <bucket-name> \
     --backup-location-config region=<region> \
     --snapshot-location-config region=<region> \
     --secret-file ./credentials-velero
   
   # Set up scheduled backup
   velero schedule create daily-backup --schedule="0 1 * * *" --ttl 720h0m0s
   ```

4. **Backup Certificate Files**: Regularly backup PKI directory
   ```bash
   # Create a script for PKI backup
   cat <<EOF > /usr/local/bin/k8s-cert-backup.sh
   #!/bin/bash
   BACKUP_DIR="/path/to/backup/certificates/\$(date +%Y-%m-%d)"
   mkdir -p \$BACKUP_DIR
   cp -r /etc/kubernetes/pki \$BACKUP_DIR/
   chmod -R 600 \$BACKUP_DIR/pki/*.key \$BACKUP_DIR/pki/etcd/*.key
   EOF
   
   chmod +x /usr/local/bin/k8s-cert-backup.sh
   echo "0 5 * * * root /usr/local/bin/k8s-cert-backup.sh" > /etc/cron.d/k8s-cert-backup
   ```

5. **Document Restore Process**: Create and test a detailed restore runbook
6. **Multi-Location Backups**: Store backups in multiple locations
7. **Backup Validation**: Regularly validate backups by performing test restores
8. **Encryption**: Encrypt sensitive backup data
9. **Retention Policy**: Implement backup retention policies
10. **Immutable Backups**: Use immutable storage for backups when possible

## Related Runbooks

* [etcd Issues](../cluster/etcd-issues.md)
* [API Server Unavailable](../cluster/api-server-unavailable.md)
* [Certificate Issues](../cluster/certificate-issues.md)
* [Node Not Ready](../cluster/node-not-ready.md)
* [PV Reclaim Policy Issues](../storage/pv-reclaim-policy-issues.md)