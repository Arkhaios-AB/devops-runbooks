# Troubleshooting Service Not Accessible

## Symptoms

* Applications unable to connect to Kubernetes Services
* `curl` or `wget` to service IP or DNS name fails
* Applications reporting connection timeouts or connection refused errors
* DNS resolution works but connectivity fails
* Service working from some pods but not others
* Services intermittently accessible
* External access to services failing

## Possible Causes

1. **Selector Mismatch**: Service selector doesn't match pod labels
2. **Pod Health Issues**: Backing pods not running or ready
3. **Incorrect Ports**: Service port configuration doesn't match container ports
4. **Network Policy Restrictions**: NetworkPolicies blocking traffic
5. **kube-proxy Issues**: Problems with kube-proxy on nodes
6. **DNS Resolution Problems**: CoreDNS or kube-dns issues
7. **Endpoint Issues**: Endpoints not properly registered
8. **CIDR Conflicts**: Service CIDR conflicts with pod or node CIDRs
9. **LoadBalancer Problems**: External LoadBalancer not properly configured
10. **CNI Plugin Issues**: Container Network Interface issues

## Diagnosis Steps

### 1. Check Service Configuration and Endpoints

```bash
# Get service details
kubectl get service <service-name> -n <namespace>

# Check detailed service information
kubectl describe service <service-name> -n <namespace>

# Check if endpoints exist for the service
kubectl get endpoints <service-name> -n <namespace>

# Check endpoint details
kubectl describe endpoints <service-name> -n <namespace>
```

### 2. Check Pod Labels and Readiness

```bash
# Check if pods are running with matching labels
kubectl get pods -n <namespace> -l <service-selector>

# Check readiness of backing pods
kubectl get pods -n <namespace> -l <service-selector> -o wide

# Check pod details for any issues
kubectl describe pod <pod-name> -n <namespace>
```

### 3. Test Connectivity from Different Locations

```bash
# Create a debug pod in the same namespace
kubectl run debug-shell --image=nicolaka/netshoot -n <namespace> -- sleep 3600

# Test service DNS resolution
kubectl exec -it debug-shell -n <namespace> -- nslookup <service-name>
kubectl exec -it debug-shell -n <namespace> -- nslookup <service-name>.<namespace>.svc.cluster.local

# Test service connectivity by IP
kubectl exec -it debug-shell -n <namespace> -- curl -v <service-ip>:<service-port>

# Test service connectivity by DNS name
kubectl exec -it debug-shell -n <namespace> -- curl -v <service-name>:<service-port>
kubectl exec -it debug-shell -n <namespace> -- curl -v <service-name>.<namespace>.svc.cluster.local:<service-port>

# Test connectivity directly to pod IP
kubectl exec -it debug-shell -n <namespace> -- curl -v <pod-ip>:<container-port>
```

### 4. Check Network Policies

```bash
# List network policies in the namespace
kubectl get networkpolicy -n <namespace>

# Examine network policy details
kubectl describe networkpolicy <policy-name> -n <namespace>
```

### 5. Check kube-proxy Status

```bash
# Check kube-proxy pods
kubectl get pods -n kube-system -l k8s-app=kube-proxy

# Check kube-proxy logs
kubectl logs -n kube-system -l k8s-app=kube-proxy

# Check kube-proxy configmap
kubectl describe configmap kube-proxy -n kube-system
```

### 6. Check DNS Service

```bash
# Check CoreDNS/kube-dns pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS/kube-dns logs
kubectl logs -n kube-system -l k8s-app=kube-dns
```

### 7. Check External Access (for LoadBalancer/NodePort)

```bash
# For LoadBalancer services, check the external IP
kubectl get service <service-name> -n <namespace> -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

# For NodePort services, check the node port
kubectl get service <service-name> -n <namespace> -o jsonpath='{.spec.ports[0].nodePort}'

# Check if nodes are accessible
curl -v <node-ip>:<node-port>
```

## Resolution Steps

### 1. Fix Selector Mismatch

If the service selector doesn't match pod labels:

```bash
# Check current service selector
kubectl get service <service-name> -n <namespace> -o jsonpath='{.spec.selector}'

# Check pod labels
kubectl get pods -n <namespace> --show-labels

# Update service selector to match pod labels
kubectl patch service <service-name> -n <namespace> \
  --type=json \
  -p='[{"op": "replace", "path": "/spec.selector", "value": {"app": "<correct-label-value>"}}]'

# OR update pod labels to match service selector
kubectl label pod <pod-name> -n <namespace> app=<service-selector-value>
```

### 2. Fix Pod Health Issues

If backing pods are not healthy:

```bash
# Check pod status and readiness
kubectl get pods -n <namespace> -l <service-selector>

# Restart unhealthy pods
kubectl delete pod <pod-name> -n <namespace>

# Fix readiness probe if needed
kubectl edit deployment <deployment-name> -n <namespace>
```

Example of a proper readiness probe:
```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

### 3. Fix Port Configuration

If service ports don't match container ports:

```bash
# Check service port configuration
kubectl get service <service-name> -n <namespace> -o jsonpath='{.spec.ports}'

# Check container port in pod spec
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[0].ports}'

# Update service to match container ports
kubectl edit service <service-name> -n <namespace>
```

Example of correct port mapping:
```yaml
ports:
- name: http
  port: 80         # Service port
  targetPort: 8080  # Container port
  protocol: TCP
```

### 4. Fix Network Policy Issues

If network policies are blocking traffic:

```bash
# Create a policy that allows traffic to the service
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-service-access
  namespace: <namespace>
spec:
  podSelector:
    matchLabels:
      app: <app-label>
  ingress:
  - from:
    - podSelector: {}
    ports:
    - port: <port>
      protocol: TCP
EOF
```

### 5. Restart kube-proxy

If kube-proxy is having issues:

```bash
# Restart kube-proxy pods
kubectl rollout restart daemonset kube-proxy -n kube-system
```

### 6. Fix DNS Issues

If DNS resolution is failing:

```bash
# Restart CoreDNS/kube-dns pods
kubectl rollout restart deployment coredns -n kube-system
```

### 7. Fix Endpoint Registration

If endpoints are not being registered correctly:

```bash
# Delete and recreate the service
kubectl delete service <service-name> -n <namespace>
kubectl create service clusterip <service-name> --tcp=<port>:<target-port> -n <namespace>

# Then set the correct selector
kubectl patch service <service-name> -n <namespace> \
  --type=json \
  -p='[{"op": "replace", "path": "/spec.selector", "value": {"app": "<correct-label-value>"}}]'
```

### 8. Fix LoadBalancer Issues

For LoadBalancer services:

```bash
# Check cloud provider status for the load balancer
# This varies by cloud provider

# Recreate the LoadBalancer service if needed
kubectl delete service <service-name> -n <namespace>
kubectl expose deployment <deployment-name> --type=LoadBalancer --port=<port> --target-port=<target-port> -n <namespace>
```

### 9. Fix NodePort Issues

For NodePort services:

```bash
# Check if firewall is blocking the NodePort
# For AWS security groups, GCP firewall rules, etc.

# Recreate the NodePort service with a specific port if needed
kubectl delete service <service-name> -n <namespace>
kubectl expose deployment <deployment-name> --type=NodePort --port=<port> --target-port=<target-port> --node-port=<node-port> -n <namespace>
```

### 10. Fix CNI Plugin Issues

If CNI plugin is having problems:

```bash
# Check CNI plugin pods
kubectl get pods -n kube-system -l k8s-app=<cni-name>

# Restart CNI pods
kubectl rollout restart daemonset <cni-daemonset> -n kube-system
```

## Prevention

1. **Service Testing**: Regularly test service connectivity as part of monitoring
2. **Standard Labeling**: Adopt consistent labeling strategy for services and pods
3. **Health Checks**: Implement proper readiness and liveness probes
4. **Network Policy Design**: Design network policies carefully with service access in mind
5. **Documentation**: Document service dependencies and expected connectivity
6. **Monitoring**: Monitor service endpoints and connectivity metrics
7. **Load Testing**: Regularly test services under load
8. **DNS Redundancy**: Ensure DNS services are properly sized and redundant
9. **Port Standardization**: Standardize port mappings where possible
10. **Access Logging**: Enable access logging for critical services

## Related Runbooks

* [DNS Resolution Problems](./dns-resolution-problems.md)
* [Ingress Issues](./ingress-issues.md)
* [Network Policy Issues](./network-policy-issues.md)
* [LoadBalancer Service Issues](./loadbalancer-issues.md)
* [NodePort Service Issues](./nodeport-issues.md)
* [CNI Plugin Issues](./cni-plugin-issues.md)
* [Pod-to-Pod Communication Issues](./pod-to-pod-communication.md)
* [CoreDNS Issues](./coredns-issues.md)