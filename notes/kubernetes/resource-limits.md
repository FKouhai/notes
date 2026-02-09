---
id: kubernetes-resource-limits
title: Kubernetes Resource Limits and Requests
aliases:
  - resource-requests
  - resource-management
  - qos-classes
tags:
  - kubernetes
  - resources
  - limits
  - requests
  - qos
description: Comprehensive guide to Kubernetes resource requests and limits with detailed examples and best practices
---

# Kubernetes Resource Limits and Requests

Resource limits and requests define how much CPU and memory a container can use in Kubernetes.

## Requests vs Limits - Key Differences

**Requests** define the guaranteed minimum resources allocated to a container.

**Limits** define the maximum resources a container can consume.

### How Requests Work

- **Scheduling**: Kubernetes uses requests to schedule pods on nodes with sufficient available resources
- **Guaranteed Resources**: The container is guaranteed to have at least the requested amount of resources
- **QoS Class**: Pods with requests set are assigned Guaranteed or Burstable QoS classes
- **Node Selection**: Scheduler only considers nodes that have enough allocatable resources to satisfy all pod requests
- **Resource Accounting**: Requests are deducted from a node's allocatable resources

### How Limits Work

- **Throttling**: When a container exceeds its CPU limit, it's throttled (doesn't get more CPU time)
- **Termination**: When a container exceeds its memory limit, it's terminated and restarted (OOM killed)
- **Bursting**: Containers can burst up to their limits when resources are available
- **Protection**: Limits prevent one container from starving others on the same node
- **Enforcement**: Linux cgroups enforce limits at the kernel level

### Key Differences

| Aspect | Requests | Limits |
|--------|----------|--------|
| Purpose | Guaranteed minimum | Maximum allowed |
| Scheduling | Used for pod placement | Not considered for scheduling |
| Default | 0 (no guarantee) | Defaults to requests (if set) or node capacity |
| CPU Behavior | Guaranteed allocation | Throttled if exceeded |
| Memory Behavior | Guaranteed allocation | OOM killed if exceeded |
| Required | No | No |
| Relationship | Must be ≤ limits | Must be ≥ requests |
| Effect on QoS | Affects QoS class | Affects QoS class |

## CPU Resources

### CPU Units

CPU is measured in "millicores" (m). 1000m = 1 CPU core.

```yaml
# Common CPU values
cpu: "100m"    # 0.1 CPU (10% of 1 core)
cpu: "250m"    # 0.25 CPU (25% of 1 core)
cpu: "500m"    # 0.5 CPU (50% of 1 core)
cpu: "1000m"   # 1 CPU (100% of 1 core)
cpu: "2000m"   # 2 CPUs (200% of 1 core)
cpu: "0.5"     # Alternative syntax: 0.5 CPU
cpu: "1"       # Alternative syntax: 1 CPU
```

### CPU Requests

CPU requests guarantee a minimum amount of CPU time.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpu-request
spec:
  containers:
  - name: app
    image: nginx:latest
    resources:
      requests:
        cpu: "250m"  # Guaranteed 0.25 CPU
```

**CPU Request Behavior**:
- Container is guaranteed 25% of 1 CPU core
- Scheduler ensures node has at least 250m available
- Container can burst higher if CPU is available and no limit is set
- If node is under load, container gets at least 250m

### CPU Limits

CPU limits cap the maximum CPU a container can use.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpu-limit
spec:
  containers:
  - name: app
    image: nginx:latest
    resources:
      requests:
        cpu: "100m"
      limits:
        cpu: "500m"  # Maximum 0.5 CPU
```

**CPU Limit Behavior**:
- Container is throttled if it tries to use more than 500m
- Throttling reduces CPU time slice, not immediate stop
- Performance degrades when near or at limit
- Container continues running (not killed)
- Cgroups enforce the limit at kernel level

### CPU Throttling Explained

When a container exceeds its CPU limit:

```yaml
# Container configured with:
resources:
  limits:
    cpu: "500m"  # 0.5 CPU

# If container tries to use 1000m:
# - CFS (Completely Fair Scheduler) throttles the container
# - Container gets 500m time slices
# - Performance degrades proportional to overage
# - No restart, just slower execution
# - CPU usage metrics show limit, not actual requested
```

**Monitoring CPU Throttling**:
```bash
# Check CPU throttling (containerd)
kubectl exec mypod -- cat /sys/fs/cgroup/cpu/cpu.stat

# Check CPU usage vs limits
kubectl top pod mypod --containers

# View detailed metrics
kubectl describe pod mypod | grep -A 5 "Limits\|Requests"
```

## Memory Resources

### Memory Units

