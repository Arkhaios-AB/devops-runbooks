# Troubleshooting Pod Security Policy Issues

## Symptoms

* Pods failing to create with "forbidden: violates PodSecurityPolicy" errors
* Deployment, StatefulSet, or DaemonSet unable to create pods
* Error messages about security contexts
* Third-party operators failing to deploy pods
* Helm charts failing to install completely
* Error messages about privileged containers, volumes, or capabilities
* Pods created but missing expected capabilities or volume mounts
* Pods unable to run with specific user IDs
* Service accounts unable to deploy workloads
* Admission controller errors related to security

## Possible Causes

1. **Restrictive PodSecurityPolicy**: Policy too restrictive for workload requirements
2. **Missing RBAC**: Service account lacks permission to use appropriate PSP
3. **Privileged Container Requirements**: Workload requires privileged capabilities
4. **Host Path Access Issues**: Container needs host path access that's restricted
5. **User/Group Constraints**: Pod running with user/group IDs outside allowed range
6. **Volume Type Restrictions**: Required volume types not allowed by PSP
7. **PSP Migration Issues**: Issues related to migration to Pod Security Standards
8. **Conflicting Policies**: Multiple PSPs causing confusion or conflicts
9. **PSP Deprecated**: Using deprecated Pod Security Policies
10. **Admission Controller Configuration**: Misconfigured PodSecurity admission controller

## Diagnosis Steps

### 1. Check Error Messages

```bash
# Get detailed error messages
kubectl describe pod <pod-name> -n <namespace>

# Check events for PSP-related errors
kubectl get events -n <namespace> | grep -i security
```

### 2. Check Existing Pod Security Policies

```bash
# List all PodSecurityPolicies (for Kubernetes < 1.25)
kubectl get psp

# View specific PSP details
kubectl describe psp <psp-name>
```

### 3. Check Role Bindings for PSP

```bash
# Check which roles can use specific PSP
kubectl get clusterrole,role --all-namespaces -o yaml | grep <psp-name>

# Check role bindings for service account
kubectl get rolebindings,clusterrolebindings --all-namespaces -o yaml | grep <service-account-name>
```

### 4. Check Pod's Security Context

```bash
# Check the security context in the pod spec
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A 20 "securityContext"
```

### 5. Check Pod Security Standards (for Kubernetes >= 1.23)

```bash
# Check namespace labels for Pod Security Standards
kubectl get namespace <namespace> --show-labels

# Check Pod Security admission configuration
kubectl get --raw /api/v1/namespaces/<namespace>/pods/<pod-name> | grep "kubernetes.io/pod-security"
```

### 6. Check Admission Controller Configuration

```bash
# For clusters using PodSecurity admission controller
kubectl get --raw /api/v1/namespaces

# For clusters with PSP (Kubernetes < 1.25)
kubectl get --raw /apis/policy/v1beta1
```

## Resolution Steps

### 1. Fix Service Account RBAC for PSP (Kubernetes < 1.25)

If service account lacks permission to use PSP:

```bash
# Create role to use PSP
kubectl create role psp:privileged \
  --verb=use \
  --resource=podsecuritypolicies \
  --resource-name=<psp-name> \
  -n <namespace>

# Bind role to service account
kubectl create rolebinding <name>:psp:privileged \
  --role=psp:privileged \
  --serviceaccount=<namespace>:<serviceaccount> \
  -n <namespace>
```

### 2. Create Less Restrictive PSP (Kubernetes < 1.25)

If existing PSPs are too restrictive:

```bash
# Create a less restrictive PSP for specific workloads
kubectl apply -f - <<EOF
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: <psp-name>
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: 'runtime/default'
    seccomp.security.alpha.kubernetes.io/defaultProfileName: 'runtime/default'
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  hostNetwork: false
  hostIPC: false
  hostPID: false
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'MustRunAs'
    ranges:
      - min: 1
        max: 65535
  fsGroup:
    rule: 'MustRunAs'
    ranges:
      - min: 1
        max: 65535
  readOnlyRootFilesystem: false
EOF
```

### 3. Configure Pod Security Context

If pod security context is incompatible with policies:

```bash
# Edit deployment to add appropriate security context
kubectl edit deployment <deployment-name> -n <namespace>
```

Example security context:
```yaml
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      containers:
      - name: <container-name>
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
```

### 4. Modify Pod Security Standards (Kubernetes >= 1.23)

If using Pod Security Standards and encountering restrictions:

```bash
# Modify namespace labels for Pod Security Standards
kubectl label --overwrite namespace <namespace> \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted
```

Available levels:
- `privileged`: Unrestricted policy
- `baseline`: Minimally restrictive policy
- `restricted`: Highly restrictive policy

### 5. Create Custom Security Context Constraints (OpenShift)

For OpenShift environments:

```bash
# Create a custom SCC
kubectl apply -f - <<EOF
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: <scc-name>
allowPrivilegedContainer: false
runAsUser:
  type: MustRunAsRange
seLinuxContext:
  type: MustRunAs
fsGroup:
  type: MustRunAs
supplementalGroups:
  type: RunAsAny
users:
- system:serviceaccount:<namespace>:<serviceaccount>
groups: []
EOF
```

### 6. Fix Privileged Container Requirements

If workload requires privileged access:

```bash
# Create a privileged PSP (use with caution)
kubectl apply -f - <<EOF
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: privileged-psp
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: '*'
spec:
  privileged: true
  allowPrivilegeEscalation: true
  volumes:
  - '*'
  hostNetwork: true
  hostPID: true
  hostIPC: true
  runAsUser:
    rule: 'RunAsAny'
  seLinux:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
EOF

# Then bind this PSP to specific service accounts only
```

### 7. Migrate from PSP to Pod Security Standards (Kubernetes >= 1.23)

As PSPs are deprecated in Kubernetes 1.25+:

```bash
# Step 1: Label namespaces with appropriate Pod Security Standards
kubectl label --overwrite namespace <namespace> \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted

# Step 2: Modify workloads to comply with Pod Security Standards
kubectl edit deployment <deployment-name> -n <namespace>
# Add security context as shown above

# Step 3: Use admission controllers like OPA/Gatekeeper or Kyverno for advanced policies
```

### 8. Fix Host Path Access Issues

If container needs specific host path access:

```bash
# For PSP (Kubernetes < 1.25)
kubectl apply -f - <<EOF
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: hostpath-psp
spec:
  # Other fields...
  volumes:
  - 'hostPath'
  - 'configMap'
  - 'emptyDir'
  - 'projected'
  - 'secret'
  - 'downwardAPI'
  - 'persistentVolumeClaim'
  allowedHostPaths:
  - pathPrefix: "/path/to/allowed/directory"
EOF

# For Pod Security Standards
# Need to use namespace with pod-security.kubernetes.io/enforce=privileged
# or implement custom admission controller
```

### 9. Fix User/Group Constraints

If pod needs to run with specific user/group:

```bash
# For PSP (Kubernetes < 1.25)
kubectl apply -f - <<EOF
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: user-group-psp
spec:
  # Other fields...
  runAsUser:
    rule: 'MustRunAs'
    ranges:
    - min: 1000
      max: 2000
  supplementalGroups:
    rule: 'MustRunAs'
    ranges:
    - min: 1000
      max: 2000
  fsGroup:
    rule: 'MustRunAs'
    ranges:
    - min: 1000
      max: 2000
EOF

# Edit pod to comply with user/group requirements
kubectl edit deployment <deployment-name> -n <namespace>
```

Example:
```yaml
spec:
  template:
    spec:
      securityContext:
        runAsUser: 1001
        fsGroup: 1001
```

### 10. Fix Volume Type Restrictions

If pod needs specific volume types:

```bash
# For PSP (Kubernetes < 1.25)
kubectl apply -f - <<EOF
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: volume-types-psp
spec:
  # Other fields...
  volumes:
  - 'configMap'
  - 'emptyDir'
  - 'projected'
  - 'secret'
  - 'downwardAPI'
  - 'persistentVolumeClaim'
  - 'hostPath'  # Add required volume types
EOF

# For Pod Security Standards
# Need custom admission controller or use baseline/privileged level
```

## Prevention

1. **Start with Audit Mode**: Begin with audit/warn modes before enforce mode
2. **Document Security Requirements**: Document security requirements for all workloads
3. **Namespace Strategy**: Use namespaces with different security levels
4. **Testing**: Test security policies in non-production environments
5. **Standardized Security Contexts**: Create templates for common security contexts
6. **Migration Plan**: Plan migration from PSP to Pod Security Standards
7. **Custom Admission Controllers**: Implement OPA/Gatekeeper or Kyverno for advanced use cases
8. **Regular Audits**: Regularly audit workload security contexts
9. **CI/CD Integration**: Validate security contexts in CI/CD pipelines
10. **Role Documentation**: Document which roles can use which security policies

## Related Runbooks

* [RBAC Permission Problems](./rbac-permission-problems.md)
* [ServiceAccount Issues](./service-account-issues.md)
* [Admission Controller Issues](../cluster/admission-controller-issues.md)
* [Deployment Rollout Issues](../workloads/deployment-rollout-issues.md)
* [Secret and ConfigMap Issues](./secret-configmap-issues.md)
* [Pod Stuck in Pending](../workloads/pod-pending.md)