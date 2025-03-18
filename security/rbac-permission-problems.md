# Troubleshooting RBAC Permission Problems

## Symptoms

* Authentication or authorization errors in application logs
* "Forbidden" or "Unauthorized" responses from the Kubernetes API
* Service accounts unable to perform required operations
* Error messages containing `Error from server (Forbidden): <resource> is forbidden`
* Users or service accounts cannot access resources they should have access to
* CI/CD pipelines failing with permission errors

## Possible Causes

1. **Missing Roles or ClusterRoles**: Required RBAC roles not defined
2. **Missing RoleBindings or ClusterRoleBindings**: RBAC bindings not created
3. **Incorrect Service Account**: Pod using the wrong service account
4. **Insufficient Permissions**: Roles with insufficient resource permissions
5. **Namespace Issues**: Trying to use RoleBindings across namespaces
6. **API Group Mismatches**: Missing API groups in role definitions
7. **Improperly Configured SubjectAccessReview**: Authentication plugin issues

## Diagnosis Steps

### 1. Check Error Messages

First, collect the exact error message. The user, resource type, and verb will be critical for diagnosis:

```
Error from server (Forbidden): deployments.apps "nginx" is forbidden: User "system:serviceaccount:default:default" cannot get resource "deployments" in API group "apps" in the namespace "default"
```

### 2. Check Service Account Configuration

```bash
# List service accounts in the namespace
kubectl get serviceaccounts -n <namespace>

# Check details of a specific service account
kubectl describe serviceaccount <sa-name> -n <namespace>

# Check which service account is used by a pod
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.serviceAccountName}'
```

### 3. Check Existing RBAC Rules

```bash
# List roles in the namespace
kubectl get roles,clusterroles -n <namespace>

# List role bindings in the namespace
kubectl get rolebindings,clusterrolebindings -n <namespace>

# Get details of a specific role
kubectl describe role <role-name> -n <namespace>
kubectl describe clusterrole <clusterrole-name>

# Get details of a specific role binding
kubectl describe rolebinding <rolebinding-name> -n <namespace>
kubectl describe clusterrolebinding <clusterrolebinding-name>
```

### 4. Check Access with auth can-i

```bash
# Check if current user can perform an action
kubectl auth can-i <verb> <resource>

# Check if a service account can perform an action
kubectl auth can-i <verb> <resource> --as=system:serviceaccount:<namespace>:<serviceaccount> -n <namespace>
```

### 5. Audit Logs

If available, check audit logs for authorization failures:

```bash
# Example of filtering audit logs for authorization failures (depends on your setup)
kubectl logs -n kube-system -l component=kube-apiserver | grep "forbidden" | grep <username-or-serviceaccount>
```

## Resolution Steps

### 1. Create Missing Role

If the required role is missing:

```bash
# Create a role with the necessary permissions
kubectl create role <role-name> --verb=<verb> --resource=<resource> -n <namespace>
```

Example:

```bash
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n default
```

For more complex roles:

```bash
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-and-deploy-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
EOF
```

### 2. Create Missing RoleBinding

If the role binding is missing:

```bash
# Create a role binding
kubectl create rolebinding <rolebinding-name> --role=<role-name> --serviceaccount=<namespace>:<serviceaccount> -n <namespace>
```

Example:

```bash
kubectl create rolebinding read-pods --role=pod-reader --serviceaccount=default:default -n default
```

For more complex role bindings:

```bash
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-and-deployments
  namespace: default
subjects:
- kind: ServiceAccount
  name: my-app
  namespace: default
roleRef:
  kind: Role
  name: pod-and-deploy-reader
  apiGroup: rbac.authorization.k8s.io
EOF
```

### 3. Create Missing ClusterRole

For cluster-wide resources:

```bash
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
EOF
```

### 4. Create Missing ClusterRoleBinding

For cluster-wide bindings:

```bash
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes
subjects:
- kind: ServiceAccount
  name: monitoring-app
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
EOF
```

### 5. Modify Pod to Use Correct Service Account

If the pod is using the wrong service account:

```bash
# Edit the deployment to use the correct service account
kubectl edit deployment <deployment-name> -n <namespace>
```

Change the service account in the pod spec:

```yaml
spec:
  template:
    spec:
      serviceAccountName: <correct-service-account>
```

### 6. Use Aggregated ClusterRoles

For complex permission sets, use aggregated roles:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
  labels:
    rbac.authorization.k8s.io/aggregate-to-view: "true"
rules:
- apiGroups: [""]
  resources: ["endpoints", "pods", "services"]
  verbs: ["get", "list", "watch"]
```

## Prevention

1. **Principle of Least Privilege**: Grant only the permissions needed for the application
2. **Use Namespaces**: Isolate applications in different namespaces
3. **Avoid Default Service Account**: Create dedicated service accounts for each application
4. **Consistent Naming**: Use clear naming conventions for roles and bindings
5. **Role Aggregation**: Use role aggregation for complex permission sets
6. **Regular Audits**: Periodically audit RBAC permissions
7. **Documentation**: Maintain documentation of service accounts and their permissions
8. **CI/CD Integration**: Test RBAC permissions as part of CI/CD pipelines

## Related Runbooks

* [ServiceAccount Issues](./service-account-issues.md)
* [Secret and ConfigMap Issues](./secret-configmap-issues.md)
* [API Server Unavailable](../cluster/api-server-unavailable.md)
* [Audit Logging Issues](./audit-logging-issues.md)
* [Pod Security Policy Issues](./pod-security-policy-issues.md)