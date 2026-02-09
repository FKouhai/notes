---
id: kubectl-advanced
title: Advanced kubectl - JSONPath and Patch
aliases:
  - kubectl-commands
  - jsonpath
  - kubectl-patch
tags:
  - kubernetes
  - kubectl
  - jsonpath
  - advanced
description: Advanced kubectl commands using JSONPath and kubectl patch with practical examples
---

# Advanced kubectl - JSONPath and Patch

Advanced `kubectl` commands using JSONPath for querying and `kubectl patch` for modifying Kubernetes resources programmatically.

## JSONPath Basics

JSONPath is a query language for extracting data from JSON documents. Kubernetes uses it to format output from `kubectl` commands.

### Basic Syntax

```bash
# Basic JSONPath query
kubectl get pods -o jsonpath='{.items[0].metadata.name}'

# Get all pod names
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Get specific fields
kubectl get pods -o jsonpath='{.items[*].metadata.name}{"\n"}'

# Custom separator
kubectl get pods -o jsonpath='{.items[*].metadata.name}{"\t"}'
```

## JSONPath Examples

### Extract Pod Information

```bash
# Get all pod names
kubectl get pods -o jsonpath='{.items[*].metadata.name}{"\n"}'

# Get pod IPs
kubectl get pods -o jsonpath='{.items[*].status.podIP}{"\n"}'

# Get pod namespaces
kubectl get pods -o jsonpath='{.items[*].metadata.namespace}{"\n"}'

# Get pod creation timestamps
kubectl get pods -o jsonpath='{.items[*].metadata.creationTimestamp}{"\n"}'

# Get pod node names
kubectl get pods -o jsonpath='{.items[*].spec.nodeName}{"\n"}'

# Get pod container names
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*]}{.name}{"\n"}{end}'

# Get pod container images
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*]}{.image}{"\n"}{end}'
```

### Extract Node Information

```bash
# Get all node names
kubectl get nodes -o jsonpath='{.items[*].metadata.name}{"\n"}'

# Get node IPs
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}{"\n"}'

# Get node ready status
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'

# Get node capacity
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.capacity.cpu}{"\t"}{.status.capacity.memory}{"\n"}{end}'

# Get node labels
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels}{"\n"}{end}'
```

### Extract Service Information

```bash
# Get all service names
kubectl get svc -o jsonpath='{.items[*].metadata.name}{"\n"}'

# Get service cluster IPs
kubectl get svc -o jsonpath='{.items[*].spec.clusterIP}{"\n"}'

# Get service ports
kubectl get svc -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.ports[*]}{.port}{"/"}{.targetPort}{"\n"}{end}'

# Get service types
kubectl get svc -o jsonpath='{.items[*].spec.type}{"\n"}'

# Get service selectors
kubectl get svc -o jsonpath='{.items[*].spec.selector}{"\n"}'
```

### Extract Deployment Information

```bash
# Get all deployment names
kubectl get deployments -o jsonpath='{.items[*].metadata.name}{"\n"}'

# Get deployment replicas
kubectl get deployments -o jsonpath='{.items[*].spec.replicas}{"\n"}'

# Get deployment images
kubectl get deployments -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.template.spec.containers[*].image}{"\n"}{end}'

# Get deployment available replicas
kubectl get deployments -o jsonpath='{.items[*].status.availableReplicas}{"\n"}'

# Get deployment selectors
kubectl get deployments -o jsonpath='{.items[*].spec.selector.matchLabels}{"\n"}'
```

## Advanced JSONPath Queries

### Filtering and Selection

```bash
# Get pods with specific label
kubectl get pods -o jsonpath='{range .items[?(@.metadata.labels.app=="web")]}{.metadata.name}{"\n"}{end}'

# Get running pods
kubectl get pods -o jsonpath='{range .items[?(@.status.phase=="Running")]}{.metadata.name}{"\n"}{end}'

# Get failed pods
kubectl get pods -o jsonpath='{range .items[?(@.status.phase=="Failed")]}{.metadata.name}{"\n"}{end}'

# Get pods with specific container
kubectl get pods -o jsonpath='{range .items[?(@.spec.containers[*].name=="nginx")]}{.metadata.name}{"\n"}{end}'

# Get services of type NodePort
kubectl get svc -o jsonpath='{range .items[?(@.spec.type=="NodePort")]}{.metadata.name}{"\n"}{end}'
```

### Nested Queries

