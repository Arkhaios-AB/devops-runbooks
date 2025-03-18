# Troubleshooting Service Mesh Issues

## Symptoms

* Service-to-service communication failures
* High latency between services
* Request timeouts or connection resets
* Circuit breaker activations
* TLS handshake failures
* Authorization failures between services
* Intermittent 503 Service Unavailable errors
* Inconsistent request routing
* Missing telemetry data
* Proxy initialization errors
* Services unable to register with the mesh
* Control plane component failures (Istiod, etc.)
* Certificate rotation failures
* Load balancing not working as expected
* Rate limiting not applying correctly

## Possible Causes

1. **Proxy Sidecar Issues**: Problems with service mesh proxies (Envoy, Linkerd proxy)
2. **Control Plane Problems**: Issues with the service mesh control plane
3. **Configuration Errors**: Misconfigured traffic management, security, or telemetry settings
4. **Version Incompatibility**: Incompatible versions between mesh components
5. **Certificate Issues**: Expired or invalid mTLS certificates
6. **Resource Constraints**: CPU or memory pressure on proxies or control plane
7. **Network Policy Conflicts**: Conflicts between service mesh policies and Kubernetes NetworkPolicies
8. **CNI Plugin Conflicts**: Interaction issues between mesh and CNI plugins
9. **Improper Installation**: Incorrect installation or upgrade of service mesh components
10. **Application Code Issues**: Application not prepared for service mesh integration

## Diagnosis Steps

### 1. Check Service Mesh Control Plane Status

#### Istio

```bash
# Check Istio control plane pods
kubectl get pods -n istio-system

# Check Istio control plane services
kubectl get svc -n istio-system

# Check Istio version
istioctl version

# Verify Istio proxy connectivity
istioctl proxy-status
```

#### Linkerd

```bash
# Check Linkerd control plane status
linkerd check

# Check Linkerd control plane pods
kubectl get pods -n linkerd

# Check Linkerd version
linkerd version
```

### 2. Check Service Mesh Proxy/Sidecar Status

#### Istio

```bash
# Check if pods have Istio sidecar injected
kubectl get pods -n <namespace> -o jsonpath='{.items[*].spec.containers[*].name}'

# Check Envoy sidecar logs
kubectl logs <pod-name> -n <namespace> -c istio-proxy

# Check Envoy proxy configuration
istioctl proxy-config all <pod-name>.<namespace>

# Check specific Envoy proxy routes
istioctl proxy-config routes <pod-name>.<namespace>

# Check Envoy proxy clusters (upstream services)
istioctl proxy-config clusters <pod-name>.<namespace>

# Check Envoy proxy listeners
istioctl proxy-config listeners <pod-name>.<namespace>
```

#### Linkerd

```bash
# Check if pods have Linkerd proxy injected
kubectl get pods -n <namespace> -o jsonpath='{.items[*].spec.containers[*].name}'

# Check Linkerd proxy logs
kubectl logs <pod-name> -n <namespace> -c linkerd-proxy

# Check Linkerd proxy status for specific pod
linkerd stat pods/<pod-name> -n <namespace>

# Check Linkerd proxy routing table
linkerd routes -n <namespace> pod/<pod-name>
```

### 3. Validate Service Mesh Configuration

#### Istio

```bash
# Analyze Istio configuration for issues
istioctl analyze -n <namespace>

# List Virtual Services
kubectl get virtualservices -A

# List Destination Rules
kubectl get destinationrules -A

# List Gateway configurations
kubectl get gateways -A

# List Service Entries
kubectl get serviceentries -A

# List Authorization Policies
kubectl get authorizationpolicies -A
```

#### Linkerd

```bash
# Validate Linkerd service profiles
kubectl get serviceprofiles -A

# Check traffic split configurations
kubectl get trafficsplit -A
```

### 4. Test Service Connectivity

```bash
# Deploy a debug container
kubectl run temp-debug --image=curlimages/curl -i --tty --rm -- sh

# Or exec into an existing pod
kubectl exec -it <pod-name> -n <namespace> -c <container-name> -- sh

# Test connectivity to a service
curl -v http://<service-name>.<namespace>.svc.cluster.local:<port>/path

# Check with headers (for Istio routing)
curl -v -H "x-user: test" http://<service-name>.<namespace>.svc.cluster.local:<port>/path
```

