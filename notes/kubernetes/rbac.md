---
id: kubernetes-rbac
title: Kubernetes RBAC (Role-Based Access Control)
aliases:
  - rbac
  - access-control
  - permissions
tags:
  - kubernetes
  - rbac
  - security
  - access-control
description: Comprehensive guide to Kubernetes RBAC with detailed examples and best practices
---

# Kubernetes RBAC (Role-Based Access Control)

RBAC manages access control through Roles and ClusterRoles, determining who can do what on which resources.

## RBAC Basics

### What is RBAC?

RBAC (Role-Based Access Control) is a method of regulating access to computer or network resources based on the roles of individual users within your organization.

**Key Concepts**:
- **Role**: Defines permissions within a namespace
- **ClusterRole**: Defines permissions cluster-wide
- **RoleBinding**: Grants permissions to users/groups/service accounts within a namespace
- **ClusterRoleBinding**: Grants permissions cluster-wide
- **Subject**: Who receives the permissions (user, group, or service account)
- **Resource**: What the permissions apply to (pods, deployments, etc.)
- **Verb**: What action can be performed (get, create, delete, etc.)

### RBAC Architecture

```
User/Service Account
       |
       v
  RoleBinding/ClusterRoleBinding
       |
       v
    Role/ClusterRole
       |
       v
    API Resources
```

### RBAC Object Relationships

| Object | Scope | Purpose |
|--------|-------|---------|
| Role | Namespace | Define permissions in a namespace |
| ClusterRole | Cluster-wide | Define permissions cluster-wide |
| RoleBinding | Namespace | Bind Role to subjects in namespace |
| ClusterRoleBinding | Cluster-wide | Bind ClusterRole to subjects cluster-wide |

### Subjects

Subjects can be:

1. **User**: External users (not stored in Kubernetes)
2. **Group**: External groups of users (not stored in Kubernetes)
3. **ServiceAccount**: Kubernetes service accounts (stored in Kubernetes)

## Roles

### Role (Namespace-Scoped)

Roles define permissions within a specific namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

**Role Components**:
- `apiGroups`: List of API groups ("" for core, "apps" for deployments, etc.)
- `resources`: List of resource types (pods, services, etc.)
- `verbs`: List of allowed operations (get, list, watch, create, update, etc.)

### Multiple Resources

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: resource-manager
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update"]
```

### Wildcard Resources and Verbs

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: full-access
  namespace: default
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

**Warning**: Use wildcards cautiously in production.

### Resource Names

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-editor
  namespace: default
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["my-config", "app-config"]
  verbs: ["get", "update", "patch"]
```

**Behavior**:
- Applies only to specific configmap names
- Cannot create new configmaps with these names
- Useful for fine-grained control

### Non-Resource URLs

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: view-healthz
rules:
- nonResourceURLs: ["/healthz", "/healthz/*"]
  verbs: ["get"]
```

**Behavior**:
- Applies to non-resource endpoints (like healthz)
- Only works with ClusterRole
- Useful for API access control

## ClusterRoles

### ClusterRole (Cluster-Scoped)

ClusterRoles define permissions cluster-wide, not bound to any namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```

**ClusterRole Use Cases**:
- Cluster-wide permissions (nodes, persistent volumes, etc.)
- Permissions for resources not namespaced
- Reusable roles across namespaces via RoleBinding

### ClusterRole for Namespace Resources

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

**Behavior**:
- Can be used with RoleBinding to grant access to pods in any namespace
- Can be used with ClusterRoleBinding to grant access to pods in all namespaces

### Aggregated ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-monitoring: "true"
rules: []
```

**Behavior**:
- Aggregates permissions from all ClusterRoles with matching labels
- Rules are automatically populated from matching ClusterRoles
- Useful for combining permissions from multiple sources

## Bindings

### RoleBinding

RoleBindings grant permissions defined in a Role to subjects within a namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**RoleBinding Components**:
- `subjects`: Who receives the permissions
- `roleRef`: Which Role to bind (required)

### Multiple Subjects

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-access
  namespace: default
subjects:
- kind: User
  name: alice
  apiGroup: rbac.authorization.k8s.io
- kind: User
  name: bob
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRoleBinding

ClusterRoleBindings grant permissions defined in a ClusterRole to subjects cluster-wide.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: User
  name: alice
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

**Behavior**:
- Grants permissions across all namespaces
- Use with caution in production
- Useful for cluster administrators

### Using ClusterRole with RoleBinding

```yaml
# ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]

---
# RoleBinding (namespace-scoped)
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
- kind: ServiceAccount
  name: monitoring
  namespace: production
roleRef:
  kind: ClusterRole
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**Behavior**:
- Grants pod-reader permissions to monitoring service account
- Only in production namespace
- Useful for reusing ClusterRoles

## Service Accounts

### Service Account Creation

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: deployer
  namespace: production
```

### Service Account with Role

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: deployer
  namespace: production

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: production
rules:
- apiGroups: ["apps", "extensions"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deployer-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: deployer
  namespace: production
roleRef:
  kind: Role
  name: deployer
  apiGroup: rbac.authorization.k8s.io
```

### Using Service Account in Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: production
spec:
  serviceAccountName: deployer
  containers:
  - name: app
    image: myapp:latest
```

**Behavior**:
- Pod uses deployer service account
- Pod has deployer's permissions
- Useful for application-specific access

### Automount Service Account Token

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp
automountServiceAccountToken: false  # Disable auto-mount

---
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  serviceAccountName: myapp
  automountServiceAccountToken: false  # Disable for this pod
  containers:
  - name: app
    image: myapp:latest
```

**Security Best Practice**:
- Disable token mounting if not needed
- Reduces attack surface
- Required for Pod Security Standards

## Common RBAC Patterns

### Read-Only Access

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: read-only
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch"]
```

### Deploy/Update Access

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: default
rules:
- apiGroups: ["apps", "extensions"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

### Admin Access

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: admin
  namespace: production
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

### View Access

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: view
  namespace: production
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
```

### Edit Access

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: edit
  namespace: production
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

## Built-in Roles

Kubernetes provides default cluster roles:

### Cluster-Admin

```yaml
# Built-in ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-admin
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
- nonResourceURLs: ["*"]
  verbs: ["*"]
```

**Access**: Full cluster access, use with extreme caution.

### Admin

```yaml
# Built-in ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: admin
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

**Access**: Full namespace access when used with RoleBinding.

### Edit

```yaml
# Built-in ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: edit
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

**Access**: Namespace access without secrets or RBAC modification.

### View

```yaml
# Built-in ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: view
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
```

**Access**: Read-only namespace access.

## Practical Examples

### Application Service Account

```yaml
# Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: webapp
  namespace: production

---
# Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: webapp-role
  namespace: production
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: webapp-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: webapp
  namespace: production
roleRef:
  kind: Role
  name: webapp-role
  apiGroup: rbac.authorization.k8s.io

---
# Deployment using ServiceAccount
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      serviceAccountName: webapp
      containers:
      - name: webapp
        image: webapp:latest
```

### Namespace Creator

```yaml
# ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: namespace-creator
rules:
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["create"]

---
# ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: create-namespaces
subjects:
- kind: User
  name: namespace-creator
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: namespace-creator
  apiGroup: rbac.authorization.k8s.io
```

### CI/CD Pipeline Service Account

```yaml
# Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cicd
  namespace: production

---
# Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cicd-role
  namespace: production
rules:
- apiGroups: ["apps", "extensions"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch", "delete"]

---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cicd-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: cicd
  namespace: production
roleRef:
  kind: Role
  name: cicd-role
  apiGroup: rbac.authorization.k8s.io
```

### Monitoring Service Account

```yaml
# Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring

---
# ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources: ["nodes", "nodes/metrics", "pods", "services"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]

---
# ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-binding
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: prometheus
  apiGroup: rbac.authorization.k8s.io
```

## Best Practices

### RBAC Best Practices

1. **Principle of Least Privilege**
   ```yaml
   # Grant only necessary permissions
   rules:
   - apiGroups: [""]
     resources: ["configmaps"]
     verbs: ["get"]  # Not "get", "list", "watch"
   ```

2. **Use Service Accounts, Not Users**
   ```yaml
   # Service accounts are Kubernetes-native
   subjects:
   - kind: ServiceAccount
     name: myapp
   ```

3. **Namespace Isolation**
   ```yaml
   # Use RoleBinding, not ClusterRoleBinding
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
     namespace: production
   ```

4. **Disable Token Mounting When Not Needed**
   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: myapp
   automountServiceAccountToken: false
   ```

5. **Use Role for Namespace Resources**
   ```yaml
   # Role for namespace-scoped resources
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     namespace: production
   ```

6. **Use ClusterRole for Cluster Resources**
   ```yaml
   # ClusterRole for cluster-wide resources
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRole
   rules:
   - apiGroups: [""]
     resources: ["nodes"]
     verbs: ["get", "list"]
   ```

7. **Reuse ClusterRoles with RoleBindings**
   ```yaml
   # Reuse ClusterRole in specific namespace
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
     namespace: production
   roleRef:
     kind: ClusterRole
     name: pod-reader
   ```

8. **Regularly Audit RBAC**
   ```bash
   # List all bindings
   kubectl get rolebindings,clusterrolebindings --all-namespaces

   # Check who can create pods
   kubectl get rolebindings --all-namespaces -o json | \
     jq '.items[] | select(.roleRef.kind=="Role") | 
     {namespace: .metadata.namespace, role: .roleRef.name, subjects: .subjects}'

   # Check cluster-admin access
   kubectl get clusterrolebinding | grep cluster-admin
   ```

9. **Document RBAC Policies**
   ```yaml
   metadata:
     name: webapp-role
     annotations:
       description: "Permissions for web application"
       owner: "team-web@example.com"
       last-audit: "2024-01-15"
   ```

10. **Use Built-in Roles When Possible**
    ```yaml
    # Use built-in roles for standard patterns
    roleRef:
      kind: ClusterRole
      name: view  # Built-in read-only role
    ```

## Troubleshooting

### Verify Permissions

```bash
# Check if user can create pods
kubectl auth can-i create pods --as alice --namespace production

# Check all permissions
kubectl auth can-i --list --as alice --namespace production

# Check service account permissions
kubectl auth can-i list pods --as system:serviceaccount:production:myapp
```

### List RBAC Resources

```bash
# List all roles
kubectl get roles --all-namespaces

# List all cluster roles
kubectl get clusterroles

# List all role bindings
kubectl get rolebindings --all-namespaces

# List all cluster role bindings
kubectl get clusterrolebindings

# List service accounts
kubectl get serviceaccounts --all-namespaces
```

### Describe RBAC Resources

```bash
# Describe role
kubectl describe role pod-reader -n default

# Describe cluster role
kubectl describe clusterrole view

# Describe role binding
kubectl describe rolebinding read-pods -n default

# Describe cluster role binding
kubectl describe clusterrolebinding read-secrets-global
```

### Check Service Account Token

```bash
# Get service account token
kubectl get secret -n $(kubectl get sa myapp -o jsonpath='{.metadata.namespace}') \
  $(kubectl get sa myapp -o jsonpath='{.secrets[0].name}') \
  -o jsonpath='{.data.token}' | base64 -d

# Check pod's service account
kubectl get pod mypod -o jsonpath='{.spec.serviceAccountName}'
```

### Common Issues

**Access Denied**:
```bash
# Verify user/group exists
kubectl get configmap kube-root-ca.crt -n kube-system

# Check if binding exists
kubectl get rolebinding -n production

# Verify binding subjects
kubectl describe rolebinding <name> -n production
```

**Permission Not Applied**:
```bash
# Check role rules
kubectl describe role <name> -n production

# Verify role is correct
kubectl get role <name> -n production -o yaml

# Check binding references
kubectl describe rolebinding <name> -n production
```

**Service Account Not Working**:
```bash
# Verify service account exists
kubectl get sa myapp -n production

# Check pod uses correct service account
kubectl get pod mypod -o jsonpath='{.spec.serviceAccountName}'

# Verify binding for service account
kubectl get rolebinding -n production -o yaml | grep -A 5 myapp
```

## Advanced Features

### Role with Conditions (Alpha)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: conditional-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get"]
  conditions:
  - type: "ResourceAttribute"
    key: "metadata.labels.app"
    operator: "Equals"
    value: "web"
```

**Behavior**:
- Applies only to pods with label `app=web`
- Requires RBAC v1.22+ with feature gate enabled
- Useful for fine-grained control

### SubjectReview API

```bash
# Review permissions for subject
kubectl create clusterrolebinding review-binding \
  --clusterrole=view \
  --user=alice

# Use SubjectAccessReview API
kubectl auth can-i list pods --as alice

# Use SelfSubjectAccessReview API
kubectl auth can-i list pods
```

### Aggregation Example

```yaml
# First ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: metrics-reader
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
rules:
- apiGroups: [""]
  resources: ["nodes/metrics", "pods/metrics"]
  verbs: ["get", "list"]

---
# Second ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: logs-reader
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
rules:
- apiGroups: [""]
  resources: ["pods/log", "nodes/log"]
  verbs: ["get", "list"]

---
# Aggregated ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-monitoring: "true"
rules: []
```

**Behavior**:
- Monitoring ClusterRole aggregates from metrics-reader and logs-reader
- Rules are automatically populated
- Useful for combining permissions from multiple sources

## Quick Reference

### Role Structure

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: role-name
  namespace: namespace-name
rules:
- apiGroups: [""]  # Core API
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]  # Apps API
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
```

### ClusterRole Structure

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: clusterrole-name
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/healthz"]
  verbs: ["get"]
```

### RoleBinding Structure

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: binding-name
  namespace: namespace-name
subjects:
- kind: User
  name: username
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: groupname
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: serviceaccount-name
  namespace: namespace-name
roleRef:
  kind: Role
  name: role-name
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRoleBinding Structure

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: binding-name
subjects:
- kind: User
  name: username
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: clusterrole-name
  apiGroup: rbac.authorization.k8s.io
```

### Common Verbs

| Verb | Description |
|------|-------------|
| get | Get individual resource |
| list | List resources of a type |
| watch | Watch for changes |
| create | Create a resource |
| update | Update a resource |
| patch | Patch a resource |
| delete | Delete a resource |
| deletecollection | Delete collection of resources |

### Commands

```bash
# Create role/clusterrole
kubectl apply -f role.yaml
kubectl apply -f clusterrole.yaml

# Create binding
kubectl apply -f rolebinding.yaml
kubectl apply -f clusterrolebinding.yaml

# List resources
kubectl get roles -n namespace
kubectl get clusterroles
kubectl get rolebindings -n namespace
kubectl get clusterrolebindings

# Check permissions
kubectl auth can-i get pods -n namespace --as username
kubectl auth can-i --list -n namespace --as username

# Describe resources
kubectl describe role rolename -n namespace
kubectl describe clusterrole clusterrolename
kubectl describe rolebinding bindingname -n namespace
kubectl describe clusterrolebinding bindingname
```
