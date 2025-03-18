# Troubleshooting Pod-to-Pod Communication Issues

## Symptoms

* Pods unable to connect to other pods
* Connection timeouts between pods
* Intermittent connectivity issues
* Applications reporting connection refused errors
* Some pod connections work while others fail
* Service discovery works but direct pod IP connections fail
* Slow or high-latency connections between pods
* TCP connections work but UDP fails (or vice versa)
* Connection issues after node failure or pod rescheduling
* `ping` works but application-level connections fail

## Possible Causes

1. **CNI Plugin Issues**: Problems with the Container Network Interface
2. **Network Policy Restrictions**: Network policies blocking traffic
3. **Overlay Network Issues**: Problems with the overlay network (Flannel, Calico, etc.)
4. **Pod CIDR Conflicts**: IP range conflicts between nodes or with external networks
5. **MTU Mismatches**: Inconsistent MTU settings causing packet fragmentation
6. **Routing Problems**: Incorrect routes between nodes
7. **Security Group/Firewall Issues**: Cloud provider security groups blocking traffic
8. **DNS Resolution Problems**: Unable to resolve pod or service hostnames
9. **Service Mesh Issues**: Service mesh (Istio, Linkerd, etc.) configuration problems
10. **Cross-Namespace Communication**: Issues with communication across namespaces

## Diagnosis Steps

### 1. Check Pod Network Status

```bash
# Get pod IPs and nodes they're running on
kubectl get pods -o wide -n <namespace>

# Create a test pod for connectivity checks
kubectl run netshoot --image=nicolaka/netshoot -- sleep 3600
```

### 2. Test Connectivity Between Pods

```bash
# Test ping connectivity
kubectl exec -it netshoot -- ping <target-pod-ip>

# Test TCP connectivity with netcat
kubectl exec -it netshoot -- nc -zv <target-pod-ip> <port>

# Test UDP connectivity
kubectl exec -it netshoot -- nc -zuv <target-pod-ip> <port>

# Trace route to another pod
kubectl exec -it netshoot -- traceroute <target-pod-ip>
```

### 3. Check CNI Plugin Status

```bash
# Check CNI plugin pods
kubectl get pods -n kube-system -l k8s-app=calico-node  # For Calico
kubectl get pods -n kube-system -l k8s-app=cilium       # For Cilium
kubectl get pods -n kube-system -l name=weave-net       # For Weave
kubectl get pods -n kube-system -l app=flannel          # For Flannel

# Check CNI logs
kubectl logs -n kube-system <cni-pod-name>
```

### 4. Check Network Policies

```bash
# List Network Policies
kubectl get networkpolicy --all-namespaces

# Examine specific Network Policy
kubectl describe networkpolicy <policy-name> -n <namespace>
```

### 5. Check MTU Settings

```bash
# Check MTU on the test pod
kubectl exec -it netshoot -- ip link show | grep mtu

# Check MTU on the node (via SSH)
ssh <node-ip> ip link show | grep mtu
```

### 6. Check Node Routing Tables

```bash
# SSH to node and check routes
ssh <node-ip> ip route
ssh <node-ip> ip rule list

# Check if Pod CIDR routes exist
ssh <node-ip> ip route | grep <pod-cidr>
```

### 7. Check Service Mesh if Applicable

```bash
# For Istio, check proxy status
istioctl proxy-status

# For Linkerd, check proxy status
linkerd check --proxy
```

### 8. Check iptables Rules

```bash
# SSH to node and check iptables
ssh <node-ip> iptables-save | grep -E 'FORWARD|KUBE'
```

## Resolution Steps

### 1. Fix CNI Plugin Issues

If CNI plugin is having problems:

```bash
# Restart CNI plugin pods
kubectl rollout restart daemonset <cni-daemonset-name> -n kube-system

# For Calico
kubectl rollout restart daemonset calico-node -n kube-system

# For Cilium
kubectl rollout restart daemonset cilium -n kube-system

# For Weave
kubectl rollout restart daemonset weave-net -n kube-system
```

If needed, reinstall the CNI plugin:

```bash
# Example for Calico
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Example for Cilium
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.9/install/kubernetes/quick-install.yaml
```

### 2. Fix Network Policy Issues

If Network Policies are blocking traffic:

```bash
# Create a permissive Network Policy
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: <namespace>
spec:
  podSelector: {}
  ingress:
  - from:
    - podSelector: {}
  egress:
  - to:
    - podSelector: {}
EOF
```

For cross-namespace communication:

```bash
# Allow traffic from another namespace
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-other-namespace
  namespace: <target-namespace>
spec:
  podSelector: {}
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: <source-namespace>
EOF
```

### 3. Fix MTU Issues

If MTU mismatches are detected:

```bash
# Update CNI configuration with consistent MTU
# For Calico
kubectl patch configmap calico-config -n kube-system --type=merge \
  -p '{"data":{"veth_mtu":"1440"}}'  # Use appropriate MTU value for your network

# Restart CNI pods to apply changes
kubectl rollout restart daemonset calico-node -n kube-system
```

For other CNI plugins, check their documentation for MTU configuration options.

### 4. Fix Pod CIDR Conflicts

If there are CIDR conflicts:

```bash
# Check current Pod CIDR configuration
kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}'

# For Calico, update the IP Pool with non-overlapping CIDR
kubectl patch ippool default-ipv4-ippool -n kube-system --type=merge \
  -p '{"spec":{"cidr":"10.244.0.0/16"}}'  # Use appropriate non-overlapping CIDR
```

The procedure varies by CNI plugin and Kubernetes distribution.

### 5. Fix Routing Issues

If routing tables are incorrect:

```bash
# SSH to node and fix routes
ssh <node-ip>

# Add missing route (example)
sudo ip route add <pod-cidr> via <node-ip> dev <interface>
```

For persistent fixes, update CNI configuration.

### 6. Fix Security Group/Firewall Issues

For cloud environments, check and update security groups/firewall rules:

#### AWS

```bash
# Ensure security groups allow pod-to-pod traffic
aws ec2 authorize-security-group-ingress \
  --group-id <security-group-id> \
  --protocol all \
  --source-group <security-group-id>
```

#### GCP

```bash
# Create firewall rule to allow pod-to-pod traffic
gcloud compute firewall-rules create allow-pod-to-pod \
  --network <network> \
  --allow tcp,udp,icmp \
  --source-ranges <pod-cidr>
```

#### Azure

```bash
# Update network security group to allow pod traffic
az network nsg rule create -g <resource-group> --nsg-name <nsg-name> \
  -n allow-pod-to-pod --priority 100 \
  --source-address-prefixes <pod-cidr> \
  --destination-address-prefixes <pod-cidr> \
  --access Allow --protocol '*' \
  --direction Inbound
```

### 7. Fix Cross-Node Communication

If pods can't communicate across nodes:

```bash
# Check if nodes can communicate with each other
# SSH to one node and ping another node
ssh <node1-ip> ping <node2-ip>

# Check if overlay network tunnels are up
# For Calico
ssh <node-ip> ip link show | grep tunl
```

For flannel, ensure UDP port 8472 is open between nodes.

### 8. Fix Service Mesh Issues

If using a service mesh:

#### Istio

```bash
# Check proxy configuration
istioctl proxy-config listener <pod-name>.<namespace>

# Restart sidecar injection
kubectl delete pod <pod-name> -n <namespace>
```

#### Linkerd

```bash
# Check proxy status and fix issues
linkerd check --proxy
linkerd uninject | linkerd inject - | kubectl apply -f -
```

### 9. Fix Cross-Namespace Service Communication

If services in different namespaces can't communicate:

```bash
# Ensure service is addressed properly
kubectl exec -it netshoot -- nslookup <service-name>.<namespace>.svc.cluster.local

# Create Network Policy allowing cross-namespace service access
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-other-namespace
  namespace: <service-namespace>
spec:
  podSelector:
    matchLabels:
      app: <service-app-label>
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: <client-namespace>
    ports:
    - protocol: TCP
      port: <service-port>
EOF
```

### 10. Fix iptables Rules

If iptables rules are causing problems:

```bash
# SSH to node
ssh <node-ip>

# Flush and recreate problematic rules (USE WITH EXTREME CAUTION)
sudo iptables -F FORWARD
sudo iptables -P FORWARD ACCEPT

# Restart kubelet and kube-proxy to recreate proper rules
sudo systemctl restart kubelet
sudo systemctl restart kube-proxy
```

## Prevention

1. **CNI Monitoring**: Implement monitoring for the CNI plugin
2. **Network Policy Testing**: Test Network Policies in non-production environments
3. **CIDR Planning**: Plan Pod and Service CIDRs to avoid overlaps
4. **MTU Standardization**: Use consistent MTU settings
5. **Cross-Namespace Communication Rules**: Establish clear rules for cross-namespace communication
6. **Regular Testing**: Periodically test pod-to-pod connectivity
7. **Documentation**: Document network architecture and expected behavior
8. **Version Compatibility**: Ensure CNI plugin is compatible with Kubernetes version
9. **Backup Rules**: Backup iptables and routing rules before making changes
10. **Network Observability**: Implement network observability tools

## Related Runbooks

* [CNI Plugin Issues](./cni-plugin-issues.md)
* [Network Policy Issues](./network-policy-issues.md)
* [Service Not Accessible](./service-not-accessible.md)
* [DNS Resolution Problems](./dns-resolution-problems.md)
* [CoreDNS Issues](./coredns-issues.md)
* [Node Not Ready](../cluster/node-not-ready.md)