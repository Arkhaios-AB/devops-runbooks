# Troubleshooting Image Pull Secret Issues

## Symptoms

* Pods stuck in `ImagePullBackOff` or `ErrImagePull` state
* Error messages containing "unauthorized" when pulling images
* Events showing authentication failures for private registries
* Deployment rollouts failing due to image pull errors
* Error logs mentioning "401 Unauthorized" or "403 Forbidden"
* Container registry authorization issues across multiple pods
* Intermittent image pull failures

## Possible Causes

1. **Missing Image Pull Secret**: No imagePullSecrets specified in pod or service account
2. **Invalid Credentials**: Incorrect username or password in the secret
3. **Expired Credentials**: Authentication token or password has expired
4. **Registry URL Mismatch**: Secret configured for wrong registry URL
5. **Secret in Wrong Namespace**: Secret exists but in a different namespace
6. **Secret Format Issues**: Incorrectly formatted or encoded registry credentials
7. **Registry Permission Issues**: User has insufficient permissions in the registry
8. **Service Account Configuration**: Service account not configured with pull secret
9. **Rate Limiting**: Registry rate limits exceeded (e.g., Docker Hub)
10. **Network Issues**: Network connectivity to the registry is blocked

## Diagnosis Steps

### 1. Check Pod Status and Events

```bash
# Check pod status
kubectl get pod <pod-name> -n <namespace>

# Check pod events for image pull errors
kubectl describe pod <pod-name> -n <namespace>
```

### 2. Verify Image Pull Secret Exists

```bash
# List secrets in the namespace
kubectl get secrets -n <namespace> | grep regcred

# Examine image pull secret content
kubectl get secret <secret-name> -n <namespace> -o yaml
```

### 3. Check Secret Format and Credentials

```bash
# Decode Docker config from secret
kubectl get secret <secret-name> -n <namespace> -o jsonpath='{.data.\.dockerconfigjson}' | base64 --decode

# Check if decoded content has proper format:
# {"auths":{"registry.example.com":{"username":"user","password":"pass","auth":"base64encoded"}}}
```

### 4. Check Pod Configuration

```bash
# Check if pod is using imagePullSecrets
kubectl get pod <pod-name> -n <namespace> -o yaml | grep imagePullSecrets -A 3

# Check the image reference
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[*].image}'
```

### 5. Check Service Account Configuration

```bash
# Get service account name used by the pod
SA=$(kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.serviceAccountName}')

# Check if service account has image pull secrets
kubectl get serviceaccount $SA -n <namespace> -o yaml | grep imagePullSecrets -A 10
```

### 6. Test Registry Authentication

```bash
# Create a test pod to manually test authentication
kubectl run auth-test --image=busybox --rm -it -- sh

# In the test pod, try manual authentication
# /bin/sh -c 'echo $AUTH_TOKEN | docker login registry.example.com --username <username> --password-stdin'
```

## Resolution Steps

### 1. Create or Fix Image Pull Secret

If the secret is missing or invalid:

```bash
# Create a new image pull secret
kubectl create secret docker-registry <secret-name> \
  --docker-server=<registry-server> \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=<email> \
  -n <namespace>
```

### 2. Add Secret to Pod

If the pod isn't configured to use the secret:

```bash
# Edit the deployment to add imagePullSecrets
kubectl edit deployment <deployment-name> -n <namespace>
```

Add the following to the pod template spec:
```yaml
imagePullSecrets:
- name: <secret-name>
```

### 3. Add Secret to Service Account

To make the secret available to all pods using a service account:

```bash
# Add the image pull secret to the service account
kubectl patch serviceaccount <service-account> -n <namespace> \
  -p '{"imagePullSecrets": [{"name": "<secret-name>"}]}'
```

### 4. Fix Expired Credentials

If credentials have expired:

```bash
# Update the secret with renewed credentials
kubectl delete secret <secret-name> -n <namespace>
kubectl create secret docker-registry <secret-name> \
  --docker-server=<registry-server> \
  --docker-username=<username> \
  --docker-password=<new-password> \
  --docker-email=<email> \
  -n <namespace>
```

### 5. Fix Registry URL

If using the wrong registry URL:

```bash
# Check the current image URL
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[*].image}'

# Update the deployment with the correct image URL
kubectl set image deployment/<deployment-name> <container-name>=<correct-registry-url>/<image>:<tag> -n <namespace>

# Update the secret with the correct registry URL
kubectl delete secret <secret-name> -n <namespace>
kubectl create secret docker-registry <secret-name> \
  --docker-server=<correct-registry-server> \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=<email> \
  -n <namespace>
```

### 6. Fix Registry Permissions

If the user has insufficient permissions in the registry:

1. Log in to the registry management console
2. Ensure the user has appropriate permissions for the repository
3. For Docker Hub, ensure user has access to the organization/repository
4. For ECR, check IAM permissions
5. For GCR, check GCP IAM permissions

### 7. Address Rate Limiting

If hitting Docker Hub rate limits:

```bash
# Create authenticated pull secret for Docker Hub
kubectl create secret docker-registry dockerhub \
  --docker-server=docker.io \
  --docker-username=<paid-account-username> \
  --docker-password=<password> \
  --docker-email=<email> \
  -n <namespace>

# Add the secret to all service accounts
kubectl patch serviceaccount default -n <namespace> \
  -p '{"imagePullSecrets": [{"name": "dockerhub"}]}'
```

Consider alternatives:
- Use a paid Docker Hub account
- Use a private registry or mirror
- Use public images from alternative registries

### 8. Copy Secret Between Namespaces

If the secret exists in another namespace:

```bash
# Get the secret data
kubectl get secret <secret-name> -n <source-namespace> -o yaml | grep -v '^\s*namespace:\s' > secret.yaml

# Edit the YAML to change or remove the namespace
# Then apply to the target namespace
kubectl apply -f secret.yaml -n <target-namespace>
```

### 9. Fix Network Connectivity

If network connectivity to the registry is blocked:

```bash
# Test network connectivity
kubectl run network-test --image=busybox --rm -it -- sh -c "ping <registry-server> && telnet <registry-server> 443"
```

Solutions may include:
- Configure proxy settings if necessary
- Update network policies to allow egress to the registry
- Configure firewall rules to allow access to the registry

### 10. Use Local Registry Cache

For frequent connectivity issues, set up a local registry mirror:

1. Deploy a registry mirror in the cluster
2. Configure nodes to use the local mirror
3. Populate the mirror with frequently used images

## Prevention

1. **Standardize Registry Access**: Use consistent auth methods across environments
2. **Automate Secret Management**: Use tools like External Secrets Operator or Vault
3. **Credential Rotation**: Regularly rotate registry credentials
4. **Service Account Configuration**: Attach pull secrets to service accounts instead of individual pods
5. **Registry Mirroring**: Set up registry mirrors to reduce external dependencies
6. **Image Caching**: Implement image caching to reduce pull frequency
7. **Pre-pull Images**: Pre-pull common images to nodes
8. **Monitoring**: Monitor image pull errors and authentication failures
9. **Documentation**: Document registry access procedures
10. **CI/CD Integration**: Test registry auth as part of CI/CD pipelines

## Related Runbooks

* [Pod ImagePullBackOff](../workloads/pod-imagepullbackoff.md)
* [Pod Stuck in ContainerCreating](../workloads/pod-container-creating.md)
* [Deployment Rollout Issues](../workloads/deployment-rollout-issues.md)
* [ServiceAccount Issues](./service-account-issues.md)
* [Secret and ConfigMap Issues](./secret-configmap-issues.md)
* [Network Policy Issues](../networking/network-policy-issues.md)