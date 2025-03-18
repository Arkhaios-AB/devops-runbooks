# Troubleshooting Storage Capacity Issues

## Symptoms

* Persistent Volume Claims (PVCs) failing with "no space left" errors
* Pods unable to write to volumes due to capacity issues
* Storage expansion operations failing
* Nodes reporting disk pressure conditions
* Cluster autoscaler unable to provision new nodes due to storage limitations
* Applications experiencing performance degradation due to storage saturation
* Cloud provider quotas preventing new volume creation
* Error messages about "volume capacity exceeded" in application logs
* Automatic volume expansion triggered but stuck
* StatefulSet pods unable to launch due to storage constraints

## Possible Causes

1. **Physical Storage Exhaustion**: Underlying storage infrastructure running out of space
2. **Cloud Provider Quotas**: Hitting storage limits imposed by cloud providers
3. **Node Disk Pressure**: Local node storage (not necessarily PVs) running out of space
4. **Volume Expansion Failures**: Issues with extending existing volumes
5. **StorageClass Configuration Issues**: Misconfigurations preventing capacity management
6. **Resource Quota Limits**: Namespace resource quotas restricting storage use
7. **Filesystem Capacity vs. Raw Capacity Mismatch**: Filesystem overhead reducing available space
8. **Incorrect Capacity Planning**: Underestimated storage needs in application design
9. **Storage Fragmentation**: Inefficient storage utilization due to fragmentation
10. **Backup or Snapshot Retention**: Excessive backup/snapshot retention consuming space

## Diagnosis Steps

### 1. Check PVC and PV Status

```bash
# List all PVCs and their status
kubectl get pvc -A

# Check PVC details
kubectl describe pvc <pvc-name> -n <namespace>

# List PVs and check capacity/used space
kubectl get pv
```

### 2. Check Node Disk Pressure

```bash
# Check if nodes are experiencing disk pressure
kubectl get nodes | grep -i pressure

# Get detailed node conditions
kubectl describe node <node-name> | grep -A 5 Conditions

# Check node storage usage
kubectl top nodes

# Check kubelet logs for disk pressure events
kubectl logs -n kube-system <kubelet-pod> | grep -i "disk pressure"
```

### 3. Check Cloud Provider Quotas and Usage

#### AWS

```bash
# Check EBS volume limits
aws service-quotas get-service-quota --service-code ec2 --quota-code L-D18FCD1D --region <region>

# List current EBS volumes and sizes
aws ec2 describe-volumes --region <region> --query "Volumes[*].[VolumeId,Size,State]" --output table
```

#### GCP

```bash
# Check disk quotas
gcloud compute project-info describe --project <project-id> | grep -i quota

# List persistent disks and sizes
gcloud compute disks list --format="table(name,sizeGb,zone,status)" --project <project-id>
```

#### Azure

```bash
# Check disk quota usage
az vm list-usage --location <location> | grep -i disk

# List existing disks
az disk list --query "[].{Name:name, Size:diskSizeGb, SkuTier:sku.tier}" -o table
```

### 4. Check Storage Backend Status

For on-premises clusters, check the underlying storage system:

```bash
# For Ceph
ceph df
ceph health detail

# For NFS
df -h /path/to/nfs/export

# For iSCSI
iscsiadm -m session -P 3
```

### 5. Check Volume Expansion Capabilities

```bash
# Check if StorageClass supports volume expansion
kubectl get sc <storage-class-name> -o jsonpath='{.allowVolumeExpansion}'

# Check for pending volume resize operations
kubectl describe pvc <pvc-name> -n <namespace> | grep -i "resize"
```

### 6. Check Namespace Resource Quotas

```bash
# List resource quotas in namespace
kubectl get resourcequota -n <namespace>

# Check detailed quota usage
kubectl describe resourcequota <quota-name> -n <namespace>
```

### 7. Check if Filesystem is Full Inside Volume

```bash
# Find pod using the PVC
POD_NAME=$(kubectl get pods -n <namespace> -o json | jq -r '.items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName=="<pvc-name>") | .metadata.name')

# Check filesystem usage inside pod
kubectl exec -n <namespace> $POD_NAME -- df -h
```

## Resolution Steps

### 1. Expand Existing Volumes

If the StorageClass supports volume expansion:

```bash
# Check if volume expansion is supported
kubectl get sc <storage-class-name> -o jsonpath='{.allowVolumeExpansion}'

# If supported, resize PVC
kubectl patch pvc <pvc-name> -n <namespace> --type='json' -p='[{"op": "replace", "path": "/spec/resources/requests/storage", "value": "20Gi"}]'

# Monitor the resize operation
kubectl describe pvc <pvc-name> -n <namespace>
```

If filesystem expansion is needed inside the pod:

```bash
# For ext4 filesystem
kubectl exec -n <namespace> <pod-name> -- resize2fs /path/to/mount

# For XFS filesystem
kubectl exec -n <namespace> <pod-name> -- xfs_growfs /path/to/mount
```

### 2. Migrate Data to Larger Volume

If volume expansion is not supported:

```bash
# Create new larger PVC
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: <new-pvc-name>
  namespace: <namespace>
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: <storage-class-name>
  resources:
    requests:
      storage: 20Gi
EOF

# Create data migration job
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: volume-migration-job
  namespace: <namespace>
spec:
  template:
    spec:
      containers:
      - name: migration
        image: busybox
        command: ["sh", "-c", "cp -av /source/* /target/ && echo 'Migration complete'"]
        volumeMounts:
        - name: source-volume
          mountPath: /source
        - name: target-volume
          mountPath: /target
      restartPolicy: Never
      volumes:
      - name: source-volume
        persistentVolumeClaim:
          claimName: <old-pvc-name>
      - name: target-volume
        persistentVolumeClaim:
          claimName: <new-pvc-name>
EOF

# Check migration job status
kubectl get jobs -n <namespace>
kubectl logs -n <namespace> -l job-name=volume-migration-job
```

After successful migration, update the deployment to use the new PVC.

### 3. Request Cloud Provider Quota Increase

#### AWS

```bash
# Request quota increase via AWS console or CLI
aws support create-case \
  --subject "EBS Volume Quota Increase Request" \
  --service-code "amazon-elastic-compute-cloud-linux" \
  --category-code "quota-increase" \
  --communication-body "Please increase our EBS volume quota in region <region> from X to Y." \
  --severity-code "normal"
```

#### GCP

```bash
# Request quota increase via GCP console or API
# Sample quota increase request
gcloud compute project-info update --project <project-id> \
  --quota "DISKS_TOTAL_GB:5000"
```

#### Azure

```bash
# Request quota increase through Azure portal
# Or via Azure CLI (where available)
az disk request-quota-increase --disk-type Premium_LRS --region <region> --new-limit 10000
```

### 4. Implement Storage Cleanup

```bash
# Create a cleanup job for temporary files
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: storage-cleanup-job
  namespace: <namespace>
spec:
  template:
    spec:
      containers:
      - name: cleanup
        image: busybox
        command: ["sh", "-c", "find /data -name '*.tmp' -type f -delete && find /data -name '*.log' -type f -mtime +30 -delete"]
        volumeMounts:
        - name: data-volume
          mountPath: /data
      restartPolicy: Never
      volumes:
      - name: data-volume
        persistentVolumeClaim:
          claimName: <pvc-name>
EOF
```

### 5. Enable Volume Expansion in StorageClass

If volume expansion is not enabled:

```bash
# Patch StorageClass to enable volume expansion
kubectl patch storageclass <storage-class-name> \
  --type='json' \
  -p='[{"op": "replace", "path": "/allowVolumeExpansion", "value": true}]'
```

Note: This will only affect future PVCs, not existing ones.

### 6. Adjust Resource Quotas

```bash
# Update resource quota for storage
kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: <namespace>
spec:
  hard:
    requests.storage: "100Gi"
    persistentvolumeclaims: "20"
EOF
```

### 7. Implement Storage Tiering

For applications with mixed storage needs:

```bash
# Create StorageClass for fast but limited storage
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: <provisioner>
parameters:
  type: <fast-storage-type>
reclaimPolicy: Delete
allowVolumeExpansion: true
EOF

# Create StorageClass for slower but larger storage
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: capacity-storage
provisioner: <provisioner>
parameters:
  type: <capacity-storage-type>
reclaimPolicy: Delete
allowVolumeExpansion: true
EOF
```

Then implement application logic to use the appropriate storage class for different types of data.

### 8. Configure Automatic Storage Scaling

For cloud environments that support it:

```bash
# Example using AWS EFS for auto-scaling storage
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: <fs-id>
  directoryPerms: "700"
EOF
```