```bash
# Get pod container resources
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .spec.containers[*]}{"  "}{.name}{":\t"}{.resources.requests.cpu}{"/"}{.resources.limits.cpu}{"\n"}{end}'

# Get pod volumes
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .spec.volumes[*]}{"  "}{.name}{":\t"}{.persistentVolumeClaim.claimName}{"\n"}{end}'

# Get pod environment variables
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .spec.containers[*]}{"  "}{.name}{":\n"}{range .env[*]}{"    "}{.name}{"="}{.value}{"\n"}{end}{end}'

# Get node taints
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .spec.taints[*]}{"  "}{.key}{"="}{.value}{":"}{.effect}{"\n"}{end}'
```

### Custom Formatting

```bash
# Create table-like output
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\t"}{.status.podIP}{"\n"}{end}'

# Format as key-value pairs
kubectl get pods -o jsonpath='{range .items[*]}{"Name: "}{.metadata.name}{"\nStatus: "}{.status.phase}{"\nIP: "}{.status.podIP}{"\n---\n"}{end}'

# Extract specific pod
kubectl get pod mypod -o jsonpath='{.metadata.name}{"\n"}{.status.phase}{"\n"}{.status.podIP}'

# Get container restart counts
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.containerStatuses[*].restartCount}{"\n"}{end}'

# Get pod conditions
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .status.conditions[*]}{"  "}{.type}{": "}{.status}{"\n"}{end}'
```

## kubectl patch Basics

`kubectl patch` updates a field(s) of a resource using strategic merge patch, JSON merge patch, or JSON merge patch.

### Strategic Merge Patch

```bash
# Update deployment replicas
kubectl patch deployment myapp -p '{"spec":{"replicas":5}}'

# Update image
kubectl patch deployment myapp -p '{"spec":{"template":{"spec":{"containers":[{"name":"myapp","image":"myapp:v2"}]}}}}'

# Update service selector
kubectl patch service myservice -p '{"spec":{"selector":{"app":"web"}}}'

# Update ConfigMap data
kubectl patch configmap myconfig -p '{"data":{"key":"newvalue"}}'

# Update Secret data (base64 encoded)
kubectl patch secret mysecret -p '{"data":{"password":"bmV3cGFzc3dvcmQ="}}'
```

### JSON Merge Patch

```bash
# Update with JSON merge patch
kubectl patch deployment myapp --type='merge' -p '{"spec":{"replicas":3}}'

# Remove fields with null
kubectl patch deployment myapp --type='merge' -p '{"spec":{"template":{"spec":{"containers":[{"name":"myapp","resources":{"limits":null}}]}}}}'

# Update multiple fields
kubectl patch deployment myapp --type='merge' -p '{"spec":{"replicas":5,"strategy":{"type":"RollingUpdate"}}}'
```

### Strategic Merge Patch (with arrays)

```bash
# Update container in deployment (strategic)
kubectl patch deployment myapp -p '{"spec":{"template":{"spec":{"containers":[{"name":"myapp","image":"myapp:v2"}]}}}}'

# Add environment variable
kubectl patch deployment myapp -p '{"spec":{"template":{"spec":{"containers":[{"name":"myapp","env":[{"name":"NEW_VAR","value":"value"}]}]}}}}'

# Update specific env var (requires strategic patch)
kubectl patch deployment myapp --type='strategic' -p '{"spec":{"template":{"spec":{"containers":[{"name":"myapp","env":[{"name":"DEBUG","value":"true"}]}]}}}}'
```

## kubectl patch Examples

### Pod Operations

```bash
# Add label to pod
kubectl patch pod mypod -p '{"metadata":{"labels":{"app":"web"}}}'

# Add annotation
kubectl patch pod mypod -p '{"metadata":{"annotations":{"description":"Web pod"}}}'

# Update pod image (not recommended, use deployment)
kubectl patch pod mypod -p '{"spec":{"containers":[{"name":"myapp","image":"myapp:v2"}]}}'

# Update pod resource requests/limits
kubectl patch pod mypod -p '{"spec":{"containers":[{"name":"myapp","resources":{"requests":{"cpu":"500m","memory":"512Mi"}}}]}}'
```

### Deployment Operations

