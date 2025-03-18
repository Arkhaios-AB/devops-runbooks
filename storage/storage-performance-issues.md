# Troubleshooting Storage Performance Issues

## Symptoms

* Slow application response times due to I/O operations
* Increased latency for database queries
* Inconsistent read/write speeds for persistent volumes
* High iowait on nodes
* Timeout errors in application logs related to storage operations
* Sporadic application crashes during heavy I/O operations
* Degraded performance during peak usage times
* Backup operations taking longer than expected
* Slow pod startup time when volumes are attached
* Application metrics showing increased storage operation durations
* Throttling errors from cloud storage providers

## Possible Causes

1. **Insufficient IOPS/Throughput**: Storage class or volume type with inadequate performance
2. **Cloud Provider Throttling**: Hitting rate limits or throughput caps
3. **Network Bottlenecks**: Network congestion affecting storage traffic
4. **Storage Backend Overload**: Shared storage system under high load
5. **Filesystem Fragmentation**: High fragmentation reducing performance
6. **Improper Storage Configuration**: Misconfigured volumes or storage classes
7. **Node Resource Contention**: Multiple high I/O workloads on the same node
8. **Storage Driver Issues**: CSI driver or storage plugin problems
9. **Resource Limits**: Pod with insufficient CPU/memory affecting I/O performance
10. **Distance/Latency Issues**: Cross-zone or region storage access

## Diagnosis Steps

### 1. Check Storage Performance Metrics

```bash
# If using Prometheus/Grafana, query volume metrics
# Example Prometheus queries:
# - kubelet_volume_stats_used_bytes
# - kubelet_volume_stats_capacity_bytes
# - node_disk_io_time_seconds_total (rate over time)
# - node_disk_read_time_seconds_total (rate over time)
# - node_disk_write_time_seconds_total (rate over time)
```

### 2. Measure Current Performance

```bash
# Create a benchmark pod
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: storage-benchmark
  namespace: <namespace>
spec:
  containers:
  - name: benchmark
    image: nixery.dev/shell/fio/hdparm
    command: 
    - "/bin/sh"
    - "-c"
    - "sleep 3600"
    volumeMounts:
    - name: test-volume
      mountPath: /data
  volumes:
  - name: test-volume
    persistentVolumeClaim:
      claimName: <pvc-name>
EOF

# Run benchmark tests
# Sequential read performance
kubectl exec -n <namespace> storage-benchmark -- hdparm -t /data

# Sequential write performance
kubectl exec -n <namespace> storage-benchmark -- dd if=/dev/zero of=/data/test bs=1M count=1024 oflag=dsync

# Random I/O performance with fio
kubectl exec -n <namespace> storage-benchmark -- fio --name=randread --ioengine=libaio --iodepth=16 --rw=randread --bs=4k --direct=1 --size=1G --numjobs=4 --runtime=60 --group_reporting --directory=/data
```

### 3. Check Node I/O Stats

```bash
# Check I/O stats on the node where the pod is running
NODE=$(kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.nodeName}')

# SSH to node and check I/O stats
ssh $NODE "iostat -x 1 10"

# Check if iowait is high
ssh $NODE "mpstat -P ALL 1 5 | grep -i '%iowait'"
```

### 4. Check Cloud Provider Storage Performance Settings

#### AWS EBS

```bash
# Get volume ID from PV
VOLUME_ID=$(kubectl get pv <pv-name> -o jsonpath='{.spec.awsElasticBlockStore.volumeID}' | cut -d '/' -f 4)

# Check volume type and IOPS
aws ec2 describe-volumes --volume-ids $VOLUME_ID --region <region> --query "Volumes[*].[VolumeType,Iops,Throughput]" --output table
```

#### GCP Persistent Disk

```bash
# Get disk name from PV
DISK_NAME=$(kubectl get pv <pv-name> -o jsonpath='{.spec.gcePersistentDisk.pdName}')

# Check disk type
gcloud compute disks describe $DISK_NAME --zone <zone> --format="table(name,type,sizeGb)"
```

