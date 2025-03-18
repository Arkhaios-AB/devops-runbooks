# Troubleshooting Cloud Storage Issues (EBS/EFS/Azure Disk)

## Symptoms

* Persistent Volume Claims (PVCs) stuck in pending state
* Volume attachment failures during pod scheduling
* Slow I/O performance on cloud storage volumes
* Volumes not detaching properly when pods terminate
* Storage quota or limit errors
* Cross-zone/region attachment issues
* Volume snapshot or backup failures
* Unexpected volume detachment during operation
* Volume encryption issues
* Billing surprises for storage consumption
* PVs not releasing when PVCs are deleted
* Inconsistent performance across different zones
* Errors related to CSI drivers for cloud storage

## Possible Causes

1. **Cloud Provider API Issues**: Temporary API failures or rate limiting
2. **CSI Driver Problems**: Outdated or misconfigured CSI drivers
3. **Quota Limitations**: Hitting cloud provider volume or API quota limits
4. **Permission Issues**: Insufficient IAM/RBAC permissions for volume operations
5. **Cross-AZ Limitations**: Attempting to use volumes across availability zones
6. **StorageClass Misconfiguration**: Incorrect parameters in StorageClass definition
7. **Cloud Infrastructure Degradation**: Underlying cloud storage service issues
8. **Volume Type Mismatch**: Using inappropriate volume types for workloads
9. **Encryption Configuration Issues**: Problems with KMS or encryption settings
10. **Node Instance Type Limitations**: Instance types with volume attachment limits

## Diagnosis Steps

### 1. Check PVC and PV Status

```bash
# List all PVCs and check status
kubectl get pvc -A

# Describe specific PVC for error details
kubectl describe pvc <pvc-name> -n <namespace>

# Check associated PV
kubectl get pv | grep <pvc-name>
kubectl describe pv <pv-name>
```

### 2. Check CSI Driver Pods

```bash
# Check CSI driver pods
kubectl get pods -n kube-system -l app=ebs-csi-controller # For AWS EBS
kubectl get pods -n kube-system -l app=efs-csi-controller # For AWS EFS
kubectl get pods -n kube-system -l app=azure-disk-csi-driver # For Azure Disk

# Check CSI driver logs
kubectl logs -n kube-system -l app=ebs-csi-controller -c ebs-plugin # For AWS EBS
kubectl logs -n kube-system -l app=efs-csi-controller -c efs-plugin # For AWS EFS
kubectl logs -n kube-system -l app=azure-disk-csi-driver -c azure-disk # For Azure Disk
```

### 3. Check Cloud Provider Health and Quotas

#### AWS

```bash
# Check AWS service health
aws health describe-events --filter eventTypeCode=AWS_EBS_DEGRADED_PERFORMANCE --region <region>

# Check EBS volume quota
aws service-quotas get-service-quota --service-code ec2 --quota-code L-D18FCD1D --region <region>

# List all EBS volumes
aws ec2 describe-volumes --filters "Name=status,Values=available" --region <region> --query 'Volumes[*].[VolumeId,Size,State]' --output table

# Check EBS volume status
aws ec2 describe-volume-status --region <region>
```

#### Azure

```bash
# Check Azure resource provider health
az resource list --resource-type Microsoft.Compute/disks

# Check disk quota limits
az vm list-usage --location <location> | grep -i disk

# List all disks
az disk list --output table
```

### 4. Check Node Volume Attachment Status

```bash
# Get node where pod is scheduled
NODE=$(kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.nodeName}')

# Describe node for attached volumes
kubectl describe node $NODE | grep -A 10 "Attached volumes"

# For AWS EBS
aws ec2 describe-instances --instance-ids <instance-id> --query "Reservations[].Instances[].BlockDeviceMappings[].DeviceName[]" --region <region>

# For Azure
az vm show -g <resource-group> -n <node-name> --query "storageProfile.dataDisks"
```

### 5. Check for Permission Issues

#### AWS