```bash
# Update replicas
kubectl patch deployment myapp -p '{"spec":{"replicas":5}}'

# Rollback to previous image
kubectl patch deployment myapp -p '{"spec":{"template":{"spec":{"containers":[{"name":"myapp","image":"myapp:v1"}]}}}}'

# Update image tag
kubectl patch deployment myapp -p '{"spec":{"template":{"spec":{"containers":[{"name":"myapp","image":"myapp:latest"}]}}}}'

# Add environment variable
kubectl patch deployment myapp -p '{"spec":{"template":{"spec":{"containers":[{"name":"myapp","env":[{"name":"API_URL","value":"https://api.example.com"}]}]}}}}'

# Update resource limits
kubectl patch deployment myapp -p '{"spec":{"template":{"spec":{"containers":[{"name":"myapp","resources":{"limits":{"cpu":"2","memory":"2Gi"}}}]}}}}'

# Set node selector
kubectl patch deployment myapp -p '{"spec":{"template":{"spec":{"nodeSelector":{"node-role.kubernetes.io/worker":"true"}}}}}'

# Add toleration
kubectl patch deployment myapp -p '{"spec":{"template":{"spec":{"tolerations":[{"key":"dedicated","operator":"Equal","value":"web","effect":"NoSchedule"}]}}}}'

# Update rolling update strategy
kubectl patch deployment myapp -p '{"spec":{"strategy":{"rollingUpdate":{"maxSurge":"25%","maxUnavailable":"25%"}}}}'

# Set revision history limit
kubectl patch deployment myapp -p '{"spec":{"revisionHistoryLimit":10}}'

# Update min ready seconds
kubectl patch deployment myapp -p '{"spec":{"minReadySeconds":30}}'
```

### Service Operations

```bash
# Change service type
kubectl patch service myservice -p '{"spec":{"type":"NodePort"}}'

# Update NodePort
kubectl patch service myservice -p '{"spec":{"ports":[{"port":80,"nodePort":30080}]}}'

# Add external IPs
kubectl patch service myservice -p '{"spec":{"externalIPs":["1.2.3.4"]}}'

# Update load balancer source ranges
kubectl patch service myservice -p '{"spec":{"loadBalancerSourceRanges":["10.0.0.0/8"]}}'

# Update session affinity
kubectl patch service myservice -p '{"spec":{"sessionAffinity":"ClientIP","sessionAffinityConfig":{"clientIP":{"timeoutSeconds":10800}}}}'

# Add service annotations
kubectl patch service myservice -p '{"metadata":{"annotations":{"service.beta.kubernetes.io/aws-load-balancer-type":"nlb"}}}'
```

### ConfigMap Operations

```bash
# Update ConfigMap data
kubectl patch configmap myconfig -p '{"data":{"database.url":"postgresql://localhost:5432/db"}}'

# Add new key
kubectl patch configmap myconfig -p '{"data":{"new.key":"new.value"}}'

# Update multiple keys
kubectl patch configmap myconfig -p '{"data":{"key1":"value1","key2":"value2"}}'

# Add ConfigMap annotations
kubectl patch configmap myconfig -p '{"metadata":{"annotations":{"description":"Database configuration"}}}'
```

### Secret Operations

```bash
# Update Secret data (base64 encoded)
echo -n "newpassword" | base64
kubectl patch secret mysecret -p '{"data":{"password":"bmV3cGFzc3dvcmQ="}}'

# Add new secret
kubectl patch secret mysecret -p '{"data":{"new_secret":"dmFsdWU="}}'

# Update stringData (automatic encoding)
kubectl patch secret mysecret --type='merge' -p '{"stringData":{"password":"newpassword"}}'

# Add secret annotations
kubectl patch secret mysecret -p '{"metadata":{"annotations":{"description":"Database credentials"}}}'
```

### Ingress Operations

```bash
# Add ingress class
kubectl patch ingress myingress -p '{"spec":{"ingressClassName":"nginx"}}'

# Update ingress annotations
kubectl patch ingress myingress -p '{"metadata":{"annotations":{"nginx.ingress.kubernetes.io/ssl-redirect":"true"}}}'

# Update ingress hosts
kubectl patch ingress myingress -p '{"spec":{"rules":[{"host":"new.example.com","http":{"paths":[{"path":"/","pathType":"Prefix","backend":{"service":{"name":"myservice","port":{"number":80}}}}]}}]}}'

# Add TLS configuration
kubectl patch ingress myingress -p '{"spec":{"tls":[{"hosts":["example.com"],"secretName":"tls-secret"}]}}'
```

### PersistentVolume Operations

