# Troubleshooting LoadBalancer Service Issues

## Symptoms

* External IP stays in `<pending>` state
* Unable to connect to service from outside the cluster
* Intermittent connectivity to LoadBalancer service
* Load balancer health checks failing
* Error messages in service events
* Load balancer not distributing traffic evenly
* SSL/TLS handshake failures for HTTPS services
* High latency when accessing services
* Unexpected connection terminations
* Cloud provider errors related to load balancer provisioning

## Possible Causes

1. **Cloud Provider Issues**: Problems with cloud provider load balancer integration
2. **Missing Permissions**: Insufficient permissions to create load balancers
3. **Quota Limits**: Reaching cloud provider quota for load balancers
4. **Network Configuration**: VPC/subnet/security group misconfiguration
5. **Service Configuration**: Misconfigured LoadBalancer service
6. **Health Check Failures**: Pods failing load balancer health checks
7. **Backend Not Ready**: Backend pods not in ready state
8. **Port Configuration**: Incorrect port mapping
9. **Cloud Controller Issues**: Cloud controller manager not functioning properly
10. **In-tree vs. Out-of-tree Provider**: Using deprecated in-tree provider

## Diagnosis Steps

### 1. Check Service Status

```bash
# Check LoadBalancer service status
kubectl get service <service-name> -n <namespace>

# Get service details
kubectl describe service <service-name> -n <namespace>

# Check service events
kubectl get events -n <namespace> | grep <service-name>
```

### 2. Check Backend Pods

```bash
# Get pods matching service selector
kubectl get pods -n <namespace> -l <service-selector> -o wide

# Check if pods are ready
kubectl get pods -n <namespace> -l <service-selector> | grep -v "Running"

# Check pod endpoints
kubectl get endpoints <service-name> -n <namespace>
```

### 3. Check Cloud Controller Manager

```bash
# Check cloud controller manager status
kubectl get pods -n kube-system -l k8s-app=cloud-controller-manager

# Check cloud controller logs
kubectl logs -n kube-system -l k8s-app=cloud-controller-manager
```

### 4. Check Cloud Provider Resources

For AWS:
```bash
# Check ELB/NLB in AWS console or CLI
aws elb describe-load-balancers | grep <cluster-name>
aws elbv2 describe-load-balancers | grep <cluster-name>
```

For GCP:
```bash
# Check load balancers in GCP
gcloud compute forwarding-rules list | grep <cluster-name>
gcloud compute target-pools list | grep <cluster-name>
```

For Azure:
```bash
# Check load balancers in Azure
az network lb list -o table | grep <cluster-name>
```

### 5. Test Connectivity

```bash
# Get external IP
EXTERNAL_IP=$(kubectl get service <service-name> -n <namespace> -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Test connectivity
curl -v http://$EXTERNAL_IP:<service-port>
```

### 6. Check NetworkPolicy and Firewalls

```bash
# Check if NetworkPolicies could be blocking traffic
kubectl get networkpolicy -n <namespace>

# Check cloud provider firewall rules (platform-specific)
```

## Resolution Steps

### 1. Fix Pending External IP

If external IP remains pending:

#### AWS

```bash
# Check AWS cloud controller manager logs
kubectl logs -n kube-system -l k8s-app=aws-cloud-controller-manager

# Verify IAM permissions (needs ELB/EC2 permissions)
# Check subnet tags (require kubernetes.io/cluster/<cluster-name>=shared|owned)
# Check security groups
```

#### GCP

```bash
# Check GKE cloud controller logs
kubectl logs -n kube-system -l k8s-app=gcp-controller-manager

# Verify IAM permissions
# Check network tags and firewall rules
```

#### Azure

```bash
# Check Azure cloud controller logs
kubectl logs -n kube-system -l component=cloud-controller-manager

# Verify Azure RBAC permissions
# Check network security groups
```

### 2. Fix Service Configuration

If service is misconfigured:

```bash
# Edit the service
kubectl edit service <service-name> -n <namespace>
```

Example of proper LoadBalancer configuration:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: <service-name>
  annotations:
    # Cloud-specific annotations (examples below)
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"  # AWS NLB
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "<cert-arn>"  # AWS TLS
    cloud.google.com/neg: '{"ingress": true}'  # GCP NEG
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"  # Azure internal LB
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local  # Preserves client source IP
  ports:
  - port: 80  # Port exposed on load balancer
    targetPort: 8080  # Port on pod
    protocol: TCP
  selector:
    app: <app-label>  # Must match pod labels