### 9. Implement Storage Monitoring

Deploy Prometheus and Grafana with storage monitoring:

```bash
# Add storage monitoring rules to Prometheus
kubectl apply -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: storage-alerts
  namespace: monitoring
spec:
  groups:
  - name: storage
    rules:
    - alert: VolumeFillingUp
      expr: kubelet_volume_stats_available_bytes / kubelet_volume_stats_capacity_bytes < 0.15
      for: 1m
      labels:
        severity: warning
      annotations:
        summary: "Volume is filling up (< 15% left)"
        description: "Volume {{ \$labels.persistentvolumeclaim }} has less than 15% free."
EOF
```

## Prevention

1. **Implement Proper Capacity Planning**: Plan storage needs based on growth projections
   ```bash
   # Example capacity planning document structure
   cat <<EOF > storage-capacity-plan.md
   # Storage Capacity Plan
   
   ## Current Usage
   - Total cluster storage: 500 GB
   - Current utilization: 300 GB (60%)
   
   ## Growth Projections
   - Expected monthly growth: 10%
   - 6-month projection: 530 GB
   - 12-month projection: 850 GB
   
   ## Planned Expansions
   - Increase storage quota to 1000 GB by Q3
   - Implement tiered storage by Q2
   - Deploy automatic backup pruning by Q1
   EOF
   ```

2. **Set Up Storage Monitoring**: Deploy monitoring for storage usage
   ```bash
   # Prometheus storage monitoring queries
   # PV usage by PVC:
   # kubelet_volume_stats_used_bytes{namespace="<namespace>"} / kubelet_volume_stats_capacity_bytes{namespace="<namespace>"}
   ```

3. **Implement Automatic Alerting**: Create alerts for storage thresholds
   ```bash
   # Alert when PVC usage exceeds 80%
   kubelet_volume_stats_used_bytes{namespace="<namespace>"} / kubelet_volume_stats_capacity_bytes{namespace="<namespace>"} > 0.8
   ```

4. **Data Lifecycle Management**: Implement policies for data cleanup
   ```bash
   # Example Kubernetes CronJob for log rotation
   kubectl apply -f - <<EOF
   apiVersion: batch/v1
   kind: CronJob
   metadata:
     name: log-cleanup
     namespace: <namespace>
   spec:
     schedule: "0 1 * * *"
     jobTemplate:
       spec:
         template:
           spec:
             containers:
             - name: log-cleaner
               image: busybox
               args:
               - /bin/sh
               - -c
               - find /data -name "*.log" -type f -mtime +7 -delete
               volumeMounts:
               - name: log-volume
                 mountPath: /data
             restartPolicy: OnFailure
             volumes:
             - name: log-volume
               persistentVolumeClaim:
                 claimName: <log-pvc-name>
   EOF
   ```

5. **Implement StorageClass Quotas**: Define limits for storage classes
   ```bash
   # Example ResourceQuota for specific StorageClass
   kubectl apply -f - <<EOF
   apiVersion: v1
   kind: ResourceQuota
   metadata:
     name: ssd-quota
     namespace: <namespace>
   spec:
     hard:
       <storage-class-name>.storageclass.storage.k8s.io/requests.storage: 100Gi
   EOF
   ```

6. **Enable Dynamic Volume Expansion**: Configure StorageClasses to support expansion
   ```bash
   # Update StorageClass to enable volume expansion
   kubectl patch storageclass <storage-class-name> \
     --type='json' \
     -p='[{"op": "replace", "path": "/allowVolumeExpansion", "value": true}]'
   ```

7. **Implement Backup Retention Policies**: Set appropriate retention for backups
   ```bash
   # For Velero backup retention
   velero schedule create daily-backup --schedule="0 0 * * *" --ttl 168h0m0s
   ```

8. **Utilize Compression and Deduplication**: Enable where available
9. **Regular Storage Audits**: Conduct regular reviews of storage usage
10. **Storage Documentation**: Document storage architecture and scaling procedures

## Related Runbooks

* [PVC Stuck in Pending](./pvc-pending.md)
* [Storage Class Issues](./storage-class-issues.md)
* [Dynamic Provisioning Issues](./dynamic-provisioning-issues.md)
* [Volume Mount Problems](./volume-mount-problems.md)
* [Node Disk Pressure](../cluster/node-disk-pressure.md)
* [Cloud Storage Issues](./cloud-storage-issues.md)