```bash
# Check IAM role used by CSI driver
ROLE_ARN=$(kubectl get serviceaccount ebs-csi-controller-sa -n kube-system -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}')

# Get the IAM role name
ROLE_NAME=$(echo $ROLE_ARN | cut -d/ -f2)

# Check IAM permissions
aws iam get-role --role-name $ROLE_NAME
aws iam list-role-policies --role-name $ROLE_NAME
aws iam list-attached-role-policies --role-name $ROLE_NAME
```

#### Azure

```bash
# Check managed identity permissions
AKS_CLUSTER_NAME=<aks-cluster-name>
RESOURCE_GROUP=<resource-group>

az aks show -g $RESOURCE_GROUP -n $AKS_CLUSTER_NAME --query "identityProfile.kubeletidentity.clientId" -o tsv
```

### 6. Check StorageClass Configuration

```bash
# List all StorageClasses
kubectl get sc

# Describe the StorageClass being used
kubectl describe sc <storage-class-name>
```

### 7. Check for Cross-AZ Issues

```bash
# Check node zones
kubectl get nodes -L topology.kubernetes.io/zone

# Check PV topology constraints
kubectl get pv <pv-name> -o jsonpath='{.spec.nodeAffinity}'
```

## Resolution Steps

### 1. Fix CSI Driver Issues

#### AWS EBS CSI Driver

```bash
# Update EBS CSI driver
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update
helm upgrade --install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.create=true \
  --set controller.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::<account-id>:role/<role-name>

# Restart CSI driver pods if needed
kubectl rollout restart deployment ebs-csi-controller -n kube-system
```

#### AWS EFS CSI Driver

```bash
# Update EFS CSI driver
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver
helm repo update
helm upgrade --install aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.create=true \
  --set controller.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::<account-id>:role/<role-name>

# Restart CSI driver pods if needed
kubectl rollout restart deployment efs-csi-controller -n kube-system
```

#### Azure Disk CSI Driver

```bash
# Update Azure Disk CSI driver
helm repo add azuredisk-csi-driver https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/master/charts
helm repo update
helm upgrade --install azuredisk-csi-driver azuredisk-csi-driver/azuredisk-csi-driver \
  --namespace kube-system

# Restart CSI driver pods if needed
kubectl rollout restart deployment csi-azuredisk-controller -n kube-system
```

### 2. Fix IAM/RBAC Permission Issues

#### AWS EBS/EFS

```bash
# Create IAM policy for EBS CSI driver
cat > ebs-csi-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateSnapshot",
        "ec2:AttachVolume",
        "ec2:DetachVolume",
        "ec2:ModifyVolume",
        "ec2:DescribeAvailabilityZones",
        "ec2:DescribeInstances",
        "ec2:DescribeSnapshots",
        "ec2:DescribeTags",
        "ec2:DescribeVolumes",
        "ec2:DescribeVolumesModifications"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateTags"
      ],
      "Resource": [
        "arn:aws:ec2:*:*:volume/*",
        "arn:aws:ec2:*:*:snapshot/*"
      ],
      "Condition": {
        "StringEquals": {
          "ec2:CreateAction": [
            "CreateVolume",
            "CreateSnapshot"
          ]
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteTags"
      ],
      "Resource": [
        "arn:aws:ec2:*:*:volume/*",
        "arn:aws:ec2:*:*:snapshot/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVolume"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:RequestTag/kubernetes.io/cluster/*": "owned"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteVolume"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "ec2:ResourceTag/kubernetes.io/cluster/*": "owned"
        }
      }
    }
  ]
}
EOF

# Create or update policy
aws iam create-policy --policy-name AmazonEKS_EBS_CSI_Driver_Policy --policy-document file://ebs-csi-policy.json

# Attach policy to IAM role
aws iam attach-role-policy --policy-arn arn:aws:iam::<account-id>:policy/AmazonEKS_EBS_CSI_Driver_Policy --role-name <role-name>
```

#### Azure Disk

