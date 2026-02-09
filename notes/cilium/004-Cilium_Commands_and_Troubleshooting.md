---
id: 004-Cilium_Commands_and_Troubleshooting
aliases:
  - cilium-commands
  - troubleshooting
tags:
  - cilium
  - kubernetes
  - sre
  - commands
  - troubleshooting
---

# Cilium Commands and Troubleshooting Guide

## Common Cilium Commands

### Basic Operations
```bash
# Check Cilium status
cilium status

# View Cilium configuration
cilium config

# View Cilium endpoint information
cilium endpoint list

# View Cilium identity information
cilium identity list
```

### Network Policy Operations
```bash
# View active network policies
cilium policy get

# Import a new policy
cilium policy import <policy-file>.json

# Delete a policy
cilium policy delete <policy-id>

# View policy verdicts (allow/deny decisions)
cilium policy trace
```

### Service and Load Balancer Operations
```bash
# View services
cilium service list

# View service statistics
cilium service get <service-id>

# Monitor service connections
cilium monitor --type svc
```

### Connectivity and Monitoring
```bash
# Connectivity check between endpoints
cilium connectivity test

# Monitor network traffic
cilium monitor

# Monitor specific traffic types
cilium monitor --type drop    # Dropped packets
cilium monitor --type debug   # Debug messages
cilium monitor --type capture # Captured packets
```

### Cilium Cluster Mesh Operations
```bash
# Check cluster mesh status
cilium clustermesh status

# View connected clusters
cilium clustermesh list

# Connectivity test between clusters
cilium connectivity test --multi-cluster
```

## Basic Troubleshooting Guide

### 1. Cilium Agent Issues
**Problem:** Cilium pods in CrashLoopBackOff state
```bash
# Check Cilium pod status
kubectl -n kube-system get pods -l k8s-app=cilium

# View detailed pod information
kubectl -n kube-system describe pod -l k8s-app=cilium

# Check Cilium logs
kubectl -n kube-system logs -l k8s-app=cilium --previous
```
**Solution:**
- Check if kernel version is compatible (4.8+ recommended)
- Verify required kernel modules are loaded (e.g., bpf, vxlan)
- Ensure sufficient privileges for BPF operations

### 2. Network Connectivity Issues
**Problem:** Pods cannot communicate with each other
```bash
# Verify endpoint status
cilium endpoint list

# Check connectivity between specific endpoints
cilium connectivity test

# Monitor dropped packets
cilium monitor --type drop
```
**Solution:**
- Check network policies for restrictive rules
- Verify pod CIDR assignments
- Confirm Cilium CNI is properly configured

### 3. Policy Enforcement Problems
**Problem:** Network policies are not being enforced
```bash
# Verify policy status
cilium policy get

# Trace policy decisions
cilium policy trace -s <source-labels> -d <destination-labels>

# Check endpoint policy resolution
cilium endpoint get <endpoint-id>
```
**Solution:**
- Confirm policies are correctly formatted
- Check that endpoint selectors match pod labels
- Ensure policies are imported without errors

### 4. Cilium Cluster Mesh Issues
**Problem:** Cross-cluster connectivity failing
```bash
# Check cluster mesh status
cilium clustermesh status

# Verify remote cluster connectivity
cilium clustermesh list

# Test cross-cluster connectivity
cilium connectivity test --multi-cluster
```
**Solution:**
- Ensure non-overlapping PodCIDRs across clusters
- Verify network connectivity between cluster nodes
- Check that shared CA certificates are properly configured

### 5. Resource Constraints
**Problem:** High memory or CPU usage
```bash
# Check resource usage
kubectl top pods -n kube-system -l k8s-app=cilium

# View Cilium metrics
cilium metrics list
```
**Solution:**
- Tune BPF map sizes if needed
- Check for excessive logging levels
- Consider increasing resources if running in high-scale environments