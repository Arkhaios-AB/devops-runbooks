# Troubleshooting NodePort Service Issues

## Symptoms

* Unable to connect to NodePort service from outside the cluster
* Connection timeouts when accessing NodePort
* Service accessible from some nodes but not others
* Intermittent connectivity to NodePort services
* Service working within cluster but not externally
* "Connection refused" errors when connecting to NodePort
* Specific ports working while others fail
* Inconsistent behavior across different client locations
* NodePort connections drop under load
* Applications report connectivity issues to NodePort services

## Possible Causes

1. **Firewall/Security Group Issues**: External firewalls blocking NodePort range
2. **Service Configuration Problems**: Incorrect service or port configuration
3. **kube-proxy Issues**: kube-proxy not setting up iptables rules correctly
4. **Backend Pod Problems**: Backend pods not ready or running
5. **Node Network Interface Issues**: Problems with node network interfaces or routing
6. **NodePort Range Configuration**: NodePort outside the configured range
7. **Network Policy Restrictions**: Network policies blocking traffic
8. **Load Balancer Configuration**: Load balancer not properly configured for NodePorts
9. **SNAT/IP Masquerading Issues**: Source NAT not working correctly
10. **Cross-Node Routing Problems**: Traffic not properly routed between nodes

## Diagnosis Steps

### 1. Check Service Configuration

```bash
# Check NodePort service configuration
kubectl get service <service-name> -n <namespace>

# Get detailed service information
kubectl describe service <service-name> -n <namespace>

# Confirm the NodePort is assigned
kubectl get service <service-name> -n <namespace> -o jsonpath='{.spec.ports[0].nodePort}'
```

### 2. Check Backend Pods

```bash
# Check if backend pods are running
kubectl get pods -n <namespace> -l <service-selector> -o wide

# Check pod readiness
kubectl describe pods -n <namespace> -l <service-selector>

# Check endpoints
kubectl get endpoints <service-name> -n <namespace>
```

### 3. Test NodePort Connectivity

```bash
# Get node external IPs
kubectl get nodes -o wide

# Test NodePort connection from within the cluster
kubectl run test-client --image=busybox -- sleep 3600
kubectl exec -it test-client -- wget -T 5 -O- http://<node-ip>:<node-port>

# Test connection directly on the node (via SSH)
ssh <node-ip> curl -v localhost:<node-port>
```

### 4. Check Firewall/Security Group

For cloud environments, check security groups/firewall rules:

```bash
# For AWS, check security groups
aws ec2 describe-security-groups --group-id <security-group-id>

# For GCP, check firewall rules
gcloud compute firewall-rules list --filter="network:<network-name>"

# For Azure, check network security groups
az network nsg rule list --resource-group <resource-group> --nsg-name <nsg-name>
```

### 5. Check kube-proxy Status

```bash
# Check kube-proxy pods
kubectl get pods -n kube-system -l k8s-app=kube-proxy

# Check kube-proxy logs
kubectl logs -n kube-system -l k8s-app=kube-proxy

# Check kube-proxy configuration
kubectl get configmap kube-proxy -n kube-system -o yaml
```

### 6. Check iptables Rules on Nodes

```bash
# SSH into a node
ssh <node-ip>

# Check iptables rules for the NodePort
sudo iptables-save | grep <node-port>
```

### 7. Check Network Policies

```bash
# List Network Policies that might affect the service
kubectl get networkpolicy -n <namespace>

# Check Network Policy details
kubectl describe networkpolicy <policy-name> -n <namespace>
```

## Resolution Steps

### 1. Fix Service Configuration

If the service is misconfigured:

```bash
# Edit the service
kubectl edit service <service-name> -n <namespace>
```

Example of proper NodePort configuration:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: <service-name>
spec:
  type: NodePort
  ports:
  - port: 80          # Service port
    targetPort: 8080  # Container port
    nodePort: 30080   # Node port (optional, auto-assigned if not specified)
    protocol: TCP
  selector:
    app: <app-label>  # Must match pod labels
```

Or recreate the service:
```bash
kubectl delete service <service-name> -n <namespace>
kubectl expose deployment <deployment-name> --type=NodePort --port=80 --target-port=8080 --name=<service-name> -n <namespace>
```

### 2. Fix Firewall/Security Group Issues

#### AWS

```bash
# Open NodePort range in security group
aws ec2 authorize-security-group-ingress \
  --group-id <security-group-id> \
  --protocol tcp \
  --port 30000-32767 \
  --cidr 0.0.0.0/0
```

#### GCP

```bash
# Create firewall rule for NodePort range
gcloud compute firewall-rules create allow-nodeports \
  --network <network-name> \
  --allow tcp:30000-32767 \
  --source-ranges 0.0.0.0/0
```

#### Azure

```bash
# Add NSG rule for NodePort range
az network nsg rule create -g <resource-group> --nsg-name <nsg-name> \
  -n allow-nodeports --priority 100 \
  --source-address-prefixes "*" \
  --destination-port-ranges 30000-32767 \
  --access Allow --protocol Tcp
