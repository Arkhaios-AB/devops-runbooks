# Troubleshooting DNS Resolution Problems

## Symptoms

* Pods cannot resolve service names or external domains
* Applications report connection timeouts or "unknown host" errors
* DNS queries time out or return incorrect results
* Service discovery between pods fails
* Error logs contain DNS-related failures

## Possible Causes

1. **CoreDNS Issues**: CoreDNS pods are not running correctly
2. **kube-dns Service Issues**: kube-dns service is misconfigured
3. **Pod DNS Config Issues**: Incorrect DNS configuration in pods
4. **Network Policy Blocking DNS**: Network policies blocking UDP/TCP port 53
5. **DNS Caching Problems**: Stale DNS cache entries
6. **Cluster DNS Capacity**: DNS service overwhelmed by request volume
7. **CNI Plugin Problems**: Container network interface issues affecting DNS traffic
8. **Node-level DNS Issues**: Node's DNS resolver configuration problems

## Diagnosis Steps

### 1. Check CoreDNS/kube-dns Pods Status

```bash
# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS pod details
kubectl describe pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns
```

### 2. Check DNS Service

```bash
# Check kube-dns service
kubectl get service kube-dns -n kube-system

# Verify endpoints
kubectl get endpoints kube-dns -n kube-system
```

### 3. Test DNS Resolution from a Pod

Create a debug pod to test DNS resolution:

```bash
# Create a debug pod
kubectl run dns-test --image=busybox:1.28 -- sleep 3600

# Test DNS resolution for a service
kubectl exec -it dns-test -- nslookup kubernetes.default.svc.cluster.local

# Test DNS resolution for an external domain
kubectl exec -it dns-test -- nslookup kubernetes.io

# Check DNS configuration in the pod
kubectl exec -it dns-test -- cat /etc/resolv.conf
```

### 4. Check Network Policies

```bash
# List network policies
kubectl get networkpolicies --all-namespaces

# Check if network policies are blocking DNS traffic (port 53)
kubectl describe networkpolicies -n <namespace>
```

### 5. Check CoreDNS Configuration

```bash
# Get CoreDNS ConfigMap
kubectl get configmap coredns -n kube-system -o yaml
```

## Resolution Steps

### 1. Fix CoreDNS Deployment

If CoreDNS pods are not running or are crashing:

```bash
# Scale up CoreDNS if needed
kubectl scale deployment coredns --replicas=2 -n kube-system

# Restart CoreDNS pods
kubectl rollout restart deployment coredns -n kube-system
```

### 2. Fix CoreDNS Configuration

If the CoreDNS configuration is incorrect:

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

### 3. Fix Pod DNS Configuration

If pods have incorrect DNS configuration:

```yaml
# Add this to pod or deployment spec
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

### 4. Fix Network Policies

If network policies are blocking DNS traffic:

```yaml
# Add this to your NetworkPolicy to allow DNS traffic
spec:
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
```

### 5. Scale CoreDNS for Higher Traffic

If DNS service is overwhelmed:

```bash
# Increase CoreDNS replicas
kubectl scale deployment coredns --replicas=3 -n kube-system

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

### 6. Node-level DNS Issues

If nodes have DNS issues:

```bash
# SSH into the affected node
# Check node's DNS configuration
cat /etc/resolv.conf

# Restart systemd-resolved if needed
sudo systemctl restart systemd-resolved
```

## Prevention

1. **Monitor CoreDNS**: Set up monitoring and alerting for CoreDNS pods
2. **Tune DNS Caching**: Adjust TTL and cache settings based on workload
3. **Scale DNS Properly**: Ensure CoreDNS has enough replicas and resources for your cluster size
4. **Use NodeLocal DNSCache**: Consider deploying NodeLocal DNSCache for large clusters
5. **Regular Testing**: Periodically test DNS resolution as part of cluster health checks
6. **Consider DNS Autoscaling**: Enable DNS horizontal autoscaler for dynamic scaling
7. **Network Policy Design**: Carefully design network policies to not block essential services

## Related Runbooks

* [CoreDNS Issues](./coredns-issues.md)
* [Pod-to-Pod Communication Issues](./pod-to-pod-communication.md)
* [Service Not Accessible](./service-not-accessible.md)
* [Network Policy Issues](./network-policy-issues.md)
* [CNI Plugin Issues](./cni-plugin-issues.md)