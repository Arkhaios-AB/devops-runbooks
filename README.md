# Kubernetes Troubleshooting Runbooks

A comprehensive collection of DevOps runbooks for troubleshooting common Kubernetes issues. Use these guides to quickly diagnose and resolve problems in your Kubernetes clusters.

## Cluster and Node Issues

1. [Node Not Ready](./cluster/node-not-ready.md) - Troubleshoot nodes stuck in NotReady state
2. [Node Memory Pressure](./cluster/node-memory-pressure.md) - Address nodes experiencing memory pressure
3. [Node CPU Pressure](./cluster/node-cpu-pressure.md) - Resolve high CPU utilization on nodes
4. [Node Disk Pressure](./cluster/node-disk-pressure.md) - Fix disk space issues on nodes
5. [Cluster Autoscaling Issues](./cluster/autoscaling-issues.md) - Debug problems with cluster autoscaling
6. [etcd Issues](./cluster/etcd-issues.md) - Troubleshoot problems with etcd
7. [Kubelet Issues](./cluster/kubelet-issues.md) - Resolve kubelet problems
8. [API Server Unavailable](./cluster/api-server-unavailable.md) - Fix API server availability problems
9. [Certificate Issues](./cluster/certificate-issues.md) - Troubleshoot certificate problems
10. [Admission Controller Issues](./cluster/admission-controller-issues.md) - Resolve admission controller failures

## Workload Issues

11. [Pod Stuck in Pending](./workloads/pod-pending.md) - Debug pods stuck in pending state
12. [Pod Stuck in ContainerCreating](./workloads/pod-container-creating.md) - Resolve container creation issues
13. [Pod Stuck in Terminating](./workloads/pod-terminating.md) - Fix pods stuck in terminating state
14. [Pod CrashLoopBackOff](./workloads/pod-crashloopbackoff.md) - Debug containers repeatedly crashing
15. [Pod OOMKilled](./workloads/pod-oomkilled.md) - Resolve out-of-memory issues
16. [Pod ImagePullBackOff](./workloads/pod-imagepullbackoff.md) - Fix image pulling problems
17. [Pod Init Container Errors](./workloads/pod-init-container-errors.md) - Debug init container issues
18. [Deployment Rollout Issues](./workloads/deployment-rollout-issues.md) - Address deployment rollout failures
19. [StatefulSet Issues](./workloads/statefulset-issues.md) - Troubleshoot problems with StatefulSets
20. [DaemonSet Issues](./workloads/daemonset-issues.md) - Debug issues with DaemonSets

## Networking Issues

21. [Service Not Accessible](./networking/service-not-accessible.md) - Debug service connectivity problems
22. [Ingress Issues](./networking/ingress-issues.md) - Resolve problems with Ingress resources
23. [DNS Resolution Problems](./networking/dns-resolution-problems.md) - Troubleshoot DNS issues in the cluster
24. [Network Policy Issues](./networking/network-policy-issues.md) - Debug Network Policy configuration
25. [LoadBalancer Service Issues](./networking/loadbalancer-issues.md) - Resolve problems with LoadBalancer services
26. [NodePort Service Issues](./networking/nodeport-issues.md) - Fix NodePort service problems
27. [CNI Plugin Issues](./networking/cni-plugin-issues.md) - Troubleshoot Container Network Interface problems
28. [Service Mesh Issues](./networking/service-mesh-issues.md) - Debug issues with service mesh implementations
29. [Pod-to-Pod Communication Issues](./networking/pod-to-pod-communication.md) - Resolve pod networking problems
30. [CoreDNS Issues](./networking/coredns-issues.md) - Troubleshoot problems with CoreDNS

## Storage Issues

31. [PVC Stuck in Pending](./storage/pvc-pending.md) - Debug persistent volume claim issues
32. [Storage Class Issues](./storage/storage-class-issues.md) - Resolve problems with storage classes
33. [Volume Mount Problems](./storage/volume-mount-problems.md) - Fix issues with volume mounts
34. [CSI Driver Issues](./storage/csi-driver-issues.md) - Troubleshoot Container Storage Interface problems
35. [StatefulSet Storage Issues](./storage/statefulset-storage-issues.md) - Debug storage for StatefulSets
36. [PV Reclaim Policy Issues](./storage/pv-reclaim-policy-issues.md) - Address persistent volume reclamation problems
37. [Dynamic Provisioning Issues](./storage/dynamic-provisioning-issues.md) - Troubleshoot dynamic volume provisioning
38. [Storage Capacity Issues](./storage/storage-capacity-issues.md) - Resolve storage capacity problems
39. [Storage Performance Issues](./storage/storage-performance-issues.md) - Debug slow storage performance
40. [EBS/EFS/Azure Disk Issues](./storage/cloud-storage-issues.md) - Troubleshoot cloud provider storage

## Resource and Security Issues

41. [Resource Quota Issues](./resources/resource-quota-issues.md) - Debug resource quota problems
42. [LimitRange Issues](./resources/limit-range-issues.md) - Resolve limit range configuration issues
43. [HPA Issues](./resources/hpa-issues.md) - Troubleshoot Horizontal Pod Autoscaler problems
44. [VPA Issues](./resources/vpa-issues.md) - Fix Vertical Pod Autoscaler issues
45. [RBAC Permission Problems](./security/rbac-permission-problems.md) - Debug RBAC permissions
46. [Secret and ConfigMap Issues](./security/secret-configmap-issues.md) - Resolve issues with Secrets and ConfigMaps
47. [ServiceAccount Issues](./security/service-account-issues.md) - Troubleshoot service account problems
48. [Pod Security Policy Issues](./security/pod-security-policy-issues.md) - Debug PSP/PSA configurations
49. [Image Pull Secret Issues](./security/image-pull-secret-issues.md) - Fix private registry authentication
50. [Audit Logging Issues](./security/audit-logging-issues.md) - Troubleshoot audit logging problems

## How to Use These Runbooks

Each runbook follows a consistent format:
1. **Symptoms** - How to identify the issue
2. **Possible Causes** - Common reasons for the problem
3. **Diagnosis Steps** - Commands and procedures to identify the root cause
4. **Resolution Steps** - Step-by-step instructions to fix the issue
5. **Prevention** - How to prevent the issue from recurring

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request to add new runbooks or improve existing ones.