```

### 3. Fix kube-proxy Issues

If kube-proxy is not working correctly:

```bash
# Restart kube-proxy pods
kubectl rollout restart daemonset kube-proxy -n kube-system

# If using specific mode (e.g., ipvs), check config
kubectl edit configmap kube-proxy -n kube-system
```

Example configuration adjustment:
```yaml
mode: "iptables"  # Or "ipvs" depending on your setup
```

### 4. Fix Backend Pod Issues

If backend pods are not ready:

```bash
# Restart deployments
kubectl rollout restart deployment <deployment-name> -n <namespace>

# Check pod logs for errors
kubectl logs -n <namespace> -l <service-selector>

# Ensure pods have correct readiness probes
kubectl edit deployment <deployment-name> -n <namespace>
```

Example of proper readiness probe:
```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

### 5. Fix Specific NodePort Issues

If only specific NodePorts are problematic:

```bash
# Recreate service with a different NodePort
kubectl delete service <service-name> -n <namespace>
kubectl expose deployment <deployment-name> --type=NodePort --port=80 --target-port=8080 --name=<service-name> -n <namespace>

# Or specify a NodePort explicitly
kubectl patch service <service-name> -n <namespace> --type='json' \
  -p='[{"op": "replace", "path": "/spec/ports/0/nodePort", "value": 30090}]'
```

### 6. Fix Node Networking Issues

If nodes have networking issues:

```bash
# SSH to the node
ssh <node-ip>

# Check node network interfaces
ip addr
ip route

# Ensure node can route traffic
sudo sysctl net.ipv4.ip_forward
# Should return net.ipv4.ip_forward = 1

# If not, enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1
sudo echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
```

### 7. Fix Network Policy Issues

If NetworkPolicies are blocking traffic:

```bash
# Create policy allowing ingress to service
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-nodeport-ingress
  namespace: <namespace>
spec:
  podSelector:
    matchLabels:
      app: <app-label>
  ingress:
  - from: []  # Allow from anywhere
    ports:
    - protocol: TCP
      port: <container-port>
EOF
```

### 8. Configure External Load Balancer for NodePorts

To provide a stable entry point for NodePort services:

#### AWS Classic Load Balancer

```bash
# Create ELB pointing to all nodes on the NodePort
aws elb create-load-balancer \
  --load-balancer-name <service-name>-lb \
  --listeners "Protocol=TCP,LoadBalancerPort=80,InstanceProtocol=TCP,InstancePort=<node-port>" \
  --availability-zones <availability-zones> \
  --security-groups <security-group-id>

# Add nodes to load balancer
aws elb register-instances-with-load-balancer \
  --load-balancer-name <service-name>-lb \
  --instances <instance-id-1> <instance-id-2>
```

#### GCP Load Balancer

```bash
# Create target pool for nodes
gcloud compute target-pools create <service-name>-pool --region <region>

# Add nodes to pool
gcloud compute target-pools add-instances <service-name>-pool \
  --instances <instance-1>,<instance-2> --zone <zone>

# Create forwarding rule to nodes
gcloud compute forwarding-rules create <service-name>-rule \
  --region <region> --target-pool <service-name>-pool \
  --ports <node-port>
```

### 9. Fix Source NAT Issues

If SNAT/Masquerading is problematic:

```bash
# Check if IP masquerade is enabled on node
ssh <node-ip> sudo iptables -t nat -L | grep MASQUERADE

# If missing, check kube-proxy configuration
kubectl edit configmap kube-proxy -n kube-system
```

Ensure the following section exists:
```yaml
iptables:
  masqueradeAll: true
```

### 10. Configure Source IP Preservation

If you need to preserve client source IPs:

```bash
# Update service with externalTrafficPolicy: Local
kubectl patch service <service-name> -n <namespace> \
  -p '{"spec":{"externalTrafficPolicy":"Local"}}'
```

Note: This will ensure traffic only goes to pods on the same node, which may affect load balancing.

## Prevention

1. **Firewall Documentation**: Document required NodePort ranges in firewall rules
2. **Service Definition Standards**: Establish standards for service definitions
3. **Health Checks**: Implement proper health checks for all services
4. **NodePort Range Planning**: Plan and document NodePort range usage
5. **Network Policy Design**: Design NetworkPolicies with NodePort access in mind
6. **Regular Testing**: Test NodePort accessibility regularly
7. **Monitoring**: Monitor NodePort service availability and performance
8. **Load Balancer Integration**: Use load balancers in front of NodePorts for production
9. **IP Tables Management**: Document and manage custom iptables rules carefully
10. **External Access Documentation**: Document external access methods for services

## Related Runbooks

* [Service Not Accessible](./service-not-accessible.md)
* [LoadBalancer Service Issues](./loadbalancer-issues.md)
* [Network Policy Issues](./network-policy-issues.md)
* [Pod-to-Pod Communication Issues](./pod-to-pod-communication.md)
* [kube-proxy Issues](../cluster/kube-proxy-issues.md)
* [Node Not Ready](../cluster/node-not-ready.md)