```bash
# Create service principal with Contributor role
AKS_CLUSTER_NAME=<aks-cluster-name>
RESOURCE_GROUP=<resource-group>

# Get cluster managed identity
IDENTITY_ID=$(az aks show -g $RESOURCE_GROUP -n $AKS_CLUSTER_NAME --query "identityProfile.kubeletidentity.objectId" -o tsv)

# Assign Contributor role to the managed identity
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
az role assignment create --assignee $IDENTITY_ID --role Contributor --scope /subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP
```

### 3. Fix StorageClass Configuration

#### AWS EBS

```bash
# Create properly configured StorageClass
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  # Specify default zone if necessary
  # csi.storage.k8s.io/availability-zone: us-east-1a
  fsType: ext4
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF
```

#### AWS EFS

```bash
# Create EFS StorageClass
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  fileSystemId: <efs-id>
  provisioningMode: efs-ap
  directoryPerms: "700"
reclaimPolicy: Delete
volumeBindingMode: Immediate
EOF
```

#### Azure Disk

```bash
# Create properly configured StorageClass
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-disk
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
  cachingMode: ReadOnly
  kind: Managed
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF
```

### 4. Force Volume Detachment (use with caution)

#### AWS EBS

```bash
# For volumes stuck in "attaching" or "detaching" state

# Identify volume ID
VOLUME_ID=$(kubectl get pv <pv-name> -o jsonpath='{.spec.awsElasticBlockStore.volumeID}' | cut -d '/' -f 4)

# Force detachment
aws ec2 detach-volume --volume-id $VOLUME_ID --force --region <region>
```

#### Azure Disk

```bash
# For disks stuck in detaching state
DISK_NAME=$(kubectl get pv <pv-name> -o jsonpath='{.spec.azureDisk.diskName}')
RESOURCE_GROUP=$(kubectl get pv <pv-name> -o jsonpath='{.spec.azureDisk.resourceGroup}')

# Get VM ID if attached
VM_ID=$(az disk show --name $DISK_NAME --resource-group $RESOURCE_GROUP --query managedBy -o tsv)

# If disk is attached to a VM, force detach
if [ -n "$VM_ID" ]; then
  VM_NAME=$(echo $VM_ID | cut -d '/' -f 9)
  VM_RG=$(echo $VM_ID | cut -d '/' -f 5)
  az vm disk detach -g $VM_RG --vm-name $VM_NAME --name $DISK_NAME
fi
```

### 5. Handle Cross-AZ Issues

```bash
# Create StorageClass with WaitForFirstConsumer binding mode
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cross-az-aware
provisioner: <csi-driver>
parameters:
  type: <volume-type>
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF
```

### 6. Fix Volume Type Mismatch

```bash
# For AWS, EBS volume types have different performance characteristics
# Create appropriate StorageClass for the workload:

# For high IOPS workloads (databases, etc.)
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: high-iops-storage
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iopsPerGB: "50"
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF

# For general purpose workloads with balanced price-performance
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: general-purpose-storage
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF

# For cost-effective storage with low performance requirements
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: economy-storage
provisioner: ebs.csi.aws.com
parameters:
  type: sc1
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF
```

### 7. Fix Encryption Issues

#### AWS EBS

```bash
# Create StorageClass with encryption enabled
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: encrypted-ebs
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
  # Specify KMS key if needed
  kmsKeyId: <kms-key-id>
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF
```

#### Azure Disk

```bash
# Create StorageClass with encryption enabled
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: encrypted-azure-disk
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
  cachingMode: ReadOnly
  kind: Managed
  fsType: ext4
  # Enable encryption using platform managed key
  # For customer-managed keys, see Azure documentation
  encryption: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF
```

### 8. Request Quota Increase

#### AWS

```bash
# Example AWS support case for quota increase
aws support create-case \
  --subject "EBS Volume Quota Increase Request" \
  --service-code "amazon-elastic-compute-cloud-linux" \
  --category-code "quota-increase" \
  --communication-body "Please increase our EBS volume quota in region <region> from X to Y." \
  --severity-code "normal"
```

