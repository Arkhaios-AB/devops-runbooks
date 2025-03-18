# Troubleshooting Secret and ConfigMap Issues

## Symptoms

* Applications can't access expected configuration data
* Pods failing to start with errors about missing files or volumes
* "ConfigMap not found" or "Secret not found" errors in pod events
* Applications reporting missing environment variables
* Changes to ConfigMaps or Secrets not reflected in running pods
* Unexpected values in application configuration
* Permission denied errors when accessing mounted ConfigMaps or Secrets
* Application crashes after ConfigMap or Secret updates
* Volume mount errors for ConfigMap or Secret volumes
* Base64 decoding errors in applications consuming Secrets

## Possible Causes

1. **Missing ConfigMap/Secret**: Referenced ConfigMap or Secret doesn't exist
2. **Wrong Namespace**: ConfigMap or Secret exists in different namespace
3. **Key Not Found**: Referenced key doesn't exist in ConfigMap or Secret
4. **Mount Issues**: Incorrect volume mount configuration
5. **Update Propagation Delay**: Changes not yet propagated to running pods
6. **Encoding Problems**: Incorrect encoding or formatting in Secret data
7. **Permission Issues**: Incorrect permissions on mounted files
8. **Immutable ConfigMaps/Secrets**: Immutable objects that can't be updated
9. **Size Limits**: ConfigMap or Secret exceeding size limits
10. **RBAC Issues**: Service account lacks permission to access Secrets

## Diagnosis Steps

### 1. Check ConfigMap/Secret Existence

```bash
# List ConfigMaps in namespace
kubectl get configmaps -n <namespace>

# List Secrets in namespace
kubectl get secrets -n <namespace>

# Check specific ConfigMap
kubectl describe configmap <configmap-name> -n <namespace>

# Check specific Secret (without revealing values)
kubectl describe secret <secret-name> -n <namespace>
```

### 2. Check Pod Configuration

```bash
# Check pod's environment variables and volume mounts
kubectl describe pod <pod-name> -n <namespace>

# Check pod events for ConfigMap/Secret errors
kubectl get events -n <namespace> | grep <pod-name>
```

### 3. Check Mounts Inside Container

```bash
# Exec into container to check mounted files
kubectl exec -it <pod-name> -n <namespace> -- ls -la /path/to/mount

# Check environment variables inside container
kubectl exec -it <pod-name> -n <namespace> -- env | grep <expected-var>
```

### 4. Check Pod YAML Definition

```bash
# Get pod YAML to examine ConfigMap/Secret references
kubectl get pod <pod-name> -n <namespace> -o yaml
```

Look for:
- Environment variables from ConfigMap/Secret
- Volume mounts for ConfigMap/Secret
- References to ConfigMap/Secret in init containers

### 5. Check Secret Content (if needed)

```bash
# Get Secret data (be cautious with sensitive information)
kubectl get secret <secret-name> -n <namespace> -o jsonpath='{.data.<key-name>}' | base64 --decode
```

### 6. Check ConfigMap Content

```bash
# Get ConfigMap data
kubectl get configmap <configmap-name> -n <namespace> -o jsonpath='{.data.<key-name>}'
```

## Resolution Steps

### 1. Create Missing ConfigMap/Secret

If ConfigMap or Secret doesn't exist:

```bash
# Create ConfigMap from literal values
kubectl create configmap <configmap-name> \
  --from-literal=key1=value1 \
  --from-literal=key2=value2 \
  -n <namespace>

# Create ConfigMap from file
kubectl create configmap <configmap-name> \
  --from-file=path/to/file.conf \
  -n <namespace>

# Create Secret from literal values
kubectl create secret generic <secret-name> \
  --from-literal=username=admin \
  --from-literal=password=secret \
  -n <namespace>

# Create Secret from file
kubectl create secret generic <secret-name> \
  --from-file=path/to/credentials.json \
  -n <namespace>
```

### 2. Fix ConfigMap/Secret References in Pod

If pod is referencing wrong ConfigMap/Secret:

```bash
# Edit deployment to fix references
kubectl edit deployment <deployment-name> -n <namespace>
```

Example of proper ConfigMap reference in environment variables:
```yaml
env:
- name: CONFIG_VALUE
  valueFrom:
    configMapKeyRef:
      name: correct-configmap-name  # Fix this
      key: correct-key  # Fix this
      optional: false  # Set to true if the ConfigMap might not exist
```

Example of proper Secret reference in environment variables:
```yaml
env:
- name: SECRET_VALUE
  valueFrom:
    secretKeyRef:
      name: correct-secret-name  # Fix this
      key: correct-key  # Fix this
      optional: false  # Set to true if the Secret might not exist
```

### 3. Fix Volume Mount Issues

If volume mounts are misconfigured:

```bash
# Edit deployment to fix volume mounts
kubectl edit deployment <deployment-name> -n <namespace>
```

Example of proper ConfigMap volume mount:
```yaml
volumes:
- name: config-volume
  configMap:
    name: correct-configmap-name
    items:  # Optional, specify keys to include
    - key: config.yaml
      path: my-config.yaml
    optional: false  # Set to true if the ConfigMap might not exist

containers:
- name: app
  volumeMounts:
  - name: config-volume
    mountPath: /etc/config
    readOnly: true
```

Example of proper Secret volume mount:
```yaml
volumes:
- name: secret-volume
  secret:
    secretName: correct-secret-name
    items:  # Optional, specify keys to include
    - key: tls.crt
      path: tls.crt
    - key: tls.key
      path: tls.key
    optional: false  # Set to true if the Secret might not exist

containers:
- name: app
  volumeMounts:
  - name: secret-volume
    mountPath: /etc/secrets
    readOnly: true
```

### 4. Fix Key Reference Issues

If referencing non-existent keys:

```bash
# Check available keys in ConfigMap
kubectl get configmap <configmap-name> -n <namespace> -o json | jq '.data | keys[]'

# Check available keys in Secret
kubectl get secret <secret-name> -n <namespace> -o json | jq '.data | keys[]'

# Update ConfigMap with missing key
kubectl patch configmap <configmap-name> -n <namespace> \
  --type=merge \
  -p '{"data":{"missing-key":"value"}}'

# Update Secret with missing key
kubectl patch secret <secret-name> -n <namespace> \
  --type=merge \
  -p '{"data":{"missing-key":"dmFsdWU="}}'  # base64-encoded value
```

### 5. Force Pod Refresh for ConfigMap/Secret Updates

If changes to ConfigMap/Secret aren't reflected:

```bash
# Restart deployment to pick up new ConfigMap/Secret values
kubectl rollout restart deployment <deployment-name> -n <namespace>
```

### 6. Fix Secret Encoding Issues

If Secret has encoding problems:

```bash
# Create a properly encoded Secret
ENCODED_VALUE=$(echo -n "my-secret-value" | base64)
kubectl create secret generic <secret-name> \
  --from-literal=key=$ENCODED_VALUE \
  -n <namespace> \
  --dry-run=client -o yaml | kubectl apply -f -
```

### 7. Fix Immutable ConfigMap/Secret

If ConfigMap/Secret is marked immutable:

```bash
# Check if ConfigMap is immutable
kubectl get configmap <configmap-name> -n <namespace> -o yaml | grep immutable

# Create a new ConfigMap with updated content
kubectl create configmap <new-configmap-name> \
  --from-literal=key1=new-value1 \
  -n <namespace>

# Update deployment to use new ConfigMap
kubectl set env deployment/<deployment-name> --from=configmap/<new-configmap-name> -n <namespace>
```

### 8. Fix RBAC Issues

If service account lacks permissions:

```bash
# Create or update role with ConfigMap/Secret access
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-secret-reader
  namespace: <namespace>
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list", "watch"]
EOF

# Bind role to service account
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-configmaps-secrets
  namespace: <namespace>
subjects:
- kind: ServiceAccount
  name: <service-account>
  namespace: <namespace>
roleRef:
  kind: Role
  name: configmap-secret-reader
  apiGroup: rbac.authorization.k8s.io
EOF
```

### 9. Fix Large ConfigMap/Secret Issues

If ConfigMap or Secret exceeds size limits (1MB):

```bash
# Split large ConfigMap into multiple smaller ones
kubectl create configmap <configmap-name>-part1 \
  --from-file=part1.conf \
  -n <namespace>

kubectl create configmap <configmap-name>-part2 \
  --from-file=part2.conf \
  -n <namespace>

# Then update deployment to mount both
```

### 10. Use Environment Variable Substitution

For complex environment variable dependencies:

```yaml
env:
- name: SERVICE_URL
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: service-url
- name: FULL_URL
  value: "$(SERVICE_URL)/api/v1"  # Uses value from SERVICE_URL
```

## Prevention

1. **Naming Convention**: Use consistent naming for ConfigMaps and Secrets
2. **Version Labeling**: Label ConfigMaps and Secrets with version information
3. **CI/CD Integration**: Automate ConfigMap and Secret creation/updates in CI/CD
4. **Configuration Validation**: Validate configuration before deployment
5. **Secret Management**: Use proper secret management tools (Vault, etc.)
6. **Documentation**: Document ConfigMap and Secret dependencies
7. **Application Resiliency**: Design applications to handle missing configuration gracefully
8. **Dynamic Configuration**: Consider solutions for dynamic configuration updates
9. **Mount Subpaths**: Mount specific files instead of entire ConfigMaps when possible
10. **Immutability**: Use immutable ConfigMaps for critical configurations

## Related Runbooks

* [ServiceAccount Issues](./service-account-issues.md)
* [RBAC Permission Problems](./rbac-permission-problems.md)
* [Image Pull Secret Issues](./image-pull-secret-issues.md)
* [Pod CrashLoopBackOff](../workloads/pod-crashloopbackoff.md)
* [Pod Stuck in ContainerCreating](../workloads/pod-container-creating.md)
* [Deployment Rollout Issues](../workloads/deployment-rollout-issues.md)