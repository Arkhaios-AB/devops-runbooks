# Troubleshooting Dynamic Provisioning Issues

## Symptoms

* Persistent Volume Claims (PVCs) stuck in Pending state
* Error messages like "failed to provision volume"
* StorageClass not creating volumes as expected
* Long delays in PVC binding
* CSI driver errors in logs
* Volume provisioner pod errors
* Cloud provider volume provisioning failures
* Volumes created but not mounted correctly
* Inconsistent volume creation behavior
* Error events in PVC events about provisioning
* Failed volume attachments after provisioning

## Possible Causes

1. **StorageClass Configuration Issues**: Misconfigured or missing StorageClass
2. **CSI Driver Problems**: CSI driver not installed or malfunctioning
3. **Cloud Provider Limitations**: Hitting volume limits or quota issues
4. **Permissions Problems**: Insufficient permissions to provision volumes
5. **Zone/Region Mismatch**: Attempting to provision in unavailable zones
6. **Resource Constraints**: Insufficient resources for volume creation
7. **Storage Backend Issues**: Problems with underlying storage infrastructure
8. **API Rate Limiting**: Cloud provider API throttling
9. **Volume Parameters Mismatch**: Incompatible parameters in StorageClass
10. **Plugin Configuration Issues**: Misconfigured volume plugins

## Diagnosis Steps

### 1. Check PVC Status and Events

```bash
# Get details about the PVC
kubectl get pvc <pvc-name> -n <namespace>

# Describe the PVC to see events
kubectl describe pvc <pvc-name> -n <namespace>

# Look for provisioning errors in events
kubectl get events -n <namespace> | grep <pvc-name>
```

### 2. Check StorageClass Configuration

```bash
# List available StorageClasses
kubectl get storageclass

# Get details of StorageClass
kubectl describe storageclass <storageclass-name>

# Check if StorageClass parameters are correct
kubectl get storageclass <storageclass-name> -o yaml
```

### 3. Check Volume Provisioner Status

```bash
# Check CSI controller and node pods
kubectl get pods -n kube-system -l app=csi-provisioner

# For specific CSI drivers, check their namespaces
# For example, AWS EBS CSI driver:
kubectl get pods -n kube-system -l app=ebs-csi-controller

# For GCP PD CSI driver:
kubectl get pods -n kube-system -l app=gcp-compute-persistent-disk-csi-driver

# For Azure Disk CSI driver:
kubectl get pods -n kube-system -l app=csi-azuredisk-controller
```

### 4. Check CSI Driver Logs

```bash
# Get logs from provisioner
kubectl logs -n kube-system -l app=csi-provisioner -c csi-provisioner

# For specific CSI driver controller logs:
kubectl logs -n kube-system <csi-controller-pod-name> -c <container-name>
```

### 5. Check Cloud Provider Status and Quotas

#### AWS

```bash
# Check EC2 volume limits and usage
aws ec2 describe-volume-status --region <region>

# Check service quotas
aws service-quotas get-service-quota --service-code ec2 --quota-code L-D18FCD1D --region <region>
```

#### GCP

```bash
# Check disk quotas
gcloud compute project-info describe --project <project-id>

# Check current disk usage
gcloud compute disks list --project <project-id>
```

#### Azure

```bash
# Check disk quotas
az vm list-usage --location <location> | grep Disks

# List disks
az disk list --output table
```

### 6. Check for Authentication and Authorization Issues

```bash
# Check if service account has correct permissions
kubectl describe serviceaccount <service-account> -n <namespace>

# Check if cluster roles and role bindings exist for CSI drivers
kubectl get clusterrole,clusterrolebinding -l app=csi-provisioner
```

### 7. Check Node Conditions

```bash
# Check if nodes have any conditions affecting volume operations
kubectl get nodes -o wide

# Describe specific node
kubectl describe node <node-name>
```

## Resolution Steps

### 1. Fix StorageClass Configuration

If StorageClass is misconfigured:

```bash
# Create new StorageClass with correct parameters
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: <storageclass-name>
provisioner: <provisioner>
parameters:
  type: <volume-type>
  fsType: <filesystem-type>
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF
```

Examples for different cloud providers:

#### AWS EBS

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: ext4
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

#### GCP Persistent Disk

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: pd-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  fstype: ext4
  replication-type: none
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

#### Azure Disk

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-premium
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
  cachingMode: ReadOnly
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### 2. Fix CSI Driver Issues

If CSI driver is not working correctly:

```bash
# Restart CSI controller
kubectl rollout restart deployment <csi-controller-deployment> -n kube-system

# For specific CSI drivers:
# AWS EBS CSI driver
kubectl rollout restart deployment ebs-csi-controller -n kube-system

# GCP PD CSI driver
kubectl rollout restart deployment gcp-pd-csi-controller -n kube-system

# Azure Disk CSI driver
kubectl rollout restart deployment csi-azuredisk-controller -n kube-system
```

If CSI driver is not installed or needs to be updated:

#### AWS EBS CSI Driver

```bash
# Using Helm
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update
helm upgrade --install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.create=true \
  --set controller.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::<account-id>:role/<role-name>
```

#### GCP PD CSI Driver

```bash
# Using manifest
kubectl apply -k "github.com/kubernetes-sigs/gcp-compute-persistent-disk-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"
```

#### Azure Disk CSI Driver

```bash
# Using Helm
helm repo add azuredisk-csi-driver https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/master/charts
helm repo update
helm upgrade --install azuredisk-csi-driver azuredisk-csi-driver/azuredisk-csi-driver \
  --namespace kube-system \
  --set cloud=AzurePublicCloud
```

### 3. Fix Cloud Provider Permissions

#### AWS

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

# Create the policy
aws iam create-policy --policy-name AmazonEKS_EBS_CSI_Driver_Policy --policy-document file://ebs-csi-policy.json
```

#### GCP

```bash
# Create service account with correct permissions
gcloud iam service-accounts create gcp-pd-csi-driver

# Assign required roles
gcloud projects add-iam-policy-binding <project-id> \
  --member serviceAccount:gcp-pd-csi-driver@<project-id>.iam.gserviceaccount.com \
  --role roles/compute.storageAdmin

# Create key
gcloud iam service-accounts keys create gcp-pd-csi-driver-key.json \
  --iam-account gcp-pd-csi-driver@<project-id>.iam.gserviceaccount.com

# Create Kubernetes secret
kubectl create secret generic gcp-pd-csi-driver-creds \
  --from-file=key.json=gcp-pd-csi-driver-key.json \
  --namespace kube-system
```

#### Azure

```bash
# Create service principal
SP_PASSWORD=$(az ad sp create-for-rbac --name AzureDiskCSIDriver \
  --query password --output tsv)
SP_APP_ID=$(az ad sp list --display-name AzureDiskCSIDriver \
  --query [].appId --output tsv)

# Assign Contributor role
az role assignment create --assignee $SP_APP_ID --role Contributor

# Create Kubernetes secret
kubectl create secret generic azure-disk-csi-driver-creds \
  --from-literal=client-id=$SP_APP_ID \
  --from-literal=client-secret=$SP_PASSWORD \
  --namespace kube-system
```

### 4. Fix Volume Binding Mode

If volumes are pending due to scheduling issues:

```bash
# Edit StorageClass to use WaitForFirstConsumer binding mode
kubectl patch storageclass <storageclass-name> \
  --type='json' \
  -p='[{"op": "replace", "path": "/volumeBindingMode", "value": "WaitForFirstConsumer"}]'
```

### 5. Fix Zone/Region Constraints

Specify allowed topologies in StorageClass:

```bash
# For AWS
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc-zonal
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: ext4
allowedTopologies:
- matchLabelExpressions:
  - key: topology.kubernetes.io/zone
    values:
    - us-east-1a
    - us-east-1b
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
EOF

# For GCP
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: pd-ssd-zonal
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
allowedTopologies:
- matchLabelExpressions:
  - key: topology.kubernetes.io/zone
    values:
    - us-central1-a
    - us-central1-b
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
EOF

# For Azure
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-premium-zonal
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
allowedTopologies:
- matchLabelExpressions:
  - key: topology.kubernetes.io/zone
    values:
    - eastus-1
    - eastus-2
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
EOF
```

### 6. Request Quota Increase

If hitting cloud provider limits:

#### AWS

```bash
# Request quota increase through AWS console or support ticket
aws support create-case --subject "EBS Volume Quota Increase" \
  --service-code amazon-elastic-compute-cloud-linux \
  --category-code general-guidance \
  --communication-body "Please increase EBS volume quota for account in region <region>"
```

#### GCP

```bash
# Request quota increase through GCP console or API
gcloud compute project-info update --project <project-id> \
  --quota "DISKS_TOTAL_GB:1000"
```

#### Azure

```bash
# Request quota increase through Azure portal or support ticket
az support ticket create --title "Disk Quota Increase" \
  --description "Please increase managed disk quota in region <region>" \
  --problem-classification "Quota or Subscription Management"
```

### 7. Recreate PVC with Correct Parameters

If a PVC is stuck due to parameter issues:

```bash
# Delete the problematic PVC (warning: data will be lost if already bound)
kubectl delete pvc <pvc-name> -n <namespace>

# Create a new PVC with correct parameters
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: <pvc-name>
  namespace: <namespace>
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: <storageclass-name>
  resources:
    requests:
      storage: 10Gi
EOF
```

## Prevention

1. **Test StorageClass Before Production**: Verify StorageClass works with test PVCs
   ```bash
   # Test PVC
   kubectl apply -f - <<EOF
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: test-pvc
   spec:
     accessModes:
       - ReadWriteOnce
     storageClassName: <storageclass-name>
     resources:
       requests:
         storage: 1Gi
   EOF
   
   # Test pod with PVC
   kubectl apply -f - <<EOF
   apiVersion: v1
   kind: Pod
   metadata:
     name: test-storage-pod
   spec:
     containers:
     - name: test-container
       image: busybox
       command: ["sh", "-c", "echo testing dynamic provisioning > /data/test.txt && sleep 3600"]
       volumeMounts:
       - mountPath: /data
         name: test-volume
     volumes:
     - name: test-volume
       persistentVolumeClaim:
         claimName: test-pvc
   EOF
   ```

2. **Implement Monitoring for PVC Creation**:
   ```bash
   # Prometheus alert rule for PVC pending too long
   cat <<EOF > pvc-pending-alert.yaml
   apiVersion: monitoring.coreos.com/v1
   kind: PrometheusRule
   metadata:
     name: pvc-pending-alert
     namespace: monitoring
   spec:
     groups:
     - name: storage
       rules:
       - alert: PersistentVolumeClaimPending
         expr: kube_persistentvolumeclaim_status_phase{phase="Pending"} == 1
         for: 15m
         labels:
           severity: warning
         annotations:
           summary: "PVC {{ \$labels.namespace }}/{{ \$labels.persistentvolumeclaim }} pending for more than 15 minutes"
           description: "PVC has been in pending state too long, check provisioner logs"
   EOF
   kubectl apply -f pvc-pending-alert.yaml
   ```

3. **Set Default StorageClass**:
   ```bash
   # Mark a StorageClass as default
   kubectl patch storageclass <storageclass-name> \
     -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
   ```

4. **Implement Proper RBAC for Storage Operations**:
   ```bash
   # Create storage admin role
   kubectl apply -f - <<EOF
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRole
   metadata:
     name: storage-admin
   rules:
   - apiGroups: [""]
     resources: ["persistentvolumes", "persistentvolumeclaims"]
     verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
   - apiGroups: ["storage.k8s.io"]
     resources: ["storageclasses"]
     verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
   EOF
   
   # Bind role to user or group
   kubectl apply -f - <<EOF
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: storage-admin-binding
   subjects:
   - kind: User
     name: <username>
     apiGroup: rbac.authorization.k8s.io
   roleRef:
     kind: ClusterRole
     name: storage-admin
     apiGroup: rbac.authorization.k8s.io
   EOF
   ```

5. **Use Resource Quotas for Storage**:
   ```bash
   # Set storage quotas for namespace
   kubectl apply -f - <<EOF
   apiVersion: v1
   kind: ResourceQuota
   metadata:
     name: storage-quota
     namespace: <namespace>
   spec:
     hard:
       persistentvolumeclaims: "10"
       requests.storage: "500Gi"
   EOF
   ```

6. **Use Volume Expansion Instead of New Volumes**:
   ```bash
   # Make StorageClass support volume expansion
   kubectl patch storageclass <storageclass-name> \
     --type='json' \
     -p='[{"op": "replace", "path": "/allowVolumeExpansion", "value": true}]'
   
   # Example of expanding existing PVC
   kubectl patch pvc <pvc-name> -n <namespace> \
     --type='json' \
     -p='[{"op": "replace", "path": "/spec/resources/requests/storage", "value": "20Gi"}]'
   ```

7. **Create Storage Documentation**: Document StorageClass parameters and limitations
8. **Regularly Update CSI Drivers**: Keep CSI drivers up to date
9. **Automate Storage Validation**: Create scripts to validate storage provisioning
10. **Cross-Zone Volume Planning**: Design applications to handle zone failures

## Related Runbooks

* [PVC Stuck in Pending](./pvc-pending.md)
* [Storage Class Issues](./storage-class-issues.md)
* [CSI Driver Issues](./csi-driver-issues.md)
* [Volume Mount Problems](./volume-mount-problems.md)
* [StatefulSet Storage Issues](./statefulset-storage-issues.md)
* [Storage Capacity Issues](./storage-capacity-issues.md)