### 5. Check mTLS Status

#### Istio

```bash
# Check mTLS status
istioctl authn tls-check <pod-name>.<namespace> <service-name>.<namespace>.svc.cluster.local

# Check PeerAuthentication policies
kubectl get peerauthentication -A -o yaml
```

#### Linkerd

```bash
# Check mTLS status for a pod
linkerd edges pod <pod-name> -n <namespace>

# Verify identity configuration
linkerd check --proxy
```

### 6. Check Service Mesh Resource Usage

```bash
# Check CPU and memory usage of control plane components
kubectl top pods -n istio-system  # or linkerd namespace

# Check CPU and memory usage of proxies
kubectl top pods -n <namespace> --containers=true | grep -E 'istio-proxy|linkerd-proxy'
```

## Resolution Steps

### 1. Fix Control Plane Issues

#### Istio

```bash
# Restart Istiod
kubectl rollout restart deployment istiod -n istio-system

# Apply recommended Istio configuration profile
istioctl install --set profile=default

# Upgrade Istio to fix version issues
istioctl upgrade --set values.pilot.resources.requests.memory=2Gi

# Enable internal debug logging
kubectl edit deployment istiod -n istio-system
# Add "--log_output_level=default:debug" to the command args
```

#### Linkerd

```bash
# Restart Linkerd control plane
kubectl rollout restart deployment linkerd-controller -n linkerd
kubectl rollout restart deployment linkerd-destination -n linkerd
kubectl rollout restart deployment linkerd-identity -n linkerd

# Reinstall Linkerd if needed
linkerd install | kubectl apply -f -
```

### 2. Fix Proxy/Sidecar Issues

#### Istio

```bash
# Restart proxies in a namespace
kubectl rollout restart deployment -n <namespace>

# Manually inject sidecar into a deployment if automatic injection isn't working
kubectl get deployment <deployment-name> -n <namespace> -o yaml | istioctl kube-inject -f - | kubectl apply -f -

# Increase proxy resource limits if needed
kubectl edit deployment <deployment-name> -n <namespace>
# Add or modify in the istio-proxy container:
# resources:
#   limits:
#     cpu: 1000m
#     memory: 1Gi
#   requests:
#     cpu: 100m
#     memory: 128Mi
```

#### Linkerd

```bash
# Reinstall Linkerd proxies
kubectl rollout restart deployment -n <namespace>

# Manually inject Linkerd proxy
kubectl get deployment <deployment-name> -n <namespace> -o yaml | linkerd inject - | kubectl apply -f -
```

### 3. Fix Configuration Errors

#### Istio

Virtual Service example for fixing routing issues:
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: service-routes
  namespace: <namespace>
spec:
  hosts:
  - <service-name>
  http:
  - route:
    - destination:
        host: <service-name>
        subset: v1
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: connect-failure,refused-stream,unavailable,cancelled,resource-exhausted
```

Destination Rule example for fixing load balancing issues:
```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: service-circuit-breaker
  namespace: <namespace>
spec:
  host: <service-name>
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

#### Linkerd

Service Profile example for fixing retries and timeouts:
```yaml
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: <service-name>.<namespace>.svc.cluster.local
  namespace: <namespace>
spec:
  routes:
  - name: GET /api/v1/users
    condition:
      method: GET
      pathRegex: /api/v1/users
    responseClasses:
    - condition:
        status:
          min: 500
          max: 599
      isRetryable: true
    timeout: 2s
    retryBudget:
      ttl: 10s
      minRetriesPerSecond: 5
      retryRatio: 0.2
```

### 4. Fix mTLS Issues

#### Istio

PeerAuthentication example to enable mTLS:
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: <namespace>
spec:
  mtls:
    mode: STRICT
```

Authorization Policy example:
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-internal
  namespace: <namespace>
spec:
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/<namespace>/sa/<serviceaccount>"]
    to:
    - operation:
        methods: ["GET"]
        paths: ["/api/*"]
```

#### Linkerd

```bash
# Enable identity on specific namespace
kubectl annotate ns <namespace> linkerd.io/inject=enabled
```

### 5. Fix Resource Constraints

For control plane:
```bash
# Increase resources for Istio control plane
kubectl edit deployment istiod -n istio-system
# Update resources:
# resources:
#   requests:
#     cpu: 500m
#     memory: 2Gi
#   limits:
#     cpu: 2000m
#     memory: 4Gi
```

For proxies:
```bash
# Create or edit global proxy configuration for Istio
kubectl edit configmap istio -n istio-system
# Add or modify:
# data:
#   mesh: |-
#     defaultConfig:
#       resources:
#         requests:
#           cpu: 100m
#           memory: 128Mi
#         limits:
#           cpu: 1000m
#           memory: 512Mi
```

### 6. Fix Certificate Issues

#### Istio

```bash
# Check certificate validity
istioctl proxy-config secret <pod-name>.<namespace>

# Rotate Istio certificates
kubectl delete secret istio-ca-secret -n istio-system
kubectl rollout restart deployment istiod -n istio-system
kubectl rollout restart deployment -n <namespace> # To update proxies
```

#### Linkerd

```bash
# Check certificate validity and expiration
linkerd check --proxy

# Rotate Linkerd certificates
# Follow Linkerd docs for certificate rotation procedure based on version
```

### 7. Clean Restart of Service Mesh

If all else fails, consider a clean reinstallation:

#### Istio

```bash
# Remove Istio
istioctl x uninstall --purge

# Remove Istio CRDs
kubectl delete -f https://raw.githubusercontent.com/istio/istio/release-1.11/manifests/charts/base/crds/crd-all.gen.yaml

# Reinstall Istio with desired profile
istioctl install --set profile=default
```

#### Linkerd

```bash
# Uninstall Linkerd
linkerd install --ignore-cluster | kubectl delete -f -

# Reinstall Linkerd
linkerd install | kubectl apply -f -
linkerd viz install | kubectl apply -f -
```

## Prevention

1. **Regular Health Checks**: Implement regular service mesh health checks
   ```bash
   # For Istio
   istioctl analyze -A
   
   # For Linkerd
   linkerd check
   ```

2. **Resource Planning**: Allocate sufficient resources for proxies and control plane
   ```yaml
   # Example Istio proxy resource settings
   spec:
     containers:
     - name: istio-proxy
       resources:
         requests:
           cpu: 100m
           memory: 128Mi
         limits:
           cpu: 1000m
           memory: 512Mi
   ```

3. **Circuit Breaking**: Configure circuit breakers to prevent cascading failures
   ```yaml
   # Example Istio circuit breaker
   apiVersion: networking.istio.io/v1beta1
   kind: DestinationRule
   metadata:
     name: circuit-breaker
   spec:
     host: service-name
     trafficPolicy:
       connectionPool:
         tcp:
           maxConnections: 100
         http:
           http1MaxPendingRequests: 1
           maxRequestsPerConnection: 1
       outlierDetection:
         consecutive5xxErrors: 5
         interval: 30s
         baseEjectionTime: 30s
   ```

4. **Traffic Management**: Implement proper traffic management policies
5. **Certificate Monitoring**: Monitor certificate expiration dates
6. **Version Compatibility**: Ensure compatibility between service mesh versions
7. **Canary Deployments**: Use canary deployments for service mesh upgrades
8. **Logging and Monitoring**: Implement comprehensive logging and monitoring
9. **Automated Testing**: Regularly test service mesh functionality
10. **Documentation**: Document service mesh configuration and troubleshooting procedures

## Related Runbooks

* [Pod-to-Pod Communication Issues](./pod-to-pod-communication.md)
* [Service Not Accessible](./service-not-accessible.md)
* [DNS Resolution Problems](./dns-resolution-problems.md)
* [Network Policy Issues](./network-policy-issues.md)
* [CNI Plugin Issues](./cni-plugin-issues.md)
* [Certificate Issues](../cluster/certificate-issues.md)