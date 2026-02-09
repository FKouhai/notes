---
id: kubernetes-network-policies
title: Kubernetes Network Policies
aliases:
  - network-security
  - pod-networking
  - network-isolation
tags:
  - kubernetes
  - network-policies
  - security
  - networking
description: Comprehensive guide to Kubernetes network policies with detailed examples and best practices
---

# Kubernetes Network Policies

Network policies control traffic flow between pods and network endpoints, providing network-level security and isolation.

## Network Policy Basics

### What are Network Policies?

Network policies are Kubernetes resources that define how pods are allowed to communicate with each other and other network endpoints.

**Key Characteristics**:
- Namespace-scoped (apply to pods within a namespace)
- Whitelist by default (deny all, then allow specific traffic)
- Require network plugin support (CNI plugin must implement NetworkPolicy)
- Select pods using label selectors
- Control both ingress and egress traffic

### Requirements

**Network Plugin Support**:
Network policies require a CNI plugin that supports them. Common plugins:
- Calico (full support)
- Cilium (full support)
- Weave Net (full support)
- Canal (partial support)
- Flannel (no native support, requires additional components)

**Verify Support**:
```bash
# Check if network policies are supported
kubectl get networkpolicy --all-namespaces

# Test with a simple policy
kubectl apply -f test-policy.yaml
kubectl get networkpolicy
```

## Default Network Policy Behavior

### Default Behavior (No Policies)

When no network policies exist in a namespace:
- All pods can communicate with each other
- All pods can communicate with network endpoints outside the cluster
- All network traffic is allowed

```yaml
# No network policies defined
# Result: All traffic allowed
```

### Deny All Traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: default
spec:
  podSelector: {}  # Match all pods
  policyTypes:
  - Ingress
  - Egress
```

**Behavior**:
- All ingress traffic to all pods is denied
- All egress traffic from all pods is denied
- Pods are completely isolated
- Use case: Least privilege, security-hardened environments

### Allow All Traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all
  namespace: default
spec:
  podSelector: {}  # Match all pods
  ingress:
  - {}  # Allow all ingress
  egress:
  - {}  # Allow all egress
```

**Behavior**:
- All ingress traffic is allowed
- All egress traffic is allowed
- Same as default behavior (no policies)
- Use case: Testing, troubleshooting, explicitly allowing all traffic

## Ingress Policies

### Deny All Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: default
spec:
  podSelector: {}  # Match all pods
  policyTypes:
  - Ingress
```

**Behavior**:
- All ingress traffic to all pods is denied
- Egress traffic is still allowed
- Use case: Prevent inbound connections while allowing outbound

### Allow Ingress from Specific Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-from-namespace
  namespace: app
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
    ports:
    - protocol: TCP
      port: 80
```

**Behavior**:
- Pods with label `app=web` in namespace `app` allow ingress
- Only from pods in namespace with label `name=frontend`
- Only on TCP port 80
- Use case: Multi-tier application architecture

### Allow Ingress from Specific Pods

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-from-pods
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

**Behavior**:
- Pods with label `app=backend` allow ingress
- Only from pods with label `app=frontend` in same namespace
- Only on TCP port 8080
- Use case: Service-to-service communication

### Allow Ingress from IP Block

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-from-ip
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 192.168.1.0/24
        except:
        - 192.168.1.1/32
    ports:
    - protocol: TCP
      port: 443
```

**Behavior**:
- Pods with label `app=web` allow ingress
- Only from IP range 192.168.1.0/24
- Except IP 192.168.1.1
- Only on TCP port 443
- Use case: Allow traffic from specific network ranges

### Allow Multiple Ingress Sources

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-multiple-ingress
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
    - podSelector:
        matchLabels:
          role: api-gateway
    ports:
    - protocol: TCP
      port: 80
```

**Behavior**:
- Pods with label `app=web` allow ingress
- From two sources:
  - Namespace with label `name=frontend`
  - Pods with label `role=api-gateway` (same namespace)
- Only on TCP port 80
- Use case: Multiple trusted sources

## Egress Policies

### Deny All Egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: default
spec:
  podSelector: {}  # Match all pods
  policyTypes:
  - Egress
```

**Behavior**:
- All egress traffic from all pods is denied
- Ingress traffic is still allowed
- Use case: Prevent data exfiltration, restricted environments

### Allow Egress to Specific Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-to-namespace
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: backend
    ports:
    - protocol: TCP
      port: 5432
```

**Behavior**:
- Pods with label `app=web` allow egress
- Only to namespace with label `name=backend`
- Only on TCP port 5432
- Use case: Database access from web tier

### Allow Egress to Specific Pods

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-to-pods
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 3306
```

**Behavior**:
- Pods with label `app=frontend` allow egress
- Only to pods with label `app=database`
- Only on TCP port 3306
- Use case: Frontend to database communication

### Allow Egress to External Service

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-external
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - protocol: TCP
      port: 443
    - protocol: TCP
      port: 53
```

