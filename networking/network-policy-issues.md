# Troubleshooting Network Policy Issues

## Symptoms

* Unexpected connection timeouts between pods
* Services unreachable despite correct DNS resolution
* Intermittent connectivity between namespaces
* Certain traffic flows work while others fail
* External connections failing from specific pods
* DNS queries failing despite CoreDNS/kube-dns functioning
* Applications reporting connection refused or timeout errors
* Traffic allowed that should be blocked by policy
* Ingress controller unable to reach backends
* Monitoring or logging agents unable to send data

## Possible Causes

1. **Missing or Conflicting Network Policies**: Policies blocking legitimate traffic
2. **Selector Mismatch**: Pod/namespace selectors not matching intended targets
3. **CNI Plugin Issues**: Container Network Interface problems
4. **Incorrect Policy Order**: Policies evaluated in unexpected order
5. **Default Deny Misconfiguration**: Default deny policy blocking all traffic
6. **Missing Egress Rules**: Egress traffic blocked by default deny
7. **Missing Ingress Rules**: Ingress traffic blocked by default deny
8. **Protocol Issues**: Policy only allowing specific protocols (TCP but not UDP)
9. **Port Configuration**: Incorrect port specifications in policy
10. **CIDR Range Issues**: Incorrect IP CIDR ranges in policy

## Diagnosis Steps

### 1. Check Existing Network Policies

```bash
# List all network policies in all namespaces
kubectl get networkpolicy --all-namespaces

# Examine specific network policy
kubectl describe networkpolicy <policy-name> -n <namespace>

# Check if default deny policies exist
kubectl get networkpolicy -A | grep default-deny
```

### 2. Test Connectivity Between Pods

```bash
# Create a test pod for connectivity checks
kubectl run test-pod --image=nicolaka/netshoot -n <namespace> -- sleep 3600

# Test connection to a service 
kubectl exec -it test-pod -n <namespace> -- curl -v <service-name>.<service-namespace>.svc.cluster.local:<port>

# Test connection to a pod directly
kubectl exec -it test-pod -n <namespace> -- curl -v <pod-ip>:<port>

# Test DNS resolution
kubectl exec -it test-pod -n <namespace> -- nslookup kubernetes.default.svc.cluster.local
```

### 3. Check Pod Labels

```bash
# Get pod labels for target pods
kubectl get pods -n <namespace> --show-labels

# Compare with selectors in network policy
kubectl get networkpolicy <policy-name> -n <namespace> -o yaml | grep podSelector -A 10
```

### 4. Check CNI Plugin

```bash
# Check CNI pods status
kubectl get pods -n kube-system -l k8s-app=<cni-name>

# Check CNI logs
kubectl logs -n kube-system -l k8s-app=<cni-name>

# Check if CNI supports NetworkPolicy
kubectl get pods -n kube-system | grep -E 'calico|cilium|weave|flannel'
```

### 5. Test with Temporary Policy Removal

```bash
# Temporarily save policy to file
kubectl get networkpolicy <policy-name> -n <namespace> -o yaml > policy-backup.yaml

# Delete policy to test if it's causing the issue
kubectl delete networkpolicy <policy-name> -n <namespace>

# Test connectivity without the policy
# ...

# Reapply the policy after testing
kubectl apply -f policy-backup.yaml
```

## Resolution Steps

### 1. Fix Selector Mismatch

If selectors don't match intended pods:

```bash
# Edit network policy
kubectl edit networkpolicy <policy-name> -n <namespace>
```

Example corrected selector:
```yaml
podSelector:
  matchLabels:
    app: frontend  # Make sure this matches pod labels
```

### 2. Add Missing Rules

If legitimate traffic is being blocked:

```bash
# Add rule to allow specific traffic
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-specific-traffic
  namespace: <namespace>
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
EOF
```

### 3. Allow DNS Traffic

If DNS is blocked:

```bash
# Create policy to allow DNS traffic
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: <namespace>
spec:
  podSelector: {}  # Applies to all pods in namespace
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

### 4. Fix Default Deny Policy

If default deny is too restrictive:

```bash
# Replace overly restrictive default deny
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-with-exceptions
  namespace: <namespace>
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          network-policy: allowed
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
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8
        - 172.16.0.0/12
        - 192.168.0.0/16
EOF
```

### 5. Allow Metrics and Monitoring Traffic

If monitoring is blocked:

```bash
# Allow traffic for Prometheus
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus
  namespace: <namespace>
spec:
  podSelector: {}
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 9090
EOF
```

### 6. Fix Protocol and Port Issues

If specific protocols are required:

```bash
# Allow both TCP and UDP traffic
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-multiple-protocols
  namespace: <namespace>
spec:
  podSelector:
    matchLabels:
      app: multiprotocol
  ingress:
  - ports:
    - protocol: TCP
      port: 8080
    - protocol: UDP
      port: 8080
EOF
```

### 7. Fix CNI Plugin Issues

If CNI plugin is having issues:

```bash
# Restart CNI plugin pods
kubectl rollout restart daemonset <cni-daemonset> -n kube-system

# Check if CNI needs to be reinstalled/updated
kubectl apply -f <updated-cni-manifest>
```

For example, for Calico:
```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

### 8. Fix Namespace Selection

If namespace selection is incorrect:

```bash
# Update policy with correct namespace selector
kubectl edit networkpolicy <policy-name> -n <namespace>
```

Example correct namespace selector:
```yaml
namespaceSelector:
  matchLabels:
    kubernetes.io/metadata.name: backend  # Use the actual namespace name
```

### 9. Implement Allow-All Policy for Testing

For testing that NetworkPolicy is the issue:

```bash
# Create temporary allow-all policy
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-temp
  namespace: <namespace>
spec:
  podSelector: {}
  ingress:
  - {}
  egress:
  - {}
  policyTypes:
  - Ingress
  - Egress
EOF
```

### 10. Correct CIDR Range Issues

If IP blocks have incorrect ranges:

```bash
# Update policy with correct CIDR ranges
kubectl edit networkpolicy <policy-name> -n <namespace>
```

Example correct CIDR configuration:
```yaml
ipBlock:
  cidr: 10.0.0.0/16  # Correct range for cluster pods
  except:
  - 10.0.5.0/24      # Except specific subnet
```

## Prevention

1. **Policy Testing**: Test network policies in dev environments first
2. **Documentation**: Document all network policies and their purposes
3. **Standard Templates**: Use standardized policy templates
4. **Monitoring**: Implement network policy monitoring and visibility
5. **Progressive Implementation**: Implement policies progressively, starting with logging-only mode
6. **Connectivity Testing**: Regularly test connectivity between components
7. **Policy Validation**: Validate policies with network policy validators
8. **Naming Conventions**: Use clear naming conventions for policies
9. **Label Strategy**: Implement consistent labeling strategy for pods and namespaces
10. **Policy Simulation**: Use tools to simulate policy effects before applying

## Related Runbooks

* [Service Not Accessible](./service-not-accessible.md)
* [DNS Resolution Problems](./dns-resolution-problems.md)
* [Ingress Issues](./ingress-issues.md)
* [Pod-to-Pod Communication Issues](./pod-to-pod-communication.md)
* [CNI Plugin Issues](./cni-plugin-issues.md)
* [CoreDNS Issues](./coredns-issues.md)