# Troubleshooting Cluster Autoscaling Issues

## Symptoms

* Pending pods not triggering new node creation
* Cluster doesn't scale down despite low utilization
* Nodes added too slowly to meet demand
* Unexpected node terminations
* Scale-up happens but pods remain in Pending state
* Cluster Autoscaler logs showing errors or warnings
* Autoscaling events not appearing in cluster events

## Possible Causes

1. **Misconfigured Autoscaler**: Incorrect settings for the Cluster Autoscaler
2. **Resource Requests Mismatch**: Pod resource requests not set properly
3. **Node Group Configuration**: Issues with node group settings
4. **Cloud Provider Limits**: Hitting cloud provider quotas or limits
5. **Pod Disruption Budgets**: PDBs preventing scale-down
6. **Node Selectors or Taints**: Pods requiring specific nodes that cannot be created
7. **Network Issues**: Network problems preventing node registration
8. **Permission Issues**: Insufficient permissions for autoscaler
9. **Instance Type Unavailability**: Requested instance types not available
10. **Scale Down Safety Mechanisms**: Scale-down safety checks preventing node removal

## Diagnosis Steps

### 1. Check Cluster Autoscaler Deployment

```bash
# Check if Cluster Autoscaler is running
kubectl get pods -n kube-system | grep cluster-autoscaler

# Check Cluster Autoscaler logs
kubectl logs -n kube-system $(kubectl get pods -n kube-system | grep cluster-autoscaler | awk '{print $1}')
```

### 2. Check Cluster Autoscaler Configuration

```bash
# Examine Cluster Autoscaler deployment
kubectl describe deployment cluster-autoscaler -n kube-system

# Check Cluster Autoscaler configuration
kubectl get configmap cluster-autoscaler-status -n kube-system -o yaml
```

### 3. Check Cloud Provider Status

For AWS:
```bash
# Check if you're hitting EC2 limits
aws ec2 describe-account-attributes
```

For GCP:
```bash
# Check Compute Engine quota
gcloud compute project-info describe
```

For Azure:
```bash
# Check VM quota
az vm list-usage --location <location>
```

### 4. Check Pending Pods and Their Requirements

```bash
# List pending pods
kubectl get pods --all-namespaces | grep Pending

# Examine a specific pending pod
kubectl describe pod <pod-name> -n <namespace>
```

### 5. Check Node Groups and Scaling Events

```bash
# Get nodes and their groups
kubectl get nodes --show-labels

# Get cluster events related to scaling
kubectl get events | grep -i "scale"
```

## Resolution Steps

### 1. Fix Cluster Autoscaler Configuration

Update the Cluster Autoscaler deployment with proper settings:

```bash
kubectl edit deployment cluster-autoscaler -n kube-system
```

Important parameters to check:
```yaml
- --max-nodes-total=100
- --scale-down-utilization-threshold=0.5
- --scale-down-unneeded-time=10m
- --max-graceful-termination-sec=600
- --expander=least-waste
```

### 2. Fix Resource Requests

Ensure pods have appropriate resource requests:

```bash
# Edit deployment to set resource requests
kubectl edit deployment <deployment-name>
```

Example resource configuration:
```yaml
resources:
  requests:
    cpu: "100m"
    memory: "256Mi"
```

### 3. Fix Node Group Configuration

For AWS:
```bash
# Update ASG configuration
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name <asg-name> \
  --min-size 1 \
  --max-size 10
```

For GCP:
```bash
# Update managed instance group
gcloud compute instance-groups managed resize <group-name> --size <size>
```

For Azure:
```bash
# Update VMSS
az vmss update --resource-group <resource-group> --name <vmss-name> --set sku.capacity=<capacity>
```

### 4. Address Cloud Provider Limits

Request quota increases from your cloud provider:
- AWS: Request service limit increases via AWS Support
- GCP: Request quota increases via Google Cloud Console
- Azure: Request quota increases via Azure Portal

### 5. Fix Pod Disruption Budgets

Adjust PDBs that might be preventing scale-down:

```bash
# List PDBs
kubectl get poddisruptionbudgets --all-namespaces

# Edit a problematic PDB
kubectl edit poddisruptionbudget <pdb-name> -n <namespace>
```

Example of a more flexible PDB:
```yaml
spec:
  maxUnavailable: 1  # or use minAvailable instead
```

### 6. Fix Node Selectors and Taints

Review pod nodeSelector and tolerations to ensure they're not too restrictive:

```bash
# Edit deployment to adjust node selectors
kubectl edit deployment <deployment-name>
```

Example of more flexible node selection:
```yaml
nodeSelector:
  kubernetes.io/os: linux  # Less restrictive than specific instance types
```

### 7. Restart Cluster Autoscaler

If the autoscaler seems stuck:
```bash
kubectl rollout restart deployment cluster-autoscaler -n kube-system
```

### 8. Fix Permissions

Ensure the autoscaler has proper permissions to scale the cluster:

For AWS, check IAM roles and policies:
```bash
# Example IAM policy needed for AWS autoscaler
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup",
        "ec2:DescribeLaunchTemplateVersions"
      ],
      "Resource": "*"
    }
  ]
}
```

### 9. Consider Instance Diversity

If specific instance types are unavailable, use more instance type options:

For AWS:
```bash
# Update mixed instance policy in ASG
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name <asg-name> \
  --mixed-instances-policy '{
    "InstancesDistribution": {
      "OnDemandPercentageAboveBaseCapacity": 0,
      "SpotAllocationStrategy": "capacity-optimized"
    },
    "LaunchTemplate": {
      "LaunchTemplateSpecification": {
        "LaunchTemplateId": "<template-id>",
        "Version": "$Latest"
      },
      "Overrides": [
        {"InstanceType": "m5.large"},
        {"InstanceType": "m5a.large"},
        {"InstanceType": "m5d.large"},
        {"InstanceType": "m5ad.large"}
      ]
    }
  }'
```

## Prevention

1. **Regular Testing**: Test autoscaling regularly with load testing
2. **Monitoring**: Set up monitoring for autoscaling events and node provisioning times
3. **Resource Planning**: Set appropriate resource requests for all workloads
4. **Use Multiple Instance Types**: Configure node groups with multiple instance types
5. **Request Higher Quotas**: Request higher quotas from cloud providers before reaching limits
6. **Documentation**: Document autoscaling configuration and expected behavior
7. **Scale-Down Configuration**: Carefully configure scale-down parameters based on workload
8. **Reserved Instances**: Consider reserved instances for baseline capacity
9. **Overprovisioning**: Slightly overprovision to handle sudden load increases

## Related Runbooks

* [Node Not Ready](./node-not-ready.md)
* [Pod Stuck in Pending](../workloads/pod-pending.md)
* [HPA Issues](../resources/hpa-issues.md)
* [Resource Quota Issues](../resources/resource-quota-issues.md)
* [Deployment Rollout Issues](../workloads/deployment-rollout-issues.md)