```bash
# Update PV reclaim policy
kubectl patch pv mypv -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'

# Update PV access modes
kubectl patch pv mypv -p '{"spec":{"accessModes":["ReadWriteMany"]}}'

# Update PV capacity
kubectl patch pv mypv -p '{"spec":{"capacity":{"storage":"20Gi"}}}'

# Add PV annotation
kubectl patch pv mypv -p '{"metadata":{"annotations":{"volume.beta.kubernetes.io/storage-class":"fast"}}}'
```

### PersistentVolumeClaim Operations

```bash
# Update PVC storage request
kubectl patch pvc mypvc -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# Update PVC storage class
kubectl patch pvc mypvc -p '{"spec":{"storageClassName":"fast"}}'

# Add PVC annotations
kubectl patch pvc mypvc -p '{"metadata":{"annotations":{"volume.beta.kubernetes.io/storage-provisioner":"efs.csi.aws.com"}}}'
```

### Namespace Operations

```bash
# Add namespace label
kubectl patch namespace myns -p '{"metadata":{"labels":{"environment":"production"}}}'

# Add namespace annotation
kubectl patch namespace myns -p '{"metadata":{"annotations":{"description":"Production namespace"}}}'

# Update namespace finalizers
kubectl patch namespace myns -p '{"metadata":{"finalizers":["kubernetes"]}}'
```

## Complex Patch Operations

### Patching from File

```bash
# Patch from JSON file
kubectl patch deployment myapp -p "$(cat patch.json)"

# Patch from YAML file (convert to JSON first)
kubectl patch deployment myapp -p "$(yq -o=json '.' patch.yaml)"

# Patch multiple deployments from file
kubectl patch deployment myapp1 -p "$(cat patch.json)"
kubectl patch deployment myapp2 -p "$(cat patch.json)"
```

### Conditional Patching

```bash
# Patch only if specific condition is met
kubectl get deployment myapp -o json | \
  jq 'select(.spec.replicas == 1)' | \
  kubectl apply -f -

# Patch based on current image
kubectl get deployment myapp -o jsonpath='{.spec.template.spec.containers[0].image}' | \
  grep -q "v1" && \
  kubectl patch deployment myapp -p '{"spec":{"template":{"spec":{"containers":[{"name":"myapp","image":"myapp:v2"}]}}}}'

# Patch only if pod is running
kubectl get pod mypod -o jsonpath='{.status.phase}' | \
  grep -q "Running" && \
  kubectl patch pod mypod -p '{"metadata":{"labels":{"status":"running"}}}'
```

### Bulk Operations

```bash
# Patch all deployments with specific label
kubectl get deployments -l app=web -o json | \
  jq -c '.items[] | .metadata.name' | \
  while read name; do
    kubectl patch deployment "$name" -p '{"spec":{"replicas":5}}'
  done

# Scale all deployments in namespace
kubectl get deployments --all-namespaces -o json | \
  jq -r '.items[] | "\(.metadata.namespace)\t\(.metadata.name)"' | \
  while IFS=$'\t' read -r ns name; do
    kubectl patch deployment "$name" -n "$ns" -p '{"spec":{"replicas":3}}'
  done

# Add annotation to all pods
kubectl get pods -o json | \
  jq -r '.items[] | .metadata.name' | \
  while read name; do
    kubectl patch pod "$name" -p '{"metadata":{"annotations":{"patched":"true"}}}'
  done
```

## Practical Use Cases

### Rolling Updates

```bash
# Update image across multiple namespaces
kubectl get deployments --all-namespaces -l app=myapp -o json | \
  jq -r '.items[] | "\(.metadata.namespace)\t\(.metadata.name)"' | \
  while IFS=$'\t' read -r ns name; do
    kubectl patch deployment "$name" -n "$ns" \
      -p '{"spec":{"template":{"spec":{"containers":[{"name":"myapp","image":"myapp:v2"}]}}}}'
  done

# Enable rollout on multiple deployments
kubectl get deployments -o json | \
  jq -r '.items[] | .metadata.name' | \
  while read name; do
    kubectl patch deployment "$name" -p '{"spec":{"strategy":{"rollingUpdate":{"maxSurge":"1","maxUnavailable":"0"}}}}'
  done
```

### Disaster Recovery

```bash
# Restore pod to previous state from backup
kubectl patch pod mypod -p "$(cat backup/mypod.json)"

# Restore deployment replicas from config
kubectl patch deployment myapp -p "$(jq -n '{"spec":{"replicas":'$(cat replicas.txt)'}}')"

# Reapply labels after recovery
kubectl get pods -o json | \
  jq -r '.items[] | .metadata.name' | \
  while read name; do
    kubectl patch pod "$name" -p '{"metadata":{"labels":{"recovered":"true"}}}'
  done
```

### Security Hardening

```bash
# Add security context to all pods
kubectl get deployments -o json | \
  jq -r '.items[] | .metadata.name' | \
  while read name; do
    kubectl patch deployment "$name" -p '{"spec":{"template":{"spec":{"securityContext":{"runAsNonRoot":true,"runAsUser":1000}}}}}'
  done

# Add resource limits to all containers
kubectl get deployments -o json | \
  jq -r '.items[] | .metadata.name' | \
  while read name; do
    kubectl patch deployment "$name" -p '{"spec":{"template":{"spec":{"containers":[{"name":"myapp","resources":{"limits":{"cpu":"1","memory":"1Gi"}}}]}}}}'
  done

# Set network policies
kubectl patch namespace default -p '{"metadata":{"labels":{"network-policy":"enabled"}}}'
```

### Configuration Management

```bash
# Update environment variables for all deployments
kubectl get deployments -o json | \
  jq -r '.items[] | .metadata.name' | \
  while read name; do
    kubectl patch deployment "$name" -p '{"spec":{"template":{"spec":{"containers":[{"name":"myapp","env":[{"name":"ENV","value":"production"}]}]}}}}'
  done

# Update ConfigMap and restart deployments
kubectl patch configmap myconfig -p '{"data":{"version":"v2"}}'
kubectl get deployments -l config=myconfig -o json | \
  jq -r '.items[] | .metadata.name' | \
  while read name; do
    kubectl rollout restart deployment "$name"
  done
```

## Tips and Best Practices

### JSONPath Tips

```bash
# Use range for iteration
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'

# Use ?() for filtering
kubectl get pods -o jsonpath='{.items[?(@.status.phase=="Running")].metadata.name}'

# Use @ for current element
kubectl get pods -o jsonpath='{items[0].metadata.name}'

# Use quotes for strings with special characters
kubectl get pods -o jsonpath='{.items[*].metadata.labels."app.kubernetes.io/name"}'

# Combine multiple queries
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'
```

### Patch Tips

```bash
# Use strategic patch for arrays
kubectl patch deployment myapp -p '{"spec":{"template":{"spec":{"containers":[{"name":"myapp","env":[{"name":"VAR","value":"val"}]}]}}}}'

# Use merge patch for simple updates
kubectl patch deployment myapp --type='merge' -p '{"spec":{"replicas":5}}'

# Test patch before applying (dry-run)
kubectl patch deployment myapp --dry-run=client -p '{"spec":{"replicas":5}}'

# Validate patch schema
kubectl patch deployment myapp --dry-run=server -p '{"spec":{"replicas":5}}'

# Use subresource for status updates
kubectl patch deployment myapp --subresource=status -p '{"status":{"replicas":5}}'
```

### Performance Tips

```bash
# Use field selectors with JSONPath for efficiency
kubectl get pods -l app=web -o jsonpath='{.items[*].metadata.name}'

# Limit output with JSONPath
kubectl get pods -o jsonpath='{.items[0:10].metadata.name}'

# Cache JSON output for multiple queries
kubectl get pods -o json > pods.json
jq -r '.items[].metadata.name' pods.json
jq -r '.items[].status.podIP' pods.json

# Use --server-side for large resources
kubectl patch deployment myapp --server-side -p '{"spec":{"replicas":5}}'
```

## Quick Reference

### JSONPath

```bash
# Get specific field
kubectl get pods -o jsonpath='{.items[0].metadata.name}'

# Get all values
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Filter with condition
kubectl get pods -o jsonpath='{.items[?(@.status.phase=="Running")].metadata.name}'

# Iterate over items
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'

# Format output
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
```

### Patch

```bash
# Strategic patch
kubectl patch deployment myapp -p '{"spec":{"replicas":5}}'

# Merge patch
kubectl patch deployment myapp --type='merge' -p '{"spec":{"replicas":5}}'

# Remove field
kubectl patch deployment myapp --type='merge' -p '{"spec":{"template":{"spec":{"containers":[{"name":"myapp","resources":{"limits":null}}]}}}}'

# Patch from file
kubectl patch deployment myapp -p "$(cat patch.json)"

# Dry run
kubectl patch deployment myapp --dry-run=client -p '{"spec":{"replicas":5}}'
```
