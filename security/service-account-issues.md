# Troubleshooting ServiceAccount Issues

## Symptoms

* Authentication errors in pods trying to access the Kubernetes API
* "Forbidden" or "Unauthorized" errors when accessing cluster resources
* Applications unable to interact with the Kubernetes API
* Tokens not mounting correctly in pods
* Missing or invalid service account tokens
* Third-party operators or controllers failing with permission errors
* CI/CD pipelines failing with authentication issues
* Applications reporting missing credentials
* ImagePullBackOff errors due to missing image pull secrets
* Unable to access cloud provider resources from pods

## Possible Causes

1. **Missing ServiceAccount**: Pod using a non-existent ServiceAccount
2. **RBAC Misconfiguration**: ServiceAccount lacks necessary permissions
3. **Token Mounting Issues**: ServiceAccount token not mounted correctly
4. **Default ServiceAccount Usage**: Using default ServiceAccount without proper permissions
5. **Token Expiration**: ServiceAccount token expired (in newer Kubernetes versions)
6. **Image Pull Secret Issues**: Missing image pull secrets in ServiceAccount
7. **Cloud Provider Integration**: ServiceAccount not properly configured for cloud provider
8. **Namespace Issues**: ServiceAccount in wrong namespace
9. **Pod Security Context**: Pod security context preventing token access
10. **API Server Authentication**: API server authentication configuration issues

## Diagnosis Steps

### 1. Check Pod's ServiceAccount

```bash
# Get the ServiceAccount used by a pod
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.serviceAccountName}'

# Check if ServiceAccount exists
kubectl get serviceaccount <service-account-name> -n <namespace>

# Examine ServiceAccount details
kubectl describe serviceaccount <service-account-name> -n <namespace>
```

### 2. Check RBAC Configuration

```bash
# List Roles and ClusterRoles
kubectl get roles,clusterroles -n <namespace>

# List RoleBindings and ClusterRoleBindings
kubectl get rolebindings,clusterrolebindings -n <namespace>

# Check specific RoleBinding
kubectl describe rolebinding <rolebinding-name> -n <namespace>

# Check specific ClusterRoleBinding
kubectl describe clusterrolebinding <clusterrolebinding-name>
```

### 3. Check Token Mounting

```bash
# Exec into pod to check token mounting
kubectl exec -it <pod-name> -n <namespace> -- ls -la /var/run/secrets/kubernetes.io/serviceaccount/

# Check token content
kubectl exec -it <pod-name> -n <namespace> -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

### 4. Test API Access from Pod

```bash
# Exec into pod to test API access
kubectl exec -it <pod-name> -n <namespace> -- sh

# Inside the pod, run:
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl -k -H "Authorization: Bearer $TOKEN" https://kubernetes.default.svc/api/v1/namespaces/$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)/pods
```

### 5. Check for Image Pull Secrets

```bash
# Check if ServiceAccount has image pull secrets
kubectl get serviceaccount <service-account-name> -n <namespace> -o yaml | grep imagePullSecrets -A 10

# List available secrets
kubectl get secrets -n <namespace> | grep regcred
```

### 6. Check Authorization with kubectl auth

```bash
# Check if ServiceAccount can perform actions
kubectl auth can-i list pods \
  --as=system:serviceaccount:<namespace>:<service-account-name> \
  -n <namespace>

# Check for specific resource and verb
kubectl auth can-i get deployments \
  --as=system:serviceaccount:<namespace>:<service-account-name> \
  -n <namespace>
```

## Resolution Steps

### 1. Create Missing ServiceAccount

If ServiceAccount doesn't exist:

```bash
# Create a new ServiceAccount
kubectl create serviceaccount <service-account-name> -n <namespace>
```

### 2. Fix RBAC Permissions

If ServiceAccount lacks necessary permissions:

```bash
# Create a Role with required permissions
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: <role-name>
  namespace: <namespace>
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "update"]
EOF

# Bind the Role to the ServiceAccount
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: <rolebinding-name>
  namespace: <namespace>
subjects:
- kind: ServiceAccount
  name: <service-account-name>
  namespace: <namespace>
roleRef:
  kind: Role
  name: <role-name>
  apiGroup: rbac.authorization.k8s.io
EOF
```

For cluster-wide permissions:

```bash
# Create a ClusterRole
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: <clusterrole-name>
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
EOF

# Bind the ClusterRole to the ServiceAccount
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: <clusterrolebinding-name>
subjects:
- kind: ServiceAccount
  name: <service-account-name>
  namespace: <namespace>
roleRef:
  kind: ClusterRole
  name: <clusterrole-name>
  apiGroup: rbac.authorization.k8s.io
EOF
```

### 3. Fix Token Mounting Issues

If token is not mounting correctly:

```bash
# Edit pod or deployment to ensure token mounting
kubectl edit deployment <deployment-name> -n <namespace>
```

Ensure the following is present (or not present, depending on your needs):

```yaml
spec:
  template:
    spec:
      serviceAccountName: <service-account-name>
      automountServiceAccountToken: true  # Set to false to disable mounting
```

### 4. Add Image Pull Secrets to ServiceAccount

If pull secrets are missing:

```bash
# Create image pull secret if needed
kubectl create secret docker-registry regcred \
  --docker-server=<your-registry-server> \
  --docker-username=<your-username> \
  --docker-password=<your-password> \
  --docker-email=<your-email> \
  -n <namespace>

# Add pull secret to ServiceAccount
kubectl patch serviceaccount <service-account-name> -n <namespace> \
  -p '{"imagePullSecrets": [{"name": "regcred"}]}'
```

### 5. Create a New Token (for Kubernetes v1.24+)

For Kubernetes 1.24+ where auto-generated service account tokens are short-lived:

```bash
# Create a long-lived token
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: <service-account-name>-token
  namespace: <namespace>
  annotations:
    kubernetes.io/service-account.name: <service-account-name>
type: kubernetes.io/service-account-token
EOF

# Verify token creation
kubectl describe secret <service-account-name>-token -n <namespace>
```

### 6. Fix Pod Configuration

If pod is not using the correct ServiceAccount:

```bash
# Update pod or deployment
kubectl patch deployment <deployment-name> -n <namespace> \
  --type=json \
  -p='[{"op": "replace", "path": "/spec/template/spec/serviceAccountName", "value": "<service-account-name>"}]'
```

### 7. Configure for Cloud Provider Access

For AWS:
```bash
# Annotate ServiceAccount for IAM role (EKS)
kubectl annotate serviceaccount <service-account-name> -n <namespace> \
  eks.amazonaws.com/role-arn=arn:aws:iam::<account-id>:role/<role-name>
```

For GCP:
```bash
# Annotate ServiceAccount for IAM (GKE)
kubectl annotate serviceaccount <service-account-name> -n <namespace> \
  iam.gke.io/gcp-service-account=<gcp-service-account>@<project-id>.iam.gserviceaccount.com
```

For Azure:
```bash
# Annotate ServiceAccount for AAD (AKS)
kubectl annotate serviceaccount <service-account-name> -n <namespace> \
  azure.workload.identity/client-id=<client-id>
```

### 8. Use Projected Service Account Tokens

For more control over token parameters:

```yaml
volumes:
- name: token
  projected:
    sources:
    - serviceAccountToken:
        path: token
        expirationSeconds: 3600  # 1 hour
        audience: "my-app"
```

### 9. Verify Token with Debug Pod

Create a debug pod to test token functionality:

```bash
kubectl run token-test \
  --image=curlimages/curl \
  --serviceaccount=<service-account-name> \
  -n <namespace> \
  -- sleep 3600

kubectl exec -it token-test -n <namespace> -- sh
# Inside the pod:
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl -k -H "Authorization: Bearer $TOKEN" https://kubernetes.default.svc/api/v1/namespaces
```

### 10. Fix Cross-Namespace Issues

If resources need to be accessed across namespaces:

```bash
# Create appropriate ClusterRole for cross-namespace access
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cross-namespace-reader
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch"]
  # To restrict to specific namespaces:
  # resourceNames: ["namespace1", "namespace2"]
EOF

# Bind to the ServiceAccount
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: <service-account-name>-cross-namespace
subjects:
- kind: ServiceAccount
  name: <service-account-name>
  namespace: <namespace>
roleRef:
  kind: ClusterRole
  name: cross-namespace-reader
  apiGroup: rbac.authorization.k8s.io
EOF
```

## Prevention

1. **Least Privilege**: Apply principle of least privilege to ServiceAccounts
2. **Documentation**: Document ServiceAccount requirements and permissions
3. **ServiceAccount Per Workload**: Create dedicated ServiceAccounts for each workload
4. **Avoid Default ServiceAccount**: Avoid using the default ServiceAccount
5. **Token Management**: Implement proper token lifecycle management
6. **Regular Audits**: Regularly audit ServiceAccount permissions
7. **Automation**: Automate ServiceAccount and RBAC creation
8. **Testing**: Test ServiceAccount permissions in non-production environments
9. **Monitoring**: Monitor and alert on unauthorized access attempts
10. **Naming Conventions**: Use clear naming conventions for ServiceAccounts and roles

## Related Runbooks

* [RBAC Permission Problems](./rbac-permission-problems.md)
* [Secret and ConfigMap Issues](./secret-configmap-issues.md)
* [Image Pull Secret Issues](./image-pull-secret-issues.md)
* [Pod Security Policy Issues](./pod-security-policy-issues.md)
* [API Server Unavailable](../cluster/api-server-unavailable.md)
* [Certificate Issues](../cluster/certificate-issues.md)