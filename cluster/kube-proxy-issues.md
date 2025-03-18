# Troubleshooting kube-proxy Issues

## Symptoms

* Service connectivity issues within the cluster
* NodePort services not accessible
* ClusterIP services unreachable
* Network policy rules not being enforced
* Intermittent service connectivity problems
* Slow service response or timeouts
* Load balancing not working properly between pods
* IPVS or iptables rules missing or incorrect
* Kubernetes events related to kube-proxy errors
* Service endpoints not correctly mapped to pods

## Possible Causes

1. **kube-proxy Pod Failures**: kube-proxy pods crashing or not running
2. **Configuration Issues**: Incorrect kube-proxy configuration
3. **Network Plugin Conflicts**: Conflicts with CNI or other network plugins
4. **Iptables/IPVS Issues**: Problems with iptables or IPVS rules
5. **Resource Constraints**: CPU or memory pressure on nodes
6. **Version Incompatibility**: Incompatible versions between kube-proxy and Kubernetes
7. **Proxy Mode Issues**: Problems with specific proxy mode (iptables, IPVS, etc.)
8. **Certificate Problems**: Invalid or expired certificates for kube-proxy
9. **Node Network Issues**: Underlying node networking problems
10. **ConnTrack Issues**: Connection tracking table full or corrupted

## Diagnosis Steps

### 1. Check kube-proxy Status

```bash
# Check if kube-proxy pods are running
kubectl get pods -n kube-system -l k8s-app=kube-proxy

# Describe kube-proxy pods for more details
kubectl describe pods -n kube-system -l k8s-app=kube-proxy

# Check kube-proxy logs
kubectl logs -n kube-system -l k8s-app=kube-proxy
```

### 2. Verify kube-proxy Configuration

```bash
# Check kube-proxy ConfigMap
kubectl get configmap -n kube-system kube-proxy -o yaml

# Check the mode kube-proxy is running in (iptables or IPVS)
kubectl get configmap -n kube-system kube-proxy -o=jsonpath='{.data.config\.conf}' | grep mode
```

### 3. Check iptables Rules

```bash
# SSH into a node
ssh <node-ip>

# Check services-related iptables rules
sudo iptables-save | grep -E 'KUBE-SERVICES|KUBE-SVC'

# Count service rules
sudo iptables-save | grep -c KUBE-SERVICES

# Check specific service with ClusterIP
sudo iptables-save | grep <service-cluster-ip>
```

### 4. Check IPVS Rules (if IPVS mode is enabled)

```bash
# SSH into a node
ssh <node-ip>

# Install ipvsadm if not already installed
sudo apt-get install ipvsadm -y   # For Debian/Ubuntu
# or
sudo yum install ipvsadm -y       # For RHEL/CentOS

# Check IPVS rules
sudo ipvsadm -ln

# Check service-specific IPVS rules
sudo ipvsadm -ln | grep <service-cluster-ip>
```

### 5. Check Network Connectivity

```bash
# Run a test pod
kubectl run test-network --image=nicolaka/netshoot -- sleep 3600

# Execute network tests from the pod
kubectl exec -it test-network -- bash

# Inside the pod, test connectivity to a service
curl <service-name>.<namespace>.svc.cluster.local
```

### 6. Check Resource Usage

```bash
# Check resource usage of kube-proxy
kubectl top pods -n kube-system -l k8s-app=kube-proxy

# Check node resource usage
kubectl top nodes
```

### 7. Check conntrack Table

```bash
# SSH into a node
ssh <node-ip>

# Check conntrack table statistics
sudo conntrack -S

# Check conntrack table usage
sudo conntrack -C

# List current connections
sudo conntrack -L
```

## Resolution Steps

### 1. Fix kube-proxy Pod Issues

If kube-proxy pods are not running or are in a crash loop:

```bash
# Delete kube-proxy pods to force recreation
kubectl delete pods -n kube-system -l k8s-app=kube-proxy

# For DaemonSet-managed kube-proxy, restart the DaemonSet
kubectl rollout restart daemonset kube-proxy -n kube-system
```

### 2. Fix kube-proxy Configuration

If the configuration is incorrect:

```bash
# Edit kube-proxy ConfigMap
kubectl edit configmap kube-proxy -n kube-system
```

Example configuration for iptables mode:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-proxy
  namespace: kube-system
data:
  config.conf: |-
    apiVersion: kubeproxy.config.k8s.io/v1alpha1
    kind: KubeProxyConfiguration
    mode: "iptables"
    ipvs:
      scheduler: "rr"
    iptables:
      masqueradeAll: false
    clusterCIDR: "10.244.0.0/16"
```

After changing the ConfigMap, restart kube-proxy:
```bash
kubectl rollout restart daemonset kube-proxy -n kube-system
```

### 3. Fix Proxy Mode Issues

If you need to change the proxy mode (e.g., from iptables to IPVS):

```bash
# Update ConfigMap with the desired mode
kubectl edit configmap kube-proxy -n kube-system
# Change mode: "iptables" to mode: "ipvs"

# Restart kube-proxy to apply changes
kubectl rollout restart daemonset kube-proxy -n kube-system
```

### 4. Fix IPVS Dependencies (if using IPVS mode)

If IPVS mode is not working due to missing dependencies:

```bash
# SSH into each node
ssh <node-ip>

# For Debian/Ubuntu:
sudo apt-get update
sudo apt-get install -y ipset ipvsadm

# For RHEL/CentOS:
sudo yum install -y ipset ipvsadm

# Load required kernel modules
sudo modprobe ip_vs
sudo modprobe ip_vs_rr
sudo modprobe ip_vs_wrr
sudo modprobe ip_vs_sh
sudo modprobe nf_conntrack_ipv4

# Make modules load at boot
echo "ip_vs" | sudo tee -a /etc/modules-load.d/ipvs.conf
echo "ip_vs_rr" | sudo tee -a /etc/modules-load.d/ipvs.conf
echo "ip_vs_wrr" | sudo tee -a /etc/modules-load.d/ipvs.conf
echo "ip_vs_sh" | sudo tee -a /etc/modules-load.d/ipvs.conf
echo "nf_conntrack_ipv4" | sudo tee -a /etc/modules-load.d/ipvs.conf
```

### 5. Clear iptables Rules if Corrupted

If iptables rules are corrupted:

```bash
# SSH into the node
ssh <node-ip>

# Backup current iptables rules
sudo iptables-save > iptables-backup

# Restart kube-proxy to regenerate rules
kubectl delete pod -n kube-system -l k8s-app=kube-proxy --field-selector spec.nodeName=<node-name>
```

### 6. Increase conntrack Limits

If connection tracking table is full:

```bash
# SSH into the node
ssh <node-ip>

# Check current conntrack limits
sysctl net.netfilter.nf_conntrack_max

# Increase conntrack limits
sudo sysctl -w net.netfilter.nf_conntrack_max=524288
sudo echo "net.netfilter.nf_conntrack_max=524288" >> /etc/sysctl.conf

# Clear conntrack table if necessary
sudo conntrack -F
```

### 7. Fix Version Compatibility Issues

If kube-proxy version is incompatible:

```bash
# Check current Kubernetes and kube-proxy versions
kubectl version
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o jsonpath='{.items[0].spec.containers[0].image}'

# Update kube-proxy image to match Kubernetes version
kubectl set image daemonset/kube-proxy -n kube-system kube-proxy=k8s.gcr.io/kube-proxy:v1.xx.y
```

Replace v1.xx.y with the matching Kubernetes version.

### 8. Fix Certificate Issues

If kube-proxy has certificate problems:

```bash
# For kubeadm-based clusters, renew certificates
sudo kubeadm certs renew all

# Check certificate paths in kube-proxy config
kubectl describe configmap kube-proxy -n kube-system

# Ensure certificates exist and are valid
sudo ls -la /etc/kubernetes/pki/
```

## Prevention

1. **Monitoring**: Implement monitoring for kube-proxy status and performance
2. **Regular Updates**: Keep kube-proxy version in sync with Kubernetes version
3. **Resource Planning**: Allocate sufficient resources for kube-proxy
4. **Connectivity Testing**: Regularly test service connectivity
5. **Config Management**: Use version control for kube-proxy configuration
6. **Backup Rules**: Backup iptables rules periodically
7. **Logging**: Enable detailed logging for easier troubleshooting
8. **Conntrack Tuning**: Tune conntrack parameters based on workload
9. **Certificate Management**: Monitor certificate expiration dates
10. **Node Maintenance**: Include kube-proxy checks in node maintenance procedures

## Related Runbooks

* [NodePort Service Issues](../networking/nodeport-issues.md)
* [Service Not Accessible](../networking/service-not-accessible.md)
* [DNS Resolution Problems](../networking/dns-resolution-problems.md)
* [CNI Plugin Issues](../networking/cni-plugin-issues.md)
* [Network Policy Issues](../networking/network-policy-issues.md)
* [Node Not Ready](./node-not-ready.md)