**Behavior**:
- Pods with label `app=web` allow egress
- To anywhere (0.0.0.0/0)
- Only on TCP ports 443 and 53
- Use case: HTTPS and DNS access to internet

### Allow DNS Resolution

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: default
spec:
  podSelector: {}  # Match all pods
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}  # Any namespace (CoreDNS is in kube-system)
    ports:
    - protocol: UDP
      port: 53
```

**Behavior**:
- All pods allow egress to DNS servers
- UDP port 53 to any namespace
- Use case: Allow DNS queries when using restrictive egress policies

## Combined Ingress and Egress

### Both Ingress and Egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 8080
```

**Behavior**:
- Pods with label `app=web`
- Ingress: From namespace `frontend` on port 80
- Egress: To pods with label `app=api` on port 8080
- Use case: Web tier with specific inbound and outbound rules

## Advanced Network Policies

### Network Policy with Multiple Ports

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: multi-port-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 443
    - protocol: TCP
      port: 8080
```

**Behavior**:
- Pods with label `app=web` allow ingress
- From pods with label `app=frontend`
- On TCP ports 80, 443, and 8080
- Use case: Multi-service endpoints

### Network Policy for Database

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 5432
```

**Behavior**:
- Pods with label `app=postgres` allow ingress
- From pods with labels `app=backend` or `app=frontend`
- Only on TCP port 5432
- Use case: Restrict database access to specific applications

### Network Policy with Port Ranges

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: port-range-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: monitoring
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: prometheus
    ports:
    - protocol: TCP
      port: 9100  # Single port
    - protocol: TCP
      port: 9090
      endPort: 9100  # Port range 9090-9100
```

**Behavior**:
- Pods with label `app=monitoring` allow ingress
- From namespace with label `name=prometheus`
- On TCP port 9100 and range 9090-9100
- Use case: Monitoring and metrics collection

### Network Policy with Protocol

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: protocol-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: custom-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 10.0.0.0/8
    ports:
    - protocol: TCP
      port: 80
    - protocol: UDP
      port: 53
    - protocol: SCTP
      port: 3010
```

**Behavior**:
- Pods with label `app=custom-app` allow ingress
- From IP range 10.0.0.0/8
- On TCP port 80, UDP port 53, SCTP port 3010
- Use case: Multi-protocol applications

## Practical Examples

### Microservices Network Isolation

```yaml
# Database Network Policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 5432

---
# Backend Network Policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53

---
# Frontend Network Policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 443
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 8080
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
```

### Default Deny with Specific Allow

```yaml
# Deny all traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: secure-app
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# Allow DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: secure-app
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53

---
# Allow specific application traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-traffic
  namespace: secure-app
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 10.0.0.0/8
    ports:
    - protocol: TCP
      port: 443
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: backend
    ports:
    - protocol: TCP
      port: 8080
```

### Multi-Tenant Isolation

```yaml
# Tenant A - Frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: tenant-a-frontend
  namespace: tenant-a
spec:
  podSelector:
    matchLabels:
      tenant: a
      tier: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 10.0.0.0/8  # Corporate network
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          tenant: a
          tier: backend
    ports:
    - protocol: TCP
      port: 8080

---
# Tenant B - Frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: tenant-b-frontend
  namespace: tenant-b
spec:
  podSelector:
    matchLabels:
      tenant: b
      tier: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 10.0.0.0/8
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          tenant: b
          tier: backend
    ports:
    - protocol: TCP
      port: 8080
```

## Best Practices

### Network Policy Best Practices

1. **Start with Default Deny**
   ```yaml
   # Apply default deny policy first
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: default-deny
   spec:
     podSelector: {}
     policyTypes:
     - Ingress
     - Egress
   ```

2. **Use Label Selectors Effectively**
   ```yaml
   # Use specific, meaningful labels
   metadata:
     labels:
       app: webapp
       tier: frontend
       environment: production
   spec:
     podSelector:
       matchLabels:
         app: webapp
         tier: frontend
   ```

3. **Always Allow DNS**
   ```yaml
   # DNS is essential for cluster operation
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: allow-dns
   spec:
     podSelector: {}
     policyTypes:
     - Egress
     egress:
     - to:
       - namespaceSelector: {}
       ports:
       - protocol: UDP
         port: 53
   ```

4. **Use Namespace Selectors for Multi-Tier Apps**
   ```yaml
   # Namespace-based isolation
   spec:
     podSelector:
       matchLabels:
         app: backend
     ingress:
     - from:
       - namespaceSelector:
           matchLabels:
             name: frontend
   ```