#### Azure

```bash
# Request quota increase through Azure portal
# Or use CLI where available
az disk request-quota-increase --disk-type Premium_LRS --region <region> --new-limit 10000
```

### 9. Handle Node Instance Type Limitations

```bash
# Some instance types have limits on the number of volumes that can be attached
# Create node group with larger instance types

# For AWS EKS
aws eks create-nodegroup \
  --cluster-name <cluster-name> \
  --nodegroup-name larger-instance-storage \
  --scaling-config minSize=2,maxSize=5,desiredSize=3 \
  --instance-types m5.xlarge,m5.2xlarge \
  --node-role <node-role-arn> \
  --subnets <subnet-1> <subnet-2> \
  --region <region>

# Deploy critical storage workloads on these nodes using node selectors
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: high-storage-deployment
  namespace: <namespace>
spec:
  replicas: 1
  selector:
    matchLabels:
      app: high-storage-app
  template:
    metadata:
      labels:
        app: high-storage-app
    spec:
      nodeSelector:
        eks.amazonaws.com/nodegroup: larger-instance-storage
      containers:
      - name: high-storage-app
        image: <image>
        volumeMounts:
        - mountPath: /data
          name: data-volume
      volumes:
      - name: data-volume
        persistentVolumeClaim:
          claimName: <pvc-name>
EOF
```

## Prevention

1. **Use WaitForFirstConsumer Binding Mode**: Ensure volume creation in the correct AZ
   ```bash
   # Update default StorageClass
   kubectl patch storageclass <storage-class-name> \
     --type='json' \
     -p='[{"op": "replace", "path": "/volumeBindingMode", "value": "WaitForFirstConsumer"}]'
   ```

2. **Implement Resource Monitoring**: Track cloud storage usage and limits
   ```bash
   # Example Prometheus alert rule for PVC quota
   cat <<EOF > volume-quota-alert.yaml
   apiVersion: monitoring.coreos.com/v1
   kind: PrometheusRule
   metadata:
     name: volume-quota-alert
     namespace: monitoring
   spec:
     groups:
     - name: storage
       rules:
       - alert: PersistentVolumeQuotaNearlyFull
         expr: |
           sum(kube_persistentvolumeclaim_resource_requests_storage_bytes) by (namespace) 
           / 
           sum(kube_resourcequota{resource="requests.storage"}) by (namespace) 
           > 0.85
         for: 10m
         labels:
           severity: warning
         annotations:
           summary: "Storage quota nearly full in namespace {{ \$labels.namespace }}"
           description: "Storage quota is more than 85% utilized in namespace {{ \$labels.namespace }}"
   EOF
   kubectl apply -f volume-quota-alert.yaml
   ```

3. **Implement StorageClass Tiering**: Define different classes for different needs
   ```bash
   # Example storage class documentation
   cat <<EOF > storage-classes.md
   # Storage Classes
   
   ## General Purpose (gp3-storage)
   - Use for: Standard web applications, CI/CD pipelines
   - AWS Type: gp3
   - Azure Type: StandardSSD_LRS
   
   ## High Performance (premium-storage)
   - Use for: Databases, analytics, high throughput
   - AWS Type: io2
   - Azure Type: Premium_LRS
   
   ## Cost Effective (standard-storage)
   - Use for: Backups, archives, batch processing
   - AWS Type: sc1
   - Azure Type: Standard_LRS
   
   ## Shared Storage (efs-storage / azure-files)
   - Use for: ReadWriteMany access mode, shared configs
   - AWS: EFS
   - Azure: Azure Files
   EOF
   ```