#### Azure Disk

```bash
# Get disk name from PV
DISK_NAME=$(kubectl get pv <pv-name> -o jsonpath='{.spec.azureDisk.diskName}')

# Check disk SKU
az disk show --name $DISK_NAME --resource-group <resource-group> --query "sku" -o table
```

### 5. Check for I/O Throttling

```bash
# Check for throttling events in cloud provider
# For AWS, check CloudWatch metrics for the volume
aws cloudwatch get-metric-statistics --namespace AWS/EBS --metric-name VolumeQueueLength --dimensions Name=VolumeId,Value=$VOLUME_ID --start-time $(date -u -v-1d +%Y-%m-%dT%H:%M:%SZ) --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) --period 300 --statistics Average

# For Azure, check metrics via Azure CLI or portal
```

### 6. Check Storage Class Configuration

```bash
# Get StorageClass used by PVC
SC_NAME=$(kubectl get pvc <pvc-name> -n <namespace> -o jsonpath='{.spec.storageClassName}')

# Check StorageClass configuration
kubectl describe storageclass $SC_NAME
```

### 7. Check for Filesystem Issues

```bash
# Check filesystem type and status
kubectl exec -n <namespace> storage-benchmark -- df -Th /data

# Check for fragmentation (ext4)
kubectl exec -n <namespace> storage-benchmark -- e4defrag -c /data
```

### 8. Check Pod Resource Limits

```bash
# Check if pod has appropriate CPU/memory limits
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[0].resources}'
```

## Resolution Steps

### 1. Upgrade Storage Class/Volume Type

#### AWS EBS

```bash
# Create a new StorageClass with higher performance
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: high-perf-storage
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iopsPerGB: "50"
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF

# For existing volumes, create snapshot
aws ec2 create-snapshot --volume-id $VOLUME_ID --description "Backup before upgrading volume type" --region <region>

# Modify volume type (for io1/io2/gp3)
aws ec2 modify-volume --volume-id $VOLUME_ID --volume-type io2 --iops 4000 --region <region>
```

#### GCP Persistent Disk

```bash
# Create new StorageClass with SSD
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: high-perf-storage
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF

# For existing disks, create snapshot and migrate
gcloud compute disks snapshot $DISK_NAME --snapshot-names=${DISK_NAME}-snapshot --zone <zone>

# Create new disk from snapshot with higher performance
gcloud compute disks create ${DISK_NAME}-new --source-snapshot=${DISK_NAME}-snapshot --type=pd-ssd --zone <zone>
```

#### Azure Disk

```bash
# Create new StorageClass with Premium SSD
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: high-perf-storage
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF

# For existing disks, create snapshot and migrate
az snapshot create --name ${DISK_NAME}-snapshot --source $DISK_NAME --resource-group <resource-group>

# Create new disk from snapshot
az disk create --name ${DISK_NAME}-new --sku Premium_LRS --source ${DISK_NAME}-snapshot --resource-group <resource-group>
```

### 2. Optimize Filesystem Settings

```bash
# Create a tuning job
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: fs-tuning-job
  namespace: <namespace>
spec:
  template:
    spec:
      containers:
      - name: tuning
        image: ubuntu
        command: ["/bin/sh", "-c"]
        args:
        - |
          # Mount with noatime to reduce unnecessary writes
          mount -o remount,noatime /data
          
          # For ext4, adjust filesystem parameters
          tune2fs -o journal_data_writeback /dev/$(findmnt -n -o SOURCE --target /data | cut -d'/' -f3)
          
          # Increase commit interval for ext4
          echo 60 > /proc/sys/vm/dirty_writeback_centisecs
          
          echo "Filesystem tuning completed"
          sleep 30
        securityContext:
          privileged: true
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

### 3. Implement Storage Anti-Affinity

To prevent high I/O workloads from competing:

```bash
# Update deployment to use pod anti-affinity
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <deployment-name>
  namespace: <namespace>