5. **Be Specific with Ports**
   ```yaml
   # Allow only necessary ports
   ports:
   - protocol: TCP
     port: 80
   - protocol: TCP
     port: 443
   ```

6. **Document Network Policies**
   ```yaml
   # Use descriptive names and annotations
   metadata:
     name: web-ingress-policy
     annotations:
       description: "Allow ingress from frontend namespace on port 80"
       contact: "network-team@example.com"
   ```

7. **Test Policies in Staging**
   - Apply policies in non-production first
   - Test all application flows
   - Verify no unintended blocking

8. **Monitor Network Policy Impact**
   ```bash
   # Check if network policies are applied
   kubectl get networkpolicy -n <namespace>

   # Check pod connectivity
   kubectl exec -it <pod> -- nc -zv <target> <port>
   ```

9. **Use Port Ranges Efficiently**
   ```yaml
   # Use port ranges instead of individual ports
   ports:
   - protocol: TCP
     port: 9090
     endPort: 9100
   ```

10. **Consider IP Block Exceptions**
    ```yaml
    # Use IP blocks with exceptions for fine-grained control
    from:
    - ipBlock:
        cidr: 10.0.0.0/8
        except:
        - 10.0.0.1/32  # Block specific IP
    ```

## Troubleshooting

### Verify Network Policy Support

```bash
# Check if network policies are supported
kubectl get networkpolicy --all-namespaces

# Check CNI plugin
kubectl get daemonset -n kube-system
kubectl logs -n kube-system <cni-pod>

# Test with simple policy
kubectl apply -f test-policy.yaml
kubectl get networkpolicy test-policy
```

### Debug Network Policies

```bash
# List network policies
kubectl get networkpolicy -n <namespace>

# Describe network policy
kubectl describe networkpolicy <policy-name> -n <namespace>

# Check pod labels
kubectl get pods -n <namespace> --show-labels

# Verify policy matches pods
kubectl get pods -n <namespace> -l <label-selector>
```

### Test Connectivity

```bash
# Test from pod to pod
kubectl exec -it <source-pod> -- nc -zv <target-service> <port>

# Test DNS resolution
kubectl exec -it <pod> -- nslookup <service-name>

# Test external connectivity
kubectl exec -it <pod> -- curl -I https://example.com
```

### Common Issues

**Policy Not Applying**:
```bash
# Check pod selector
kubectl describe networkpolicy <policy-name>

# Verify pod labels
kubectl get pods -n <namespace> --show-labels

# Check for multiple policies
kubectl get networkpolicy -n <namespace>
```

**Connectivity Blocked**:
```bash
# Check egress policies
kubectl describe networkpolicy <policy-name>

# Verify DNS access
kubectl exec -it <pod> -- nslookup kubernetes.default

# Check for default deny
kubectl get networkpolicy -n <namespace>
```

**Service Not Accessible**:
```bash
# Check service endpoint
kubectl get endpoints <service-name>

# Verify policy allows traffic
kubectl describe networkpolicy <policy-name>

# Check port configuration
kubectl describe pod <pod-name>
```

## Quick Reference

### Basic Policies

```yaml
# Deny all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

# Allow all
spec:
  podSelector: {}
  ingress:
  - {}
  egress:
  - {}
```

### Ingress Patterns

```yaml
# From namespace
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        name: frontend
  ports:
  - protocol: TCP
    port: 80

# From pod
ingress:
- from:
  - podSelector:
      matchLabels:
        app: backend
  ports:
  - protocol: TCP
    port: 8080

# From IP
ingress:
- from:
  - ipBlock:
      cidr: 192.168.1.0/24
  ports:
  - protocol: TCP
    port: 443
```

### Egress Patterns

```yaml
# To namespace
egress:
- to:
  - namespaceSelector:
      matchLabels:
        name: backend
  ports:
  - protocol: TCP
    port: 5432

# To pod
egress:
- to:
  - podSelector:
      matchLabels:
        app: database
  ports:
  - protocol: TCP
    port: 3306

# To external
egress:
- to:
  - ipBlock:
      cidr: 0.0.0.0/0
  ports:
  - protocol: TCP
    port: 443
```

### Combined Patterns

```yaml
# Ingress and Egress
policyTypes:
- Ingress
- Egress
ingress:
- from:
  - podSelector:
      matchLabels:
        app: frontend
  ports:
  - protocol: TCP
    port: 80
egress:
- to:
  - podSelector:
      matchLabels:
        app: backend
  ports:
  - protocol: TCP
    port: 8080
```

### Commands

```bash
# Create network policy
kubectl apply -f policy.yaml

# List network policies
kubectl get networkpolicy

# Describe network policy
kubectl describe networkpolicy <name>

# Delete network policy
kubectl delete networkpolicy <name>

# Test connectivity
kubectl exec -it <pod> -- nc -zv <target> <port>
```
