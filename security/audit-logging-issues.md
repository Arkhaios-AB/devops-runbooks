# Troubleshooting Audit Logging Issues

## Symptoms

* Missing audit logs
* Incomplete audit events
* High API server resource usage due to audit logging
* Excessive audit log size
* Audit logs not being sent to expected destinations
* API server performance degradation
* Error messages related to audit configuration
* Unexpected audit policy behavior
* Storage issues related to audit logs
* Audit logs not matching expected format or content

## Possible Causes

1. **Misconfigured Audit Policy**: Incorrect audit policy configuration
2. **Missing or Incorrect Audit Webhook**: Webhook configuration issues
3. **API Server Configuration**: API server not configured for audit logging
4. **Storage Backend Issues**: Problems with storage backend for audit logs
5. **Performance Impact**: Excessive audit logging causing performance problems
6. **Log Rotation Issues**: Audit logs not being rotated properly
7. **Audit Log Parsing Issues**: Errors in parsing or processing audit logs
8. **Webhook Connectivity Problems**: Webhooks unable to connect to audit backends
9. **Incomplete Event Capture**: Audit policy not capturing required events
10. **Version Compatibility**: Audit configuration incompatible with Kubernetes version

## Diagnosis Steps

### 1. Check API Server Audit Configuration

```bash
# For kubeadm clusters, check API server pod
kubectl get pods -n kube-system -l component=kube-apiserver -o yaml | grep -A 20 "audit"

# For non-kubeadm clusters, check API server configuration
# Location depends on installation method and platform
```

### 2. Check Audit Policy

```bash
# If using kubeadm, check the policy file
kubectl get pod -n kube-system -l component=kube-apiserver -o yaml | grep audit-policy.yaml
kubectl exec -it <apiserver-pod> -n kube-system -- cat <audit-policy-path>

# For managed clusters, check documentation for audit policy configuration
```

### 3. Check Audit Log Files

For file-based audit logging:

```bash
# SSH into control plane node and check log files
ssh <control-plane-node>
ls -la <audit-log-path>
tail -f <audit-log-file>
```

### 4. Check Audit Webhook Configuration

For webhook-based audit logging:

```bash
# Check webhook configuration
kubectl get pod -n kube-system -l component=kube-apiserver -o yaml | grep audit-webhook
```

### 5. Check API Server Logs

```bash
# Check API server logs for audit-related issues
kubectl logs -n kube-system -l component=kube-apiserver | grep -i audit
```

### 6. Check API Server Resource Usage

```bash
# Check API server pod resource usage
kubectl top pod -n kube-system -l component=kube-apiserver
```

## Resolution Steps

### 1. Fix Audit Policy Configuration

If audit policy is misconfigured:

```bash
# Create or update audit policy file
cat > audit-policy.yaml <<EOF
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log pod changes at RequestResponse level
  - level: RequestResponse
    resources:
    - group: ""
      resources: ["pods"]
  
  # Log configmaps and secrets at Metadata level
  - level: Metadata
    resources:
    - group: ""
      resources: ["configmaps", "secrets"]

  # Log authentication at Metadata level
  - level: Metadata
    users: ["system:anonymous"]
    verbs: ["authenticate"]

  # Log all other resources at level Request
  - level: Request
    resources:
    - group: ""
  
  # Don't log requests to certain non-resource endpoints
  - level: None
    nonResourceURLs:
      - "/healthz*"
      - "/version"
      - "/swagger*"
EOF

# Update API server configuration to use the new policy
# For kubeadm:
sudo cp audit-policy.yaml /etc/kubernetes/audit-policy.yaml
sudo chmod 644 /etc/kubernetes/audit-policy.yaml

# Then edit API server pod manifest
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

Add audit configuration to the API server:
```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit/audit.log
    - --audit-log-maxage=30
    - --audit-log-maxbackup=10
    - --audit-log-maxsize=100
    volumeMounts:
    - mountPath: /etc/kubernetes/audit-policy.yaml
      name: audit-policy
      readOnly: true
    - mountPath: /var/log/kubernetes/audit/
      name: audit-log
  volumes:
  - hostPath:
      path: /etc/kubernetes/audit-policy.yaml
      type: File
    name: audit-policy
  - hostPath:
      path: /var/log/kubernetes/audit/
      type: DirectoryOrCreate
    name: audit-log
```

### 2. Fix Webhook Configuration

If using webhook backends:

```bash
# Create webhook config file
cat > webhook-config.yaml <<EOF
apiVersion: v1
kind: Config
clusters:
- name: audit-webhook
  cluster:
    server: https://audit-webhook.example.com/audit
    certificate-authority: /path/to/ca.crt
users:
- name: audit-webhook-client
  user:
    client-certificate: /path/to/client.crt
    client-key: /path/to/client.key
current-context: webhook
contexts:
- context:
    cluster: audit-webhook
    user: audit-webhook-client
  name: webhook
EOF

sudo cp webhook-config.yaml /etc/kubernetes/webhook-config.yaml
sudo chmod 644 /etc/kubernetes/webhook-config.yaml

# Update API server configuration
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

Add webhook configuration:
```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
    - --audit-webhook-config-file=/etc/kubernetes/webhook-config.yaml
    - --audit-webhook-initial-backoff=5s
    volumeMounts:
    - mountPath: /etc/kubernetes/webhook-config.yaml
      name: webhook-config
      readOnly: true
  volumes:
  - hostPath:
      path: /etc/kubernetes/webhook-config.yaml
      type: File
    name: webhook-config
```

### 3. Fix Log Rotation

If audit logs are not being rotated properly:

```bash
# Update API server configuration with rotation parameters
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

Add or update rotation parameters:
```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit/audit.log
    - --audit-log-maxage=30        # Maximum number of days to retain old files
    - --audit-log-maxbackup=10     # Maximum number of backup files to keep
    - --audit-log-maxsize=100      # Maximum size in MB before rotation
```

### 4. Optimize Audit Policy for Performance

If audit logging is causing performance issues:

```bash
# Create a more selective audit policy
cat > optimized-audit-policy.yaml <<EOF
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Don't log read-only actions on common resources to reduce volume
  - level: None
    verbs: ["get", "list", "watch"]
    resources:
    - group: ""
      resources: ["endpoints", "services", "configmaps"]
  
  # Log changes at RequestResponse level
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete"]
    resources:
    - group: ""
  
  # Log authentication at Metadata level
  - level: Metadata
    users: ["system:anonymous"]
    verbs: ["authenticate"]
  
  # Don't log requests to certain non-resource endpoints
  - level: None
    nonResourceURLs:
      - "/healthz*"
      - "/version"
      - "/swagger*"
      - "/metrics"
      - "/apis/metrics.k8s.io*"
  
  # Default catch-all rule
  - level: Metadata
EOF

sudo cp optimized-audit-policy.yaml /etc/kubernetes/audit-policy.yaml
```

### 5. Fix Storage Issues

If there are storage problems for audit logs:

```bash
# Create dedicated storage for audit logs
sudo mkdir -p /var/log/kubernetes/audit/
sudo chmod 700 /var/log/kubernetes/audit/

# If needed, move logs to a larger volume
sudo mount /dev/large-volume /var/log/kubernetes/audit/
```

For cloud environments, configure persistent volumes for audit logs:

```bash
# Create PV and PVC for audit logs
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: audit-logs-pvc
  namespace: kube-system
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: standard
EOF
```

Then update API server configuration to use this PVC.

### 6. Fix Webhook Connectivity Issues

If webhooks can't connect:

```bash
# Test webhook connectivity
curl -v --cert /path/to/client.crt --key /path/to/client.key --cacert /path/to/ca.crt https://audit-webhook.example.com/audit

# Update webhook configuration with correct TLS certificates
sudo vi /etc/kubernetes/webhook-config.yaml
```

For unreliable webhooks, use file logging as a backup:

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit/audit.log
    - --audit-webhook-config-file=/etc/kubernetes/webhook-config.yaml
    - --audit-webhook-mode=batch      # Options: batch or blocking
    - --audit-webhook-batch-max-size=100
    - --audit-webhook-batch-max-wait=1s
```

### 7. Fix Parsing Issues

If audit logs are not in the expected format:

```bash
# Check log format
sudo head -n 1 /var/log/kubernetes/audit/audit.log | jq .

# Ensure proper format in API server configuration
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

Add or check format parameters:
```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --audit-log-format=json
```

### 8. Configure for Specific Events

If specific events are missing:

```bash
# Update audit policy to capture specific events
cat > specific-audit-policy.yaml <<EOF
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log specific resource changes in detail
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete"]
    resources:
    - group: ""
      resources: ["secrets", "serviceaccounts", "roles", "rolebindings"]
    - group: "rbac.authorization.k8s.io"
      resources: ["*"]
  
  # Add other rules as needed
EOF

sudo cp specific-audit-policy.yaml /etc/kubernetes/audit-policy.yaml
```

## Prevention

1. **Plan Audit Strategy**: Design audit policy based on security requirements and performance constraints
2. **Dedicated Storage**: Use dedicated storage for audit logs
3. **Log Rotation**: Configure proper log rotation and retention
4. **Performance Testing**: Test audit policies for performance impact
5. **Selective Auditing**: Use selective auditing to reduce volume and impact
6. **Redundant Logging**: Configure both file and webhook backends for redundancy
7. **Log Aggregation**: Implement log aggregation for long-term storage and analysis
8. **Regular Review**: Regularly review audit policies and logs
9. **Monitoring**: Monitor audit log size and API server performance
10. **Documentation**: Document audit configuration and expected behavior

## Related Runbooks

* [API Server Unavailable](../cluster/api-server-unavailable.md)
* [Certificate Issues](../cluster/certificate-issues.md)
* [RBAC Permission Problems](./rbac-permission-problems.md)
* [Admission Controller Issues](../cluster/admission-controller-issues.md)
* [Storage Capacity Issues](../storage/storage-capacity-issues.md)