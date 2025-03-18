# Troubleshooting Admission Controller Issues

## Symptoms

* Resource creation or update failures with error messages mentioning "admission webhook"
* Error messages containing phrases like "denied the request" or "admission webhook ... denied the request"
* Unexpected modifications to resources during creation or update
* High latency when creating or updating resources
* API server logs showing admission controller errors
* Webhook timeout errors
* Resources being rejected even when they appear to meet all requirements

## Possible Causes

1. **Misconfigured Webhook**: Incorrectly configured admission webhook
2. **Webhook Unavailability**: Admission webhook service is down
3. **Certificate Issues**: TLS certificate problems for webhook communication
4. **Network Problems**: Network connectivity issues between API server and webhook
5. **Webhook Performance**: Slow webhook response causing timeouts
6. **Incompatible Validations**: Multiple admission controllers with conflicting requirements
7. **Missing Exemptions**: Missing namespace or resource exemptions for system components
8. **Configuration Changes**: Recent changes to admission controller configuration
9. **Version Incompatibility**: Webhook not compatible with current Kubernetes version
10. **Resource Constraints**: Webhook pods having resource issues

## Diagnosis Steps

### 1. Identify the Failing Admission Controller

```bash
# Attempt to create a resource that fails and note the error message
kubectl apply -f my-resource.yaml

# The error message should indicate which webhook is failing
# Example: "admission webhook "policy.example.com" denied the request"
```

### 2. Check Admission Webhook Configurations

```bash
# List all ValidatingWebhookConfigurations
kubectl get validatingwebhookconfigurations

# List all MutatingWebhookConfigurations
kubectl get mutatingwebhookconfigurations

# Get details of a specific webhook configuration
kubectl describe validatingwebhookconfigurations <webhook-name>
kubectl describe mutatingwebhookconfigurations <webhook-name>
```

### 3. Check Webhook Service Availability

```bash
# Check if the webhook service is running
kubectl get pods -n <webhook-namespace> | grep <webhook-service>

# Check webhook service logs
kubectl logs -n <webhook-namespace> <webhook-pod-name>
```

### 4. Check API Server Logs

```bash
# Check API server logs for admission errors
kubectl logs -n kube-system -l component=kube-apiserver | grep admission

# Look for specific webhook names in the logs
kubectl logs -n kube-system -l component=kube-apiserver | grep <webhook-name>
```

### 5. Test Webhook Connectivity

```bash
# Create a test pod in the webhook's namespace to verify connectivity
kubectl run -n <webhook-namespace> --rm -it test-curl --image=curlimages/curl -- curl -k https://<webhook-service>.<webhook-namespace>.svc:443/
```

### 6. Check TLS Certificate

```bash
# Check if CA bundle is correct in webhook configuration
kubectl get validatingwebhookconfigurations <webhook-name> -o yaml | grep caBundle
```

## Resolution Steps

### 1. Temporarily Disable Problematic Webhook

If you need immediate relief while investigating:

```bash
# Patch the webhook configuration to disable it temporarily
kubectl patch validatingwebhookconfiguration <webhook-name> \
  --type json \
  -p='[{"op": "add", "path": "/webhooks/0/failurePolicy", "value":"Ignore"}]'
```

### 2. Fix Webhook Service

If the webhook service is down:

```bash
# Restart webhook deployment
kubectl rollout restart deployment -n <webhook-namespace> <webhook-deployment>

# Scale up if needed
kubectl scale deployment -n <webhook-namespace> <webhook-deployment> --replicas=2
```

### 3. Fix Certificate Issues

If TLS certificates are causing problems:

```bash
# Get the CA bundle from the Kubernetes API server
CA_BUNDLE=$(kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.certificate-authority-data}')

# Update the caBundle field in the webhook configuration
kubectl patch validatingwebhookconfiguration <webhook-name> \
  --type='json' -p="[{'op': 'replace', 'path': '/webhooks/0/clientConfig/caBundle', 'value':'${CA_BUNDLE}'}]"
```

### 4. Fix Webhook Configuration

If the webhook configuration is incorrect:

```bash
# Edit the webhook configuration
kubectl edit validatingwebhookconfiguration <webhook-name>
```

Key areas to check and fix:
- `clientConfig.url` or `clientConfig.service` is correct
- `rules` are properly defined for target resources
- `namespaceSelector` and `objectSelector` are configured correctly
- `failurePolicy` is appropriate (Ignore or Fail)
- `timeoutSeconds` is set to a reasonable value (like 10 seconds)

### 5. Add Namespace or Object Exemptions

If system resources are being blocked:

```yaml
# Add namespace selector to exempt kube-system
namespaceSelector:
  matchExpressions:
  - key: kubernetes.io/metadata.name
    operator: NotIn
    values: ["kube-system"]
```

### 6. Adjust Webhook Timeout

If webhook is timing out:

```bash
# Patch webhook configuration to increase timeout
kubectl patch validatingwebhookconfiguration <webhook-name> \
  --type='json' -p='[{"op": "replace", "path": "/webhooks/0/timeoutSeconds", "value":10}]'
```

### 7. Scale Webhook Resources

If webhook is resource-constrained:

```bash
# Edit the webhook deployment to increase resources
kubectl edit deployment -n <webhook-namespace> <webhook-deployment>
```

Example resource adjustment:
```yaml
resources:
  limits:
    cpu: "500m"
    memory: "512Mi"
  requests:
    cpu: "200m"
    memory: "256Mi"
```

### 8. Upgrade or Downgrade Webhook

If the webhook is incompatible with the current Kubernetes version:
- Check the webhook's documentation for compatibility
- Upgrade webhook to a version compatible with your Kubernetes version
- Consider temporarily disabling the webhook until it can be upgraded

## Prevention

1. **Testing**: Test admission controllers in non-production before deployment
2. **Monitoring**: Monitor webhook availability and performance
3. **Gradual Rollout**: Roll out new admission controllers to a subset of namespaces first
4. **Documentation**: Document all admission controllers and their configurations
5. **HA Setup**: Run critical webhooks with high availability
6. **Appropriate Timeouts**: Configure appropriate timeouts based on webhook complexity
7. **Resource Planning**: Allocate sufficient CPU and memory resources for webhook pods
8. **Version Compatibility**: Ensure webhooks are compatible with your Kubernetes version
9. **Exemption Strategy**: Plan and implement proper exemption strategy for system namespaces
10. **Failure Policy**: Use appropriate failurePolicy settings based on webhook criticality

## Related Runbooks

* [API Server Unavailable](./api-server-unavailable.md)
* [Certificate Issues](./certificate-issues.md)
* [Service Not Accessible](../networking/service-not-accessible.md)
* [Deployment Rollout Issues](../workloads/deployment-rollout-issues.md)
* [Pod Security Policy Issues](../security/pod-security-policy-issues.md)