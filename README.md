# Kubernetes Troubleshooting Runbooks

A comprehensive collection of DevOps runbooks for troubleshooting common Kubernetes issues. Use these guides to quickly diagnose and resolve problems in your Kubernetes clusters.

## Issue Index

| Category | Issue | Description | Key Symptoms |
|----------|-------|-------------|-------------|
| **Cluster and Node** | [Node Not Ready](./cluster/node-not-ready.md) | Troubleshoot nodes stuck in NotReady state | • Node status shows `NotReady`<br>• Pods on node may be stuck or evicted<br>• New pods not scheduled on node |
| **Cluster and Node** | [Node Memory Pressure](./cluster/node-memory-pressure.md) | Address nodes experiencing memory pressure | • Node shows `MemoryPressure=True`<br>• Pod evictions occurring<br>• OOM events in logs |
| **Cluster and Node** | [Node CPU Pressure](./cluster/node-cpu-pressure.md) | Resolve high CPU utilization on nodes | • Node shows sustained high CPU usage<br>• Slow pod operations<br>• System processes unresponsive |
| **Cluster and Node** | [Node Disk Pressure](./cluster/node-disk-pressure.md) | Fix disk space issues on nodes | • Node shows `DiskPressure=True`<br>• Pod evictions occurring<br>• Low disk space alerts |
| **Cluster and Node** | [Cluster Autoscaling Issues](./cluster/autoscaling-issues.md) | Debug problems with cluster autoscaling | • Cluster fails to scale up/down<br>• Pending pods despite autoscaler enabled<br>• Unexpected node additions/removals |
| **Cluster and Node** | [etcd Issues](./cluster/etcd-issues.md) | Troubleshoot problems with etcd | • API server operations slow or timing out<br>• Kubernetes API errors<br>• Failed etcd health checks |
| **Cluster and Node** | [Kubelet Issues](./cluster/kubelet-issues.md) | Resolve kubelet problems | • Kubelet service failures<br>• Node reported as NotReady<br>• Pod operations failing |
| **Cluster and Node** | [API Server Unavailable](./cluster/api-server-unavailable.md) | Fix API server availability problems | • kubectl commands fail with connection errors<br>• 503 Service Unavailable errors<br>• Control plane components cannot communicate |
| **Cluster and Node** | [Certificate Issues](./cluster/certificate-issues.md) | Troubleshoot certificate problems | • Certificate validation errors<br>• x509 errors in logs<br>• Services unable to authenticate |
| **Cluster and Node** | [Admission Controller Issues](./cluster/admission-controller-issues.md) | Resolve admission controller failures | • Resource creation rejected<br>• Webhook timeout errors<br>• Pod scheduling failures with admission errors |
| **Cluster and Node** | [kube-proxy Issues](./cluster/kube-proxy-issues.md) | Fix service networking problems | • Service connectivity issues<br>• NodePort not accessible<br>• iptables/IPVS rules missing |
| **Workloads** | [Pod Stuck in Pending](./workloads/pod-pending.md) | Debug pods stuck in pending state | • Pods remain in Pending state<br>• Events show resource constraints<br>• Node selection issues |
| **Workloads** | [Pod Stuck in ContainerCreating](./workloads/pod-container-creating.md) | Resolve container creation issues | • Pods stuck in ContainerCreating state<br>• Image pull issues<br>• Volume mounting problems |
| **Workloads** | [Pod Stuck in Terminating](./workloads/pod-terminating.md) | Fix pods stuck in terminating state | • Pods stuck in Terminating state<br>• Force deletion required<br>• Finalizer issues |
| **Workloads** | [Pod CrashLoopBackOff](./workloads/pod-crashloopbackoff.md) | Debug containers repeatedly crashing | • Pods showing CrashLoopBackOff status<br>• Container exit codes indicating failures<br>• Rapidly restarting containers |
| **Workloads** | [Pod OOMKilled](./workloads/pod-oomkilled.md) | Resolve out-of-memory issues | • Containers terminated with OOMKilled reason<br>• Pods restarting unexpectedly<br>• Memory limit exceeded messages |
| **Workloads** | [Pod ImagePullBackOff](./workloads/pod-imagepullbackoff.md) | Fix image pulling problems | • ImagePullBackOff status<br>• Repository access errors<br>• Image not found errors |
| **Workloads** | [Pod Init Container Errors](./workloads/pod-init-container-errors.md) | Debug init container issues | • Pod stuck with init containers<br>• Init container failures<br>• Init logic errors |
| **Workloads** | [Deployment Rollout Issues](./workloads/deployment-rollout-issues.md) | Address deployment rollout failures | • Deployment rollouts stalled<br>• Deployments not progressing<br>• ReplicaSet creation failures |
| **Workloads** | [StatefulSet Issues](./workloads/statefulset-issues.md) | Troubleshoot problems with StatefulSets | • StatefulSet pods not deploying in order<br>• Persistent volume claim issues<br>• Headless service problems |
| **Workloads** | [DaemonSet Issues](./workloads/daemonset-issues.md) | Debug issues with DaemonSets | • DaemonSet pods not running on all nodes<br>• Node selector or affinity issues<br>• DaemonSet updates failing |
| **Networking** | [Service Not Accessible](./networking/service-not-accessible.md) | Debug service connectivity problems | • Services unreachable within cluster<br>• Endpoint connection failures<br>• Service selector issues |
| **Networking** | [Ingress Issues](./networking/ingress-issues.md) | Resolve problems with Ingress resources | • Ingress routes not working<br>• 404/503 errors for Ingress paths<br>• TLS certificate issues |
| **Networking** | [DNS Resolution Problems](./networking/dns-resolution-problems.md) | Troubleshoot DNS issues in the cluster | • Service name resolution failures<br>• Intermittent DNS timeouts<br>• CoreDNS pods issues |
| **Networking** | [Network Policy Issues](./networking/network-policy-issues.md) | Debug Network Policy configuration | • Unexpected connection denials<br>• Network policies not enforced<br>• Blocked service communication |
| **Networking** | [LoadBalancer Service Issues](./networking/loadbalancer-issues.md) | Resolve problems with LoadBalancer services | • External IP not assigned<br>• Cloud load balancer not provisioned<br>• Health check failures |
| **Networking** | [NodePort Service Issues](./networking/nodeport-issues.md) | Fix NodePort service problems | • Unable to connect to NodePort<br>• Connection timeouts<br>• NodePort service unreachable from outside |
| **Networking** | [CNI Plugin Issues](./networking/cni-plugin-issues.md) | Troubleshoot Container Network Interface problems | • Pod network connectivity issues<br>• IP allocation failures<br>• CNI configuration errors |
| **Networking** | [Service Mesh Issues](./networking/service-mesh-issues.md) | Debug issues with service mesh implementations | • Service-to-service communication failures<br>• High latency<br>• TLS handshake failures |
| **Networking** | [Pod-to-Pod Communication Issues](./networking/pod-to-pod-communication.md) | Resolve pod networking problems | • Pods unable to communicate<br>• Network timeouts between pods<br>• Cross-node networking fails |
| **Networking** | [CoreDNS Issues](./networking/coredns-issues.md) | Troubleshoot problems with CoreDNS | • DNS resolution failures<br>• Slow DNS queries<br>• CoreDNS pods crashlooping |
| **Storage** | [PVC Stuck in Pending](./storage/pvc-pending.md) | Debug persistent volume claim issues | • PVCs remain in Pending state<br>• Volume provisioning failures<br>• No available persistent volumes |
| **Storage** | [Storage Class Issues](./storage/storage-class-issues.md) | Resolve problems with storage classes | • StorageClass not working<br>• Volume provisioning errors<br>• Default StorageClass conflicts |
| **Storage** | [Volume Mount Problems](./storage/volume-mount-problems.md) | Fix issues with volume mounts | • Volume mount failures<br>• Permission denied errors<br>• Wrong filesystem type |
| **Storage** | [CSI Driver Issues](./storage/csi-driver-issues.md) | Troubleshoot Container Storage Interface problems | • Volume provisioning failures<br>• CSI driver errors<br>• Volume attachment issues |
| **Storage** | [StatefulSet Storage Issues](./storage/statefulset-storage-issues.md) | Debug storage for StatefulSets | • PVC creation failures for StatefulSets<br>• Volume attachment errors<br>• StatefulSet pod scheduling problems |
| **Storage** | [PV Reclaim Policy Issues](./storage/pv-reclaim-policy-issues.md) | Address persistent volume reclamation problems | • PVs not released after PVC deletion<br>• Storage resources orphaned<br>• Reclaim policy configuration errors |
| **Storage** | [Dynamic Provisioning Issues](./storage/dynamic-provisioning-issues.md) | Troubleshoot dynamic volume provisioning | • PVCs failing with provisioning errors<br>• StorageClass not creating volumes<br>• CSI driver errors |
| **Storage** | [Storage Capacity Issues](./storage/storage-capacity-issues.md) | Resolve storage capacity problems | • PVCs failing with "no space left" errors<br>• Storage expansion failures<br>• Cloud provider quota errors |
| **Storage** | [Storage Performance Issues](./storage/storage-performance-issues.md) | Debug slow storage performance | • Slow application response times<br>• High I/O wait<br>• Timeout errors in storage operations |
| **Storage** | [Cloud Storage Issues](./storage/cloud-storage-issues.md) | Troubleshoot cloud provider storage | • PVCs stuck in pending<br>• Volume attachment failures<br>• Slow cloud storage I/O performance |
| **Resources** | [Resource Quota Issues](./resources/resource-quota-issues.md) | Debug resource quota problems | • Pod creation rejected due to quota<br>• Namespace resource limits reached<br>• Resource quota calculation issues |
| **Resources** | [LimitRange Issues](./resources/limit-range-issues.md) | Resolve limit range configuration issues | • Container creation failed due to LimitRange<br>• Default limits not applied<br>• LimitRange conflicts |
| **Resources** | [HPA Issues](./resources/hpa-issues.md) | Troubleshoot Horizontal Pod Autoscaler problems | • HPA not scaling pods<br>• Unexpected scaling behavior<br>• Metrics unavailable for HPA |
| **Resources** | [VPA Issues](./resources/vpa-issues.md) | Fix Vertical Pod Autoscaler issues | • VPA not recommending resources<br>• VPA recommendations not applied<br>• Conflicts with HPA |
| **Security** | [RBAC Permission Problems](./security/rbac-permission-problems.md) | Debug RBAC permissions | • Permission denied errors<br>• Unauthorized API access<br>• ServiceAccount permission issues |
| **Security** | [Secret and ConfigMap Issues](./security/secret-configmap-issues.md) | Resolve issues with Secrets and ConfigMaps | • Secrets not accessible by pods<br>• ConfigMap data not visible<br>• Volume mounting issues |
| **Security** | [ServiceAccount Issues](./security/service-account-issues.md) | Troubleshoot service account problems | • Default ServiceAccount not working<br>• Token not mounted in pods<br>• Authentication failures |
| **Security** | [Pod Security Policy Issues](./security/pod-security-policy-issues.md) | Debug PSP/PSA configurations | • Pods rejected by security policies<br>• Security context validation failures<br>• PSP migration issues |
| **Security** | [Image Pull Secret Issues](./security/image-pull-secret-issues.md) | Fix private registry authentication | • ImagePullBackOff due to auth failures<br>• Private registry access issues<br>• Secret configuration problems |
| **Security** | [Audit Logging Issues](./security/audit-logging-issues.md) | Troubleshoot audit logging problems | • Missing audit logs<br>• Incorrect audit policy<br>• Log storage and retention issues |
| **Security** | [Cluster Backup and Restore](./security/cluster-backup-restore.md) | Manage cluster data backup and recovery | • Need to recover from data loss<br>• etcd data corruption<br>• Failed cluster upgrades |

## How to Use These Runbooks

Each runbook follows a consistent format:
1. **Symptoms** - How to identify the issue
2. **Possible Causes** - Common reasons for the problem
3. **Diagnosis Steps** - Commands and procedures to identify the root cause
4. **Resolution Steps** - Step-by-step instructions to fix the issue
5. **Prevention** - How to prevent the issue from recurring
6. **Related Runbooks** - Links to other related troubleshooting guides

## Finding Relevant Runbooks

You can use the table above to find the appropriate runbook based on:
- **Category**: Choose between Cluster/Node, Workloads, Networking, Storage, Resources, or Security
- **Issue**: Specific problem you're encountering
- **Symptoms**: Observable indicators that match what you're experiencing

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request to add new runbooks or improve existing ones.