# Troubleshooting CoreDNS Issues

## Symptoms

* DNS resolution failures within pods
* `nslookup` or `dig` commands timing out or failing
* Applications reporting "unknown host" or name resolution errors
* Intermittent DNS resolution
* Increased latency for DNS queries
* CoreDNS pods restarting frequently
* High CPU or memory usage by CoreDNS pods
* DNS resolution working for some pods but not others
* Error logs in CoreDNS pods
* External domain resolution failing while internal works (or vice versa)

## Possible Causes

1. **CoreDNS Deployment Issues**: CoreDNS pods not running correctly
2. **kube-dns Service Issues**: kube-dns service not properly configured
3. **Pod DNS Config Issues**: Incorrect DNS configuration in pods
4. **Network Policy Blocking DNS**: Network policies blocking UDP/TCP port 53
5. **CoreDNS Configuration**: Misconfigured Corefile or ConfigMap
6. **Resource Constraints**: CoreDNS pods hitting resource limits
7. **Cluster DNS Capacity**: Too many DNS queries for CoreDNS capacity
8. **CNI Plugin Problems**: Container network interface issues affecting DNS traffic
9. **Node-level DNS Issues**: Node's DNS resolver configuration problems
10. **DNS Caching Issues**: DNS caching problems at various levels

## Diagnosis Steps

### 1. Check CoreDNS Pods Status

```bash
# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS pod detailed status
kubectl describe pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns
```

### 2. Check kube-dns Service

```bash
# Check kube-dns service
kubectl get service kube-dns -n kube-system

# Verify kube-dns service endpoints
kubectl get endpoints kube-dns -n kube-system

# Check service details
kubectl describe service kube-dns -n kube-system
```

### 3. Test DNS Resolution from a Pod

```bash
# Create a debug pod
kubectl run dnsutils --image=tutum/dnsutils -- sleep 3600

# Test internal DNS resolution
kubectl exec -it dnsutils -- nslookup kubernetes.default.svc.cluster.local
kubectl exec -it dnsutils -- nslookup <service-name>.<namespace>.svc.cluster.local

# Test external DNS resolution
kubectl exec -it dnsutils -- nslookup google.com

# Check DNS configuration in the pod
kubectl exec -it dnsutils -- cat /etc/resolv.conf
```

### 4. Check Pod DNS Configuration

```bash
# Inspect a pod's DNS configuration
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A 15 dnsConfig
```

### 5. Check CoreDNS Configuration

```bash
# Get CoreDNS ConfigMap
kubectl get configmap coredns -n kube-system -o yaml
```

### 6. Check Network Policies

```bash
# List network policies that might block DNS
kubectl get networkpolicies --all-namespaces
```

### 7. Check CoreDNS Resource Usage

```bash
# Check CoreDNS resource usage
kubectl top pods -n kube-system -l k8s-app=kube-dns
```

### 8. Check Node DNS Configuration

SSH to a problematic node and check:

```bash
# Check node DNS resolv.conf
cat /etc/resolv.conf

# Check if systemd-resolved is running
systemctl status systemd-resolved
```

## Resolution Steps

### 1. Fix CoreDNS Deployment

If CoreDNS pods are crashing or not running:

```bash
# Restart CoreDNS pods
kubectl rollout restart deployment coredns -n kube-system

# If needed, recreate the deployment
kubectl scale deployment coredns --replicas=0 -n kube-system
kubectl scale deployment coredns --replicas=2 -n kube-system
```

### 2. Fix CoreDNS Configuration

If CoreDNS configuration is incorrect:

```bash
# Edit CoreDNS ConfigMap
kubectl edit configmap coredns -n kube-system
```

Example of a proper Corefile:

```
.:53 {
    errors
    health {
       lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf {
       max_concurrent 1000
    }
    cache 30
    loop
    reload
    loadbalance
}
```

### 3. Fix kube-dns Service

If kube-dns service is misconfigured:

```bash
# Check if kube-dns service has correct selectors
kubectl edit service kube-dns -n kube-system
```

Ensure the selector matches CoreDNS pods:
```yaml
selector:
  k8s-app: kube-dns
```

### 4. Fix Pod DNS Configuration

If pods have incorrect DNS configuration:

```bash
# Edit deployment to add proper DNS config
kubectl edit deployment <deployment-name> -n <namespace>
```

Example of explicit DNS configuration:
```yaml
spec:
  template:
    spec:
      dnsConfig:
        nameservers:
          - 10.96.0.10  # kube-dns service IP
        searches:
          - default.svc.cluster.local
          - svc.cluster.local
          - cluster.local
        options:
          - name: ndots
            value: "5"
```

### 5. Fix Network Policies

If network policies are blocking DNS traffic:

```bash
# Create a policy that allows DNS traffic
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: <namespace>
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
EOF
```

### 6. Fix Resource Constraints

If CoreDNS is hitting resource limits:

```bash
# Update CoreDNS resource limits
kubectl edit deployment coredns -n kube-system
```

Example resource adjustment:
```yaml
resources:
  limits:
    memory: "170Mi"
    cpu: "100m"
  requests:
    memory: "70Mi"
    cpu: "70m"
```

### 7. Scale CoreDNS for Higher Traffic

If DNS service is overwhelmed:

```bash
# Increase CoreDNS replicas
kubectl scale deployment coredns --replicas=3 -n kube-system
```

### 8. Deploy NodeLocal DNSCache

For large clusters, deploy NodeLocal DNSCache:

```bash
# Install NodeLocal DNSCache
kubectl apply -f https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml
```

Then update CoreDNS ConfigMap to work with NodeLocal DNSCache.

### 9. Fix CoreDNS Upstream DNS

If external name resolution is failing:

```bash
# Edit CoreDNS ConfigMap to use specific upstream DNS
kubectl edit configmap coredns -n kube-system
```

Change the forward plugin:
```
forward . 8.8.8.8 8.8.4.4 {
    max_concurrent 1000
}
```

Instead of:
```
forward . /etc/resolv.conf {
    max_concurrent 1000
}
```

### 10. Fix DNS Caching Issues

If DNS caching is causing problems:

```bash
# Edit CoreDNS ConfigMap to adjust cache settings
kubectl edit configmap coredns -n kube-system
```

Adjust the cache plugin:
```
cache {
    success 9984 30
    denial 9984 5
}
```

### 11. Fix Node-level DNS Issues

If nodes have DNS issues:

```bash
# SSH into the node
ssh user@<node-ip>

# Fix DNS resolver configuration
sudo vi /etc/resolv.conf

# If using systemd-resolved, fix its configuration
sudo vi /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved
```

## Prevention

1. **Monitor CoreDNS**: Set up monitoring and alerting for CoreDNS
2. **Resource Planning**: Ensure CoreDNS has adequate resources
3. **Regular Testing**: Periodically test DNS resolution
4. **NodeLocal DNSCache**: Consider deploying NodeLocal DNSCache for large clusters
5. **Redundancy**: Ensure multiple CoreDNS replicas
6. **Configuration Management**: Manage CoreDNS configuration with version control
7. **Network Policy Design**: Design network policies to allow DNS traffic
8. **DNS Metrics**: Collect and analyze DNS metrics
9. **Use DNS Autoscaling**: Enable DNS horizontal autoscaler
10. **Documentation**: Document DNS architecture and troubleshooting procedures

## Related Runbooks

* [DNS Resolution Problems](./dns-resolution-problems.md)
* [Service Not Accessible](./service-not-accessible.md)
* [Network Policy Issues](./network-policy-issues.md)
* [Pod-to-Pod Communication Issues](./pod-to-pod-communication.md)
* [CNI Plugin Issues](./cni-plugin-issues.md)