spec:
  replicas: 3
  selector:
    matchLabels:
      app: <app-label>
  template:
    metadata:
      labels:
        app: <app-label>
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - <app-label>
                  - other-high-io-app
              topologyKey: kubernetes.io/hostname
      containers:
      - name: <container-name>
        image: <image>
        # ... other container settings
        volumeMounts:
        - name: data-volume
          mountPath: /data
      volumes:
      - name: data-volume
        persistentVolumeClaim:
          claimName: <pvc-name>
EOF
```

### 4. Adjust Application Resource Limits

```bash
# Update pod/deployment with appropriate CPU/memory
kubectl patch deployment <deployment-name> -n <namespace> --type='json' -p='[
  {"op": "replace", "path": "/spec/template/spec/containers/0/resources", "value": 
    {"requests": {"cpu": "1", "memory": "2Gi"}, "limits": {"cpu": "2", "memory": "4Gi"}}
  }
]'
```

### 5. Implement Read-Write Splitting

For databases and applications that support it:

```bash
# Example for MySQL with read replicas
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: mysql-read
  namespace: <namespace>
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
    readonly: "true"
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-write
  namespace: <namespace>
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
    readonly: "false"
EOF
```

### 6. Implement Storage Caching

```bash
# Example - Deploy Redis as a cache
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
  namespace: <namespace>
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis-cache
  template:
    metadata:
      labels:
        app: redis-cache
    spec:
      containers:
      - name: redis
        image: redis:6
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 1
            memory: 2Gi
        ports:
        - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis-cache
  namespace: <namespace>
spec:
  ports:
  - port: 6379
  selector:
    app: redis-cache
EOF
```

### 7. Optimize Cloud Storage Configuration

#### AWS EBS with gp3

```bash
# Modify gp3 volume with higher throughput and IOPS
aws ec2 modify-volume \
  --volume-id $VOLUME_ID \
  --volume-type gp3 \
  --iops 16000 \
  --throughput 1000 \
  --region <region>
```

#### GCP Regional Persistent Disk

```bash
# Create new StorageClass with regional PD for higher availability
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: regional-pd-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd
  # Specify two zones within the region
  zones: <zone1>,<zone2>
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF
```

#### Azure Ultra Disk

```bash
# Create StorageClass for ultra high-performance workloads
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ultra-disk
provisioner: disk.csi.azure.com
parameters:
  skuName: UltraSSD_LRS
  diskIopsReadWrite: "5000"
  diskMbpsReadWrite: "200"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF
```

### 8. Implement Local Volumes for High-Performance Workloads

```bash
# Create local volume StorageClass
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
EOF

# Create PersistentVolume using local storage
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 100Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - <node-name>
EOF

# Create PVC for local storage
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: high-perf-local-pvc
  namespace: <namespace>
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: local-storage
EOF
```

### 9. Adjust Filesystem Mount Options

```bash
# Create a new PV with optimized mount options
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: optimized-pv
  annotations:
    pv.kubernetes.io/provisioned-by: <provisioner>
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: <storage-class-name>
  mountOptions:
    - noatime
    - nodiratime
    - nobarrier
  csi:
    driver: <csi-driver>
    volumeHandle: <volume-id>
    fsType: ext4
EOF
```

## Prevention

1. **Implement Storage Performance Testing**: Set up regular storage benchmark tests
   ```bash
   # Create a CronJob for regular storage benchmarking
   kubectl apply -f - <<EOF
   apiVersion: batch/v1
   kind: CronJob
   metadata:
     name: storage-benchmark
     namespace: <namespace>
   spec:
     schedule: "0 0 * * 0"  # Weekly
     jobTemplate:
       spec:
         template:
           spec:
             containers:
             - name: benchmark
               image: nixery.dev/shell/fio
               command: ["/bin/sh", "-c"]
               args:
               - |
                 fio --name=readwrite --ioengine=libaio --iodepth=32 --rw=randrw --rwmixread=70 --bs=4k --direct=1 --size=1G --numjobs=4 --runtime=60 --time_based --group_reporting --directory=/data
                 exit 0
               volumeMounts:
               - name: test-volume
                 mountPath: /data
             restartPolicy: OnFailure
             volumes:
             - name: test-volume
               persistentVolumeClaim:
                 claimName: <pvc-name>
   EOF
   ```

2. **Set Up Storage Performance Monitoring**: Implement comprehensive monitoring
   ```bash
   # Example Prometheus StorageClass usage alert
   cat <<EOF > storage-performance-alert.yaml
   apiVersion: monitoring.coreos.com/v1
   kind: PrometheusRule
   metadata:
     name: storage-performance-alert
     namespace: monitoring
   spec:
     groups:
     - name: storage-performance
       rules:
       - alert: HighIoWait
         expr: rate(node_cpu_seconds_total{mode="iowait"}[5m]) > 0.2
         for: 5m
         labels:
           severity: warning
         annotations:
           summary: "High I/O wait on node {{ \$labels.instance }}"
           description: "Node {{ \$labels.instance }} has high I/O wait (> 20%) for more than 5 minutes."
   EOF
   kubectl apply -f storage-performance-alert.yaml
   ```

3. **Use Storage Tiering**: Define appropriate storage classes for different performance needs
   ```bash
   cat <<EOF > storage-policy.md
   # Storage Policy
   
   ## Performance Tiers
   
   1. **high-performance-storage**
      - Use for: Databases, message queues, and latency-sensitive applications
      - Type: SSD/NVMe-based storage
      - IOPS: Min 3000
      - Throughput: Min 125 MB/s
   
   2. **standard-storage**
      - Use for: Application servers, web services
      - Type: SSD or hybrid storage
      - IOPS: Min 1000
      - Throughput: Min 50 MB/s
   
   3. **capacity-storage**
      - Use for: Backups, archives, large files
      - Type: HDD-based storage
      - IOPS: Best effort
      - Throughput: Min 25 MB/s
   
   ## Performance Requirements by Application
   
   - MySQL/PostgreSQL: high-performance-storage
   - Elasticsearch: high-performance-storage
   - Web servers: standard-storage
   - Log storage: capacity-storage
   EOF
   ```

4. **Set I/O-Specific Resource Limits**: Use Pod QoS to manage I/O resources
   ```bash
   # Example deployment with I/O priority annotations
   kubectl apply -f - <<EOF
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: database
     namespace: <namespace>
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: database
     template:
       metadata:
         labels:
           app: database
         annotations:
           # For Linux kernels that support it (via PSI)
           resources.alpha.kubernetes.io/io-class: "high"
           resources.alpha.kubernetes.io/io-priority: "0"
       spec:
         containers:
         - name: database
           image: postgres:14
           resources:
             requests:
               cpu: 1
               memory: 2Gi
             limits:
               cpu: 2
               memory: 4Gi
   EOF
   ```

5. **Implement Pre-Production Storage Testing**: Test performance before production
6. **Document Storage Requirements**: Define clear performance requirements for applications
7. **Regular Storage Performance Reviews**: Conduct regular reviews of storage metrics
8. **Geographic Placement Strategy**: Position storage close to compute resources
9. **Capacity Planning with Performance**: Include performance in capacity planning
10. **Implement Storage Auto-scaling**: Set up automatic scaling based on usage patterns

## Related Runbooks

* [PVC Stuck in Pending](./pvc-pending.md)
* [Storage Class Issues](./storage-class-issues.md)
* [Dynamic Provisioning Issues](./dynamic-provisioning-issues.md)
* [Storage Capacity Issues](./storage-capacity-issues.md)
* [Cloud Storage Issues](./cloud-storage-issues.md)
* [StatefulSet Storage Issues](./statefulset-storage-issues.md)