4. **Set Up Cross-Region Backup**: Implement automated PV snapshots with cross-region copy
   ```bash
   # For AWS EBS snapshots
   cat <<EOF > ebs-snapshot-cronjob.yaml
   apiVersion: batch/v1
   kind: CronJob
   metadata:
     name: ebs-snapshot-job
     namespace: kube-system
   spec:
     schedule: "0 1 * * *"
     jobTemplate:
       spec:
         template:
           spec:
             serviceAccountName: ebs-snapshot-controller
             containers:
             - name: aws-cli
               image: amazon/aws-cli:latest
               command:
               - /bin/sh
               - -c
               - |
                 #!/bin/bash
                 # Get list of PVs using EBS
                 VOLUMES=$(aws ec2 describe-volumes --filters "Name=tag-key,Values=kubernetes.io/cluster/<cluster-name>" --query "Volumes[*].[VolumeId]" --output text --region <region>)
                 # Create snapshots
                 for vol in $VOLUMES; do
                   aws ec2 create-snapshot --volume-id $vol --description "Automated backup" --tag-specifications "ResourceType=snapshot,Tags=[{Key=Name,Value=auto-backup-$vol}]" --region <region>
                 done
                 # Copy to another region for DR
                 SNAPSHOTS=$(aws ec2 describe-snapshots --owner-ids self --filters "Name=description,Values=Automated backup" --query "Snapshots[?StartTime>='$(date -d '1 day ago' +'%Y-%m-%d')'].SnapshotId" --output text --region <region>)
                 for snap in $SNAPSHOTS; do
                   aws ec2 copy-snapshot --source-region <region> --source-snapshot-id $snap --destination-region <dr-region> --description "DR copy of $snap"
                 done
             restartPolicy: OnFailure
   EOF
   kubectl apply -f ebs-snapshot-cronjob.yaml
   ```

5. **Implement Proper IAM Documentation and Reviews**: Document and review IAM policies
   ```bash
   # Example IAM policy documentation
   cat <<EOF > storage-iam-policy.md
   # Storage IAM Policies
   
   ## EBS CSI Driver Policy
   
   Required permissions:
   - ec2:CreateSnapshot
   - ec2:AttachVolume
   - ec2:DetachVolume
   - ec2:ModifyVolume
   - ec2:DescribeAvailabilityZones
   - ec2:DescribeInstances
   - ec2:DescribeSnapshots
   - ec2:DescribeTags
   - ec2:DescribeVolumes
   - ec2:DescribeVolumesModifications
   - ec2:CreateTags (for volumes and snapshots)
   - ec2:DeleteTags (for volumes and snapshots)
   - ec2:CreateVolume
   - ec2:DeleteVolume
   
   See full policy in IAM console or source control.
   EOF
   ```

6. **Use Pod Topology Constraints**: Ensure pods are scheduled with data locality in mind
   ```bash
   # Example deployment with topology constraints
   kubectl apply -f - <<EOF
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: zone-aware-deployment
     namespace: <namespace>
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: zone-aware-app
     template:
       metadata:
         labels:
           app: zone-aware-app
       spec:
         # Ensure pods are spread across zones
         topologySpreadConstraints:
         - maxSkew: 1
           topologyKey: topology.kubernetes.io/zone
           whenUnsatisfiable: DoNotSchedule
           labelSelector:
             matchLabels:
               app: zone-aware-app
         containers:
         - name: app
           image: <image>
           volumeMounts:
           - mountPath: /data
             name: data-volume
         volumes:
         - name: data-volume
           persistentVolumeClaim:
             claimName: <pvc-name>
   EOF
   ```

7. **Regular CSI Driver Updates**: Keep CSI drivers current
8. **Document Cloud Storage Costs**: Track and document storage expenses
9. **Implement Retention Policies**: Define storage lifecycle and cleanup procedures
10. **Test Failure Scenarios**: Regularly test volume attachment/detachment recovery

## Related Runbooks

* [PVC Stuck in Pending](./pvc-pending.md)
* [Storage Class Issues](./storage-class-issues.md)
* [Dynamic Provisioning Issues](./dynamic-provisioning-issues.md)
* [Storage Performance Issues](./storage-performance-issues.md)
* [Storage Capacity Issues](./storage-capacity-issues.md)
* [CSI Driver Issues](./csi-driver-issues.md)