```

### 3. Fix Health Check Issues

If health checks are failing:

```bash
# Configure readiness probe for proper health checks
kubectl edit deployment <deployment-name> -n <namespace>
```

Example of good readiness probe:
```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10
  failureThreshold: 3
```

For cloud-specific health check configuration:
```yaml
metadata:
  annotations:
    # AWS
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: "/health"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-port: "8080"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "30"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-timeout: "5"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-healthy-threshold: "2"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-unhealthy-threshold: "2"
    
    # GCP
    cloud.google.com/load-balancer-type: "Internal"
    cloud.google.com/health-check-path: "/health"
    cloud.google.com/health-check-port: "8080"
```

### 4. Fix Backend Service Issues

If backend pods aren't ready:

```bash
# Scale up the deployment if needed
kubectl scale deployment <deployment-name> --replicas=3 -n <namespace>

# Check pod logs for issues
kubectl logs -n <namespace> -l <service-selector>

# Restart deployment if needed
kubectl rollout restart deployment <deployment-name> -n <namespace>
```

### 5. Fix Port Configuration

If ports are misconfigured:

```bash
# Update the service with correct ports
kubectl patch service <service-name> -n <namespace> --type=json \
  -p='[{"op": "replace", "path": "/spec/ports", "value":[{"port": 80, "targetPort": 8080, "protocol": "TCP"}]}]'
```

### 6. Fix SSL/TLS Configuration

For HTTPS services:

#### AWS

```bash
# Create or update a service with SSL configuration
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: <service-name>
  namespace: <namespace>
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:<region>:<account-id>:certificate/<cert-id>"
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"
spec:
  type: LoadBalancer
  ports:
  - port: 443
    targetPort: 8080
    protocol: TCP
  selector:
    app: <app-label>
EOF
```

#### GCP

```bash
# For GCP, typically use Ingress for SSL termination
# Or set up a managed certificate with Ingress
```

### 7. Fix Cloud Provider Permissions

For AWS:
```bash
# Ensure proper IAM permissions
# Attach the following policies to node IAM role:
# - AmazonEKSClusterPolicy
# - AmazonEKSVPCResourceController
# - AmazonEKSWorkerNodePolicy
# - AmazonEC2ContainerRegistryReadOnly
# - Custom policy with ELB permissions
```

For GCP:
```bash
# Ensure the following roles:
# - Kubernetes Engine Admin
# - Compute Network Admin
# - Compute Security Admin
```

For Azure:
```bash
# Ensure the following roles:
# - Contributor role on the resource group
# - Network Contributor role
```

### 8. Fix Internal LoadBalancer Issues

For internal load balancers:

```bash
# Configure internal load balancer
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: <service-name>
  namespace: <namespace>
  annotations:
    # AWS
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
    # GCP
    cloud.google.com/load-balancer-type: "Internal"
    # Azure
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  selector:
    app: <app-label>
EOF
```

### 9. Fix Source IP Preservation

To preserve client source IPs:

```bash
# Configure externalTrafficPolicy to Local
kubectl patch service <service-name> -n <namespace> \
  -p '{"spec":{"externalTrafficPolicy":"Local"}}'
```

### 10. Fix Cloud Controller Manager Issues

If the cloud controller manager is having problems:

```bash
# Restart cloud controller manager pods
kubectl rollout restart deployment cloud-controller-manager -n kube-system
# Or for a daemonset:
kubectl rollout restart daemonset cloud-controller-manager -n kube-system
```

## Prevention

1. **IAM Permissions**: Ensure proper IAM/RBAC permissions for load balancer creation
2. **Subnet Configuration**: Correctly tag and configure subnets
3. **Quota Monitoring**: Monitor cloud provider quotas and request increases if needed
4. **Health Checks**: Implement proper health checks for all services
5. **Documentation**: Document load balancer configuration and requirements
6. **Network Policies**: Design NetworkPolicies to allow load balancer traffic
7. **Security Groups**: Configure security groups to allow health checks and traffic
8. **Monitoring**: Monitor load balancer health and performance
9. **Testing**: Test load balancer configurations in non-production first
10. **Use Ingress**: For HTTP/HTTPS services, consider using Ingress instead of direct LoadBalancer

## Related Runbooks

* [Service Not Accessible](./service-not-accessible.md)
* [NodePort Service Issues](./nodeport-issues.md)
* [Ingress Issues](./ingress-issues.md)
* [Network Policy Issues](./network-policy-issues.md)
* [Pod-to-Pod Communication Issues](./pod-to-pod-communication.md)
* [Certificate Issues](../cluster/certificate-issues.md)