Memory is measured in bytes. Common suffixes:

```yaml
# Common memory values
memory: "128Mi"   # 128 Mebibytes
memory: "256Mi"   # 256 Mebibytes
memory: "512Mi"   # 512 Mebibytes
memory: "1Gi"     # 1 Gibibyte
memory: "2Gi"     # 2 Gibibytes
memory: "4Gi"     # 4 Gibibytes

# Unit comparison
# K/Ki = Kilobyte/Kibibyte = 1,000/1,024 bytes
# M/Mi = Megabyte/Mebibyte = 1,000,000/1,048,576 bytes
# G/Gi = Gigabyte/Gibibyte = 1,000,000,000/1,073,741,824 bytes
```

### Memory Requests

Memory requests guarantee minimum RAM allocation.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-request
spec:
  containers:
  - name: app
    image: nginx:latest
    resources:
      requests:
        memory: "256Mi"  # Guaranteed 256 MiB RAM
```

**Memory Request Behavior**:
- Container is guaranteed at least 256 MiB of RAM
- Scheduler ensures node has at least 256 MiB available
- Memory is reserved but not necessarily allocated upfront
- Other pods cannot overprovision this memory

### Memory Limits

Memory limits cap maximum RAM usage.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-limit
spec:
  containers:
  - name: app
    image: nginx:latest
    resources:
      requests:
        memory: "128Mi"
      limits:
        memory: "512Mi"  # Maximum 512 MiB
```

**Memory Limit Behavior**:
- Container is OOM killed if it exceeds 512 MiB
- OOM (Out of Memory) killer terminates container immediately
- Container is automatically restarted by Kubernetes
- Application state is lost unless using persistent storage
- Cgroups enforce the limit at kernel level

### OOM Kill Explained

When a container exceeds its memory limit:

```yaml
# Container configured with:
resources:
  limits:
    memory: "512Mi"  # 512 MiB

# If container tries to allocate 600 MiB:
# - Linux OOM killer detects limit breach
# - Container is terminated immediately
# - Kubernetes restarts the container
# - Restart count increases
# - Previous state is lost
# - OOM event recorded in pod status
```

**Detecting OOM Kills**:
```bash
# Check if pod was OOM killed
kubectl describe pod mypod | grep -i "oom"

# View pod events
kubectl get events --field-selector involvedObject.name=mypod

# Check restart count
kubectl get pod mypod
```

## Resource Scenarios

### Scenario 1: Requests Only (Burstable QoS)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: requests-only
spec:
  containers:
  - name: app
    image: nginx:latest
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      # No limits
```

**Behavior**:
- Guaranteed 100m CPU, 128 MiB memory
- Can use all available CPU on node (may be throttled under load)
- Can use all available memory on node (risk of OOM if node runs out)
- Lower priority than Guaranteed pods
- QoS Class: Burstable
- Use case: Batch jobs, development environments

### Scenario 2: Limits Only (Burstable QoS)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: limits-only
spec:
  containers:
  - name: app
    image: nginx:latest
    resources:
      limits:
        cpu: "500m"
        memory: "512Mi"
      # No requests - defaults to limits
```

**Behavior**:
- Requests default to limits (500m CPU, 512 MiB memory)
- Guaranteed 500m CPU, 512 MiB memory
- Cannot exceed 500m CPU (throttled)
- OOM killed at 512 MiB memory
- QoS Class: Burstable (because requests weren't explicitly set)
- Use case: Production workloads that need guaranteed resources

### Scenario 3: Requests = Limits (Guaranteed QoS)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed
spec:
  containers:
  - name: app
    image: nginx:latest
    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
```

**Behavior**:
- Guaranteed 500m CPU, 512 MiB memory
- Cannot exceed 500m CPU (throttled)
- OOM killed at 512 MiB memory
- Highest priority for scheduling
- Less likely to be evicted
- QoS Class: Guaranteed
- Use case: Critical production workloads

### Scenario 4: Requests < Limits (Burstable QoS)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: burstable
spec:
  containers:
  - name: app
    image: nginx:latest
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "1000m"
        memory: "1Gi"
```

**Behavior**:
- Guaranteed 100m CPU, 128 MiB memory
- Can burst up to 1000m CPU, 1 GiB memory
- Throttled when exceeding 1000m CPU
- OOM killed when exceeding 1 GiB memory
- Flexible for bursty workloads
- QoS Class: Burstable
- Use case: Web servers with variable traffic

### Scenario 5: No Resources (BestEffort QoS)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: best-effort
spec:
  containers:
  - name: app
    image: nginx:latest
    # No requests, no limits
```

**Behavior**:
- No guaranteed resources
- Can use all available CPU and memory
- First to be evicted under memory pressure
- Lowest priority
- QoS Class: BestEffort
- Use case: One-off tasks, testing

## Quality of Service (QoS) Classes

Kubernetes assigns QoS classes based on resource requests and limits.

### Guaranteed QoS

All containers must have both requests and limits set, with requests == limits for all resources.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
spec:
  containers:
  - name: app
    image: nginx:latest
    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
  - name: sidecar
    image: busybox:latest
    command: ["sleep", "3600"]
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "100m"
        memory: "64Mi"
```

**Guaranteed QoS Characteristics**:
- Highest priority for scheduling
- Lowest chance of being evicted
- Guaranteed resources
- Use case: Critical production services

**Verification**:
```bash
kubectl get pod guaranteed-pod -o jsonpath='{.status.qosClass}'
# Output: Guaranteed
```

### Burstable QoS

At least one container has requests or limits (but not Guaranteed).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod
spec:
  containers:
  - name: app
    image: nginx:latest
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "1000m"
        memory: "1Gi"
  - name: sidecar
    image: busybox:latest
    command: ["sleep", "3600"]
    resources:
      requests:
        cpu: "50m"
      # No limits
```

**Burstable QoS Characteristics**:
- Medium priority for scheduling
- Medium chance of being evicted
- Can burst to limits
- Use case: Standard production workloads

**Verification**:
```bash
kubectl get pod burstable-pod -o jsonpath='{.status.qosClass}'
# Output: Burstable
```

### BestEffort QoS

No requests or limits for any container.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: best-effort-pod
spec:
  containers:
  - name: app
    image: nginx:latest
  - name: sidecar
    image: busybox:latest
    command: ["sleep", "3600"]
```

**BestEffort QoS Characteristics**:
- Lowest priority for scheduling
- Highest chance of being evicted
- No guaranteed resources
- Use case: Testing, development

**Verification**:
```bash
kubectl get pod best-effort-pod -o jsonpath='{.status.qosClass}'
# Output: BestEffort
```

## Multi-Container Pods

Each container in a pod has its own resource requests and limits.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
spec:
  containers:
  - name: nginx
    image: nginx:latest
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
  - name: sidecar
    image: busybox:latest
    command: ["sleep", "3600"]
    resources:
      requests:
        cpu: "50m"
        memory: "64Mi"
      limits:
        cpu: "100m"
        memory: "128Mi"
  # Total pod requests: 150m CPU, 192Mi memory
  # Total pod limits: 600m CPU, 640Mi memory
```

**Pod-Level Scheduling**:
- Scheduler considers sum of all container requests
- Each container is independently limited
- Pod is scheduled on node with capacity for total requests

## Init Containers

Init containers also need resource requests and limits.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-init
spec:
  initContainers:
  - name: init
    image: busybox:latest
    command: ["sh", "-c", "echo init"]
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
  containers:
  - name: app
    image: nginx:latest
    resources:
      requests:
        cpu: "200m"
        memory: "128Mi"
      limits:
        cpu: "1000m"
        memory: "512Mi"
```

**Scheduling with Init Containers**:
- Scheduler considers highest resource request across all init and app containers
- Init containers run to completion before app containers start
- Init container resources are freed after completion

## Resource Management

### LimitRange - Default Values

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
```

**Behavior**:
- Containers without limits get default limits
- Containers without requests get default requests
- Ensures all containers have resource limits

### ResourceQuota - Namespace Limits

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
  namespace: production
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    persistentvolumeclaims: "5"
```

**Behavior**:
- Enforces maximum resource usage in namespace
- Prevents overprovisioning
- Sum of all pod requests/limits must fit within quota

## Best Practices

### Production Workloads

```yaml
# Always set both requests and limits
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    spec:
      containers:
      - name: webapp
        image: webapp:latest
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
```

**Best Practice Guidelines**:

1. **Set Requests Based on Typical Usage**
   - Monitor actual usage over time
   - Use average + some buffer for requests
   - Don't use minimum possible (causes overprovisioning)

2. **Set Limits Based on Maximum Acceptable Usage**
   - Use 95th or 99th percentile of actual usage
   - Consider burst scenarios
   - Leave room for unexpected spikes

3. **Use Requests < Limits for Bursty Workloads**
   - Web servers, APIs benefit from bursting
   - Allows flexibility while preventing runaway

4. **Use Requests == Limits for Critical Workloads**
   - Databases, critical services need guaranteed performance
   - Ensures predictable behavior

5. **Always Set Memory Limits**
   - Prevents memory leaks from affecting node
   - Protects other pods on same node

6. **Consider Not Setting CPU Limits for Non-Critical Apps**
   - Allows full CPU utilization
   - Fair sharing via CFS
   - Consider impact on other pods

7. **Test Limits in Staging**
   - Verify application behaves correctly under limits
   - Test CPU throttling and OOM scenarios
   - Adjust based on real-world usage

8. **Monitor Resource Usage**
   ```bash
   # View resource usage
   kubectl top pods
   kubectl top nodes

   # Check pod resource requests/limits
   kubectl describe pod mypod | grep -A 5 "Requests\|Limits"

   # View QoS class
   kubectl get pod mypod -o jsonpath='{.status.qosClass}'
   ```

9. **Use LimitRange for Enforceable Defaults**
   - Ensures all pods have resource limits
   - Prevents runaway containers
   - Makes resource management easier

10. **Use ResourceQuota for Namespace Isolation**
    - Prevents one namespace from consuming all resources
    - Enforces multi-tenant boundaries
    - Enables cost allocation

## Troubleshooting

### Pod Not Scheduling

```bash
# Check why pod is not scheduling
kubectl describe pod mypod | grep -A 10 "Events"

# Common causes:
# - Insufficient resources on nodes
# - Requests exceed available capacity
# - Node selector/affinity constraints
```

### OOM Kills

```bash
# Check for OOM kills
kubectl describe pod mypod | grep -i "oom"

# View events
kubectl get events --field-selector involvedObject.name=mypod

# Check restart count
kubectl get pod mypod

# Solutions:
# - Increase memory limits
# - Fix memory leaks in application
# - Add vertical pod autoscaler
```

### CPU Throttling

```bash
# Check CPU throttling
kubectl top pod mypod --containers

# View detailed metrics
kubectl describe node <node-name> | grep -A 20 "Allocated resources"

# Solutions:
# - Increase CPU limits
# - Optimize application performance
# - Scale horizontally instead of vertically
```

### QoS Issues

```bash
# Check QoS class
kubectl get pod mypod -o jsonpath='{.status.qosClass}'

# Verify all containers have same QoS
kubectl get pod -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.qosClass}{"\n"}{end}'

# Ensure all containers in pod have requests == limits for Guaranteed
```

## Advanced Features

### Vertical Pod Autoscaler (VPA)

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: webapp-vpa
  namespace: default
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: '*'
      controlledResources: ["cpu", "memory"]
```

**VPA Behavior**:
- Automatically adjusts pod resource requests and limits
- Learns from historical usage patterns
- Can update pods automatically (with recreation)
- Supports multiple update modes

### Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

**HPA Behavior**:
- Scales pods based on CPU/memory utilization
- Uses resource requests as baseline
- Requires resource requests to be set
- Works alongside VPA (with careful configuration)

### Node Allocatable

```bash
# View node allocatable resources
kubectl describe node <node-name> | grep -A 5 "Allocatable resources"

# Allocatable = Capacity - System Reserved - Kubelet Reserved
```

**Understanding Allocatable**:
- System Reserved: Resources for OS and system daemons
- Kubelet Reserved: Resources for Kubernetes components
- Allocatable: Available for pod scheduling

### Resource Classes

```yaml
# Resource class for high-priority workloads
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: high-priority
handler: nvidia
overhead:
  podFixed:
    memory: "1Gi"
    cpu: "500m"
scheduling:
  nodeSelector:
    node-role.kubernetes.io/high-priority: "true"
  tolerations:
  - key: "high-priority"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

## Quick Reference

### Resource Syntax

```yaml
# CPU
cpu: "100m"     # 0.1 CPU
cpu: "500m"     # 0.5 CPU
cpu: "1000m"    # 1 CPU
cpu: "1"        # 1 CPU (alternative)

# Memory
memory: "128Mi"  # 128 MiB
memory: "1Gi"    # 1 GiB
memory: "512M"   # 512 MB

# Requests only
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"

# Limits only (defaults requests to limits)
resources:
  limits:
    cpu: "500m"
    memory: "512Mi"

# Both requests and limits
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

### QoS Classes

```yaml
# Guaranteed (requests == limits)
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

# Burstable (requests < limits or only requests/limits)
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "1000m"
    memory: "1Gi"

# BestEffort (no requests or limits)
# No resources section
```

### Commands

```bash
# Check resource usage
kubectl top pods
kubectl top nodes

# View QoS class
kubectl get pod mypod -o jsonpath='{.status.qosClass}'

# Check for OOM kills
kubectl describe pod mypod | grep -i "oom"

# View requests and limits
kubectl describe pod mypod | grep -A 5 "Requests\|Limits"

# View events
kubectl get events --field-selector involvedObject.name=mypod
```
