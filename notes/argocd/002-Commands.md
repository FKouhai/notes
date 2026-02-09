---
id: 002-Commands
aliases:
  - argocd-commands
tags:
  - argocd
  - kubernetes
  - gitops
  - cli
---

# Argo CD CLI Commands

## Authentication

### Login
```bash
# Login to Argo CD server
argocd login <server-hostname>:443

# Login with username and password
argocd login <server-hostname>:443 --username <username> --password <password>

# Login with sso (if configured)
argocd login <server-hostname>:443 --sso

# Login to insecure Argo CD server
argocd login --grpc-web <server-hostname>:443 --insecure
```

### Account Management
```bash
# Change password
argocd account update-password

# Get current user info
argocd account get-user-info

# List accounts
argocd account list
```

## Application Management

### Create Applications
```bash
# Create app from git repository
argocd app create <app-name> \
  --repo <repo-url> \
  --path <path-to-manifests> \
  --dest-server <cluster-url> \
  --dest-namespace <namespace>

# Create app with helm values file
argocd app create <app-name> \
  --repo <repo-url> \
  --path <chart-path> \
  --dest-server <cluster-url> \
  --dest-namespace <namespace> \
  --helm-chart <chart-name> \
  --helm-values <values.yaml>

# Create app with directory recurse
argocd app create <app-name> \
  --repo <repo-url> \
  --path <path-to-manifests> \
  --dest-server <cluster-url> \
  --dest-namespace <namespace> \
  --directory-recurse
```

### View Applications
```bash
# List all applications
argocd app list

# Get specific application details
argocd app get <app-name>

# Get application history
argocd app history <app-name>

# Show application manifest
argocd app manifests <app-name>

# Show application resources
argocd app resources <app-name>
```

### Sync Applications
```bash
# Sync application
argocd app sync <app-name>

# Sync with dry-run
argocd app sync <app-name> --dry-run

# Sync with specific revision
argocd app sync <app-name> --revision <sha1>

# Sync with prune enabled
argocd app sync <app-name> --prune

# Wait for sync completion
argocd app sync <app-name> --timeout <seconds>

# Force sync even if application is degraded
argocd app sync <app-name> --force
```

### Delete Applications
```bash
# Delete application
argocd app delete <app-name>

# Delete with cascade
argocd app delete <app-name> --cascade
```

## Project Management

### Create Projects
```bash
# Create project with source and destination restrictions
argocd proj create <project-name> \
  -d https://kubernetes.default.svc,<namespace> \
  -s <repo-url>

# Create project from file
argocd proj create -f <project-file.yaml>
```

### View Projects
```bash
# List projects
argocd proj list

# Get project details
argocd proj get <project-name>
```

### Manage Projects
```bash
# Add destination to project
argocd proj add-destination <project-name> <cluster-url> <namespace>

# Remove destination from project
argocd proj remove-destination <project-name> <cluster-url> <namespace>

# Add source to project
argocd proj add-source <project-name> <repo-url>

# Remove source from project
argocd proj remove-source <project-name> <repo-url>
```

## Repository Management

### Add Repositories
```bash
# Add private repository with SSH key
argocd repo add <repo-url> --ssh-private-key-path <key-path>

# Add repository with username and password
argocd repo add <repo-url> --username <username> --password <password>

# Add repository with TLS client certificate
argocd repo add <repo-url> --tls-client-cert-path <cert-path> --tls-client-cert-key-path <key-path>
```

### View Repositories
```bash
# List repositories
argocd repo list

# Get repository details
argocd repo get <repo-url>
```

### Remove Repositories
```bash
# Remove repository
argocd repo rm <repo-url>
```

## Cluster Management

### Add Clusters
```bash
# Add cluster using kubeconfig context
argocd cluster add <context-name>

# Add cluster with specific name
argocd cluster add <context-name> --name <cluster-name>

# Add cluster with namespaces restriction
argocd cluster add <context-name> --namespace <namespace1> --namespace <namespace2>
```

### View Clusters
```bash
# List clusters
argocd cluster list

# Get cluster details
argocd cluster get <cluster-server>
```

### Remove Clusters
```bash
# Remove cluster
argocd cluster rm <cluster-server>
```

## Monitoring and Status

### Application Status
```bash
# Wait for application to reach healthy state
argocd app wait <app-name> --health

# Wait for application to be synced
argocd app wait <app-name> --sync

# Wait for operation completion
argocd app wait <app-name> --operation

# Diff application live vs target state
argocd app diff <app-name>
```

### System Status
```bash
# Check Argo CD version
argocd version

# Get server connection status
argocd version --client=false

# List settings
argocd app list --output json | jq

# Get stats about application sync statuses
argocd app list -o name | xargs -I {} argocd app get {} --refresh
```

## Advanced Operations

### Parameter Overrides
```bash
# Set helm parameter override
argocd app set <app-name> -p <param>=<value>

# Set kustomize image override
argocd app set <app-name> --kustomize-image <image-tag>

# Set json patch
argocd app set <app-name> --jsonnet-ext-var-str <var>=<value>
```

### Refresh Operations
```bash
# Hard refresh application
argocd app get <app-name> --hard-refresh

# Refresh all applications
argocd app list -o name | xargs -I {} argocd app get {} --refresh
```

### Rollback Operations
```bash
# Rollback to previous deployment
argocd app rollback <app-name> <revision>

# Preview rollback
argocd app history <app-name>
```

These commands form the core of Argo CD's operational capabilities, allowing you to manage the complete lifecycle of GitOps deployments through the CLI.