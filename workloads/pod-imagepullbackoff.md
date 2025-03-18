# Troubleshooting Pod ImagePullBackOff

## Symptoms

* Pods stuck in `ImagePullBackOff` or `ErrImagePull` state
* `kubectl get pods` shows `ImagePullBackOff` status
* Events showing "Failed to pull image" errors
* Container registry authentication errors in events
* Pod creation stalls due to image pull issues
* Deployments not reaching desired replica count

## Possible Causes

1. **Image Does Not Exist**: Container image doesn't exist in the specified registry
2. **Invalid Image Tag**: Tag specified doesn't exist or is misspelled
3. **Registry Authentication Issues**: Missing or invalid image pull secrets
4. **Registry Rate Limiting**: Hitting registry rate limits (e.g., Docker Hub)
5. **Network Connectivity**: Node cannot reach container registry
6. **Registry Availability**: Container registry is down or having issues
7. **Image Pull Policy**: Misconfigured imagePullPolicy
8. **Private Registry Setup**: Improperly configured private registry
9. **Node Disk Pressure**: Node out of disk space for new images
10. **Container Runtime Issues**: Problems with Docker, containerd, or CRI-O

## Diagnosis Steps

### 1. Check Pod Status and Events

```bash
# Get pod status
kubectl get pod <pod-name> -n <namespace>

# Check pod details
kubectl describe pod <pod-name> -n <namespace>

# Look at events
kubectl get events -n <namespace> | grep <pod-name>
```

### 2. Verify Image Name and Tag

```bash
# Extract image from pod spec
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[0].image}'

# For Docker Hub, check if image exists
docker pull <image-name>:<tag>

# For other registries, use appropriate commands or UI
```

### 3. Check Registry Authentication

```bash
# Check if image pull secrets are specified
kubectl get pod <pod-name> -n <namespace> -o yaml | grep imagePullSecrets -A 3

# List secrets in namespace
kubectl get secrets -n <namespace> | grep regcred

# Examine image pull secret contents
kubectl get secret <image-pull-secret> -n <namespace> -o jsonpath='{.data.\.dockerconfigjson}' | base64 --decode
```

### 4. Check Node Network Connectivity

SSH to the node and check connectivity:

```bash
# Check DNS resolution
nslookup docker.io

# Check connectivity to registry
curl -v https://registry-1.docker.io/v2/

# Check proxy settings if using proxy
env | grep -i proxy
```

### 5. Check Container Runtime Status

SSH to the node and check container runtime:

```bash
# For Docker
sudo systemctl status docker
sudo docker info

# For containerd
sudo systemctl status containerd
sudo crictl info

# For CRI-O
sudo systemctl status crio
sudo crictl info
```

### 6. Check Node Disk Space

```bash
# Check disk space on nodes
df -h

# Check space for container runtime
du -sh /var/lib/docker    # For Docker
du -sh /var/lib/containerd  # For containerd
du -sh /var/lib/containers  # For CRI-O
```

## Resolution Steps

### 1. Fix Invalid Image Name or Tag

If image or tag doesn't exist:

```bash
# Edit deployment to fix image name or tag
kubectl edit deployment <deployment-name> -n <namespace>
```

Example correct image reference:
```yaml
containers:
- name: myapp
  image: nginx:1.21.0  # Specific, existing tag
```

### 2. Fix Registry Authentication

If authentication is failing:

```bash
# Create correct image pull secret
kubectl create secret docker-registry regcred \
  --docker-server=<your-registry-server> \
  --docker-username=<your-username> \
  --docker-password=<your-password> \
  --docker-email=<your-email> \
  -n <namespace>

# Add pull secret to service account
kubectl patch serviceaccount default -n <namespace> \
  -p '{"imagePullSecrets": [{"name": "regcred"}]}'

# Or update pod/deployment to use the secret
kubectl edit deployment <deployment-name> -n <namespace>
```

Example pull secret in pod spec:
```yaml
imagePullSecrets:
- name: regcred
```

### 3. Fix Registry Rate Limiting

If hitting Docker Hub rate limits:

```bash
# Create authenticated pull secret for Docker Hub
kubectl create secret docker-registry dockerhub \
  --docker-server=docker.io \
  --docker-username=<your-username> \
  --docker-password=<your-password> \
  --docker-email=<your-email> \
  -n <namespace>

# Use authenticated pulls
kubectl patch serviceaccount default -n <namespace> \
  -p '{"imagePullSecrets": [{"name": "dockerhub"}]}'
```

### 4. Fix Network Connectivity

If nodes can't reach registry:

```bash
# Configure proxy if needed
kubectl create configmap proxy-config \
  --from-literal=HTTP_PROXY=http://proxy:port \
  --from-literal=HTTPS_PROXY=http://proxy:port \
  --from-literal=NO_PROXY=localhost,127.0.0.1,10.0.0.0/8

# Update container runtime to use proxy
# This varies by container runtime and node setup
```

### 5. Pre-pull Images

If network issues are intermittent:

```bash
# Pre-pull images on nodes
kubectl create daemonset pull-images --image=busybox -n kube-system -- \
  /bin/sh -c "docker pull <image1>; docker pull <image2>; sleep infinity"
```

### 6. Fix Container Runtime Issues

SSH to the node and:

```bash
# Restart container runtime
# For Docker
sudo systemctl restart docker

# For containerd
sudo systemctl restart containerd

# For CRI-O
sudo systemctl restart crio
```

### 7. Clean Up Disk Space

If nodes are out of disk space:

```bash
# Remove unused images
sudo crictl rmi --prune

# Remove unused containers
sudo crictl rm $(sudo crictl ps -a -q --state exited)
```

### 8. Use Image Digest Instead of Tag

For critical applications, use image digest for immutability:

```yaml
containers:
- name: myapp
  image: nginx@sha256:9b1e1e7164db75ad0f64e8deeb33e771d455fa590126b2e16d25e5a75fc6f517
```

### 9. Update Image Pull Policy

Adjust image pull policy if needed:

```yaml
containers:
- name: myapp
  image: nginx:1.21.0
  imagePullPolicy: IfNotPresent  # Only pull if not present
```

## Prevention

1. **Use Trusted Registries**: Use reliable container registries
2. **Image Tags**: Avoid using "latest" tag, use specific versions
3. **Registry Authentication**: Properly configure registry authentication
4. **Private Registry**: Consider using a private registry or registry mirror
5. **Resource Planning**: Ensure nodes have adequate disk space
6. **Network Setup**: Ensure proper network setup for registry access
7. **Image Pull Secrets**: Properly manage image pull secrets
8. **Image Scanning**: Implement image scanning and validation
9. **Image Caching**: Implement image caching where appropriate
10. **Documentation**: Document registry access and credentials management

## Related Runbooks

* [Node Disk Pressure](../cluster/node-disk-pressure.md)
* [Pod Stuck in ContainerCreating](./pod-container-creating.md)
* [Deployment Rollout Issues](./deployment-rollout-issues.md)
* [Image Pull Secret Issues](../security/image-pull-secret-issues.md)
* [Container Runtime Issues](../cluster/kubelet-issues.md)