# Troubleshooting Ingress Issues

## Symptoms

* External HTTP/HTTPS requests to applications fail
* Ingress URLs returning 404, 503, or connection errors
* TLS certificate errors when accessing services
* Intermittent connectivity to backend services
* Incorrect routing of requests to backend services
* Ingress controller pods showing errors or restarts
* Timeout errors when accessing services via Ingress

## Possible Causes

1. **Ingress Controller Issues**: Problems with Ingress controller deployment
2. **Ingress Resource Misconfiguration**: Errors in Ingress resource definition
3. **Backend Service Unavailability**: Backend services not running or not accessible
4. **TLS Certificate Problems**: Issues with TLS certificates or secrets
5. **Path Routing Errors**: Incorrect path configuration in Ingress rules
6. **Host Configuration**: Host header matching issues or DNS configuration
7. **Load Balancer Problems**: External load balancer issues (for cloud environments)
8. **Annotations Misconfiguration**: Incorrect or missing Ingress annotations
9. **Resource Limitations**: Ingress controller hitting resource limits
10. **Network Policy Restrictions**: NetworkPolicies blocking traffic

## Diagnosis Steps

### 1. Check Ingress Controller Status

```bash
# Check which Ingress controller is running
kubectl get pods -A | grep -E 'ingress|nginx-controller|traefik|ambassador|contour'

# Check Ingress controller logs
kubectl logs -n <ingress-namespace> <ingress-controller-pod> -f

# Check Ingress controller service
kubectl get svc -n <ingress-namespace> <ingress-controller-service>
```

### 2. Check Ingress Resource Definition

```bash
# List all Ingress resources
kubectl get ingress --all-namespaces

# Check specific Ingress resource details
kubectl describe ingress <ingress-name> -n <namespace>

# Check the Ingress resource definition
kubectl get ingress <ingress-name> -n <namespace> -o yaml
```

### 3. Check Backend Services and Endpoints

```bash
# Check if backend services exist
kubectl get svc <backend-service> -n <namespace>

# Check if service has endpoints
kubectl get endpoints <backend-service> -n <namespace>

# Check backing pods for the service
kubectl get pods -n <namespace> -l <service-selector>
```

### 4. Check TLS Certificate

```bash
# Check if the TLS secret exists
kubectl get secret <tls-secret-name> -n <namespace>

# Verify certificate data
kubectl get secret <tls-secret-name> -n <namespace> -o jsonpath='{.data.tls\.crt}' | base64 --decode | openssl x509 -text -noout
```

### 5. Test Connectivity Directly

```bash
# Create a debug pod
kubectl run debug-shell --image=nicolaka/netshoot -- sleep 3600

# Test backend service directly
kubectl exec -it debug-shell -- curl -v http://<service-name>.<namespace>.svc.cluster.local:<service-port>
```

### 6. Check External DNS Resolution

```bash
# From outside the cluster
dig <ingress-hostname>

# Check if it resolves to the right IP
nslookup <ingress-hostname>
```

### 7. Check Network Policies

```bash
# List Network Policies in the namespace
kubectl get networkpolicy -n <namespace>

# Check if policies might be blocking ingress traffic
kubectl describe networkpolicy -n <namespace>
```

## Resolution Steps

### 1. Fix Ingress Controller Issues

If Ingress controller is not running properly:

```bash
# Restart Ingress controller deployment
kubectl rollout restart deployment <ingress-controller-deployment> -n <ingress-namespace>

# Check for sufficient resources
kubectl describe deployment <ingress-controller-deployment> -n <ingress-namespace>

# Increase resources if necessary
kubectl edit deployment <ingress-controller-deployment> -n <ingress-namespace>
```

Example resource adjustment:
```yaml
resources:
  limits:
    cpu: "1"
    memory: "1Gi"
  requests:
    cpu: "500m"
    memory: "512Mi"
```

### 2. Fix Ingress Resource Definition

If the Ingress resource is misconfigured:

```bash
# Edit the Ingress resource
kubectl edit ingress <ingress-name> -n <namespace>
```

Example of a properly configured Ingress:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

### 3. Fix Backend Service Issues

If backend services are not available:

```bash
# Check if service exists and has correct selectors
kubectl describe svc <service-name> -n <namespace>

# Check if pods are running and match service selector
kubectl get pods -n <namespace> -l <service-selector>

# Fix service if needed
kubectl edit svc <service-name> -n <namespace>
```

### 4. Fix TLS Certificate Issues

If TLS certificate is invalid, expired, or missing:

```bash
# Create or update TLS secret
kubectl create secret tls <tls-secret-name> \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key \
  -n <namespace> \
  --dry-run=client -o yaml | kubectl apply -f -
```

### 5. Fix Path Routing

If path routing is incorrect:

```bash
# Update Ingress path configuration
kubectl edit ingress <ingress-name> -n <namespace>
```

Example path fix:
```yaml
paths:
- path: /app
  pathType: Prefix  # Use Exact for exact matching
  backend:
    service:
      name: my-service
      port:
        number: 80
```

### 6. Fix Host Configuration

If host header matching is not working:

```bash
# Update DNS to point to Ingress controller's external IP or load balancer
# Then update Ingress host configuration if needed
kubectl edit ingress <ingress-name> -n <namespace>
```

Example host configuration:
```yaml
rules:
- host: myapp.example.com  # Must match DNS and HTTP Host header
  http:
    paths:
    # path configuration
```

### 7. Fix Load Balancer Issues

For cloud environments with load balancer issues:

```bash
# Check load balancer status
kubectl describe svc <ingress-controller-service> -n <ingress-namespace>

# Recreate the load balancer service if needed
kubectl delete svc <ingress-controller-service> -n <ingress-namespace>
kubectl expose deployment <ingress-controller-deployment> \
  --type=LoadBalancer \
  --port=80 \
  --target-port=80 \
  -n <ingress-namespace>
```

### 8. Fix Ingress Class

If Ingress class is missing or incorrect:

```bash
# Update Ingress class
kubectl patch ingress <ingress-name> -n <namespace> \
  --type=json \
  -p='[{"op": "replace", "path": "/spec/ingressClassName", "value": "nginx"}]'
```

### 9. Fix Ingress Annotations

If Ingress controller-specific annotations are needed:

```bash
# Add necessary annotations
kubectl annotate ingress <ingress-name> -n <namespace> \
  nginx.ingress.kubernetes.io/rewrite-target="/" \
  nginx.ingress.kubernetes.io/ssl-redirect="true"
```

Common annotations for NGINX Ingress controller:
```
nginx.ingress.kubernetes.io/rewrite-target: /
nginx.ingress.kubernetes.io/ssl-redirect: "true"
nginx.ingress.kubernetes.io/proxy-body-size: "10m"
nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
```

### 10. Fix Network Policy Issues

If NetworkPolicies are blocking Ingress traffic:

```bash
# Create a policy that allows ingress traffic
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-traffic
  namespace: <namespace>
spec:
  podSelector:
    matchLabels:
      app: <app-label>
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: <ingress-namespace>
    ports:
    - port: <port>
      protocol: TCP
EOF
```

## Prevention

1. **Ingress Testing**: Regularly test Ingress routes as part of monitoring
2. **TLS Certificate Monitoring**: Monitor TLS certificate expiration
3. **Backend Health Checks**: Implement health checks for all backend services
4. **Documentation**: Document Ingress configurations and dependencies
5. **Standardized Annotations**: Use consistent annotations for similar Ingress resources
6. **Multi-Zone Redundancy**: Deploy Ingress controllers across multiple zones
7. **Resource Planning**: Ensure Ingress controllers have adequate resources
8. **Logs and Metrics**: Collect and monitor Ingress controller logs and metrics
9. **Automation**: Automate certificate renewal and validation
10. **Testing**: Test Ingress configurations before production deployment

## Related Runbooks

* [Service Not Accessible](./service-not-accessible.md)
* [LoadBalancer Service Issues](./loadbalancer-issues.md)
* [Certificate Issues](../cluster/certificate-issues.md)
* [DNS Resolution Problems](./dns-resolution-problems.md)
* [Network Policy Issues](./network-policy-issues.md)
* [Pod-to-Pod Communication Issues](./pod-to-pod-communication.md)