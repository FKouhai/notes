---
id: 005-Declarative_Setup_Examples
aliases:
  - argocd-declarative
  - argocd-examples
tags:
  - argocd
  - kubernetes
  - gitops
  - declarative
  - examples
---

# Argo CD Declarative Setup Examples

## Overview

Argo CD follows GitOps principles, meaning all configuration should be managed declaratively through Git repositories. This guide provides practical examples of setting up Argo CD and applications using declarative Kubernetes manifests rather than CLI commands.

## Declarative Application Resources

### Basic Application Manifest

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Application with Helm Chart

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-ingress
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://kubernetes.github.io/ingress-nginx
    chart: ingress-nginx
    targetRevision: 4.8.3
    helm:
      values: |
        controller:
          service:
            type: LoadBalancer
          replicaCount: 2
        admissionWebhooks:
          enabled: false
  destination:
    server: https://kubernetes.default.svc
    namespace: ingress-nginx
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Application with Kustomize

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kustomize-guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: kustomize-guestbook
    kustomize:
      images:
        - name: paulbouwer/hello-kubernetes
          newTag: "1.10"
  destination:
    server: https://kubernetes.default.svc
    namespace: kustomize-guestbook
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Application with Values from ConfigMap

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: helm-values
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://charts.bitnami.com/bitnami
    chart: nginx
    targetRevision: 15.4.1
    helm:
      values: |
        image:
          registry: docker.io
          repository: bitnami/nginx
          tag: 1.25.3-debian-11-r10
        service:
          type: ClusterIP
        replicaCount: 1
  destination:
    server: https://kubernetes.default.svc
    namespace: nginx
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Application with Multiple Sources

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: multi-source-app
  namespace: argocd
spec:
  project: default
  sources:
    - repoURL: https://github.com/your-org/base-configs.git
      path: base
      targetRevision: main
    - repoURL: https://charts.bitnami.com/bitnami
      chart: redis
      targetRevision: 18.6.1
      helm:
        values: |
          architecture: standalone
          auth:
            enabled: true
            password: redis123
  destination:
    server: https://kubernetes.default.svc
    namespace: multi-source
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## ApplicationSet for Automated App Creation

### Directory Generator

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/argoproj/argocd-example-apps.git
        revision: HEAD
        directories:
          - path: guestbook/*
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj/argocd-example-apps.git
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

### List Generator

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: clusters
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - cluster: cluster1
            url: https://kubernetes.default.svc
          - cluster: cluster2
            url: https://cluster2.kubernetes.default.svc
  template:
    metadata:
      name: '{{cluster}}-guestbook'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj/argocd-example-apps.git
        targetRevision: HEAD
        path: guestbook
      destination:
        server: '{{url}}'
        namespace: guestbook
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

### Cluster Generator

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: multi-cluster
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            env: prod
  template:
    metadata:
      name: '{{name}}-nginx'
    spec:
      project: default
      source:
        repoURL: https://github.com/your-org/manifests.git
        targetRevision: main
        path: nginx
      destination:
        server: '{{server}}'
        namespace: nginx
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

### Git Directory Generator with Values

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: apps
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/your-org/apps.git
        revision: main
        directories:
          - path: apps/*
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/your-org/apps.git
        targetRevision: main
        path: '{{path}}'
        helm:
          valueFiles:
            - values-{{path.basename}}.yaml
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

## Project Configuration

### Basic Project

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: my-project
  namespace: argocd
spec:
  description: My custom project
  sourceRepos:
    - '*'
  destinations:
    - namespace: '*'
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
```

### Restricted Project

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: restricted-project
  namespace: argocd
spec:
  description: Restricted project with limited access
  sourceRepos:
    - https://github.com/your-org/approved-repos.git
  destinations:
    - namespace: prod
      server: https://kubernetes.default.svc
    - namespace: staging
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: 'rbac.authorization.k8s.io'
      kind: 'Role'
    - group: 'rbac.authorization.k8s.io'
      kind: 'RoleBinding'
    - group: 'rbac.authorization.k8s.io'
      kind: 'ClusterRole'
  namespaceResourceWhitelist:
    - group: 'apps'
      kind: 'Deployment'
    - group: 'apps'
      kind: 'StatefulSet'
    - group: ''
      kind: 'Service'
    - group: 'networking.k8s.io'
      kind: 'Ingress'
  roles:
    - name: read-only
      description: Read-only access
      policies:
        - p, proj:restricted-project:read-only, applications, get, restricted-project/*, allow
      groups:
        - my-oidc-group:read-only
    - name: admin
      description: Full admin access
      policies:
        - p, proj:restricted-project:admin, applications, *, restricted-project/*, allow
      groups:
        - my-oidc-group:admins
```

## Repository Configuration

### Public Git Repository

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  url: https://github.com/argoproj/argocd-example-apps.git
```

### Private Git Repository with SSH

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: private-git-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  url: git@github.com:your-org/private-repo.git
  sshPrivateKey: |
    -----BEGIN RSA PRIVATE KEY-----
    MIIEpAIBAAKCAQEA...
    -----END RSA PRIVATE KEY-----
```

### Private Git Repository with HTTPS and Token

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: git-token-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  url: https://github.com/your-org/private-repo.git
  password: <github-personal-access-token>
  username: <github-username>
```

### Helm Repository

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: helm-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  url: https://charts.bitnami.com/bitnami
  name: bitnami
  type: helm
```

## Complete Example: Full Stack Application

### Application Manifest

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: webapp-stack
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/infrastructure.git
    targetRevision: main
    path: apps/webapp
    helm:
      releaseName: webapp
      valueFiles:
        - values.yaml
        - values-prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: webapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### Values Files

```yaml
# values.yaml
replicaCount: 2

image:
  repository: your-registry/webapp
  pullPolicy: IfNotPresent
  tag: "latest"

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: webapp.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: webapp-tls
      hosts:
        - webapp.example.com

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
  targetMemoryUtilizationPercentage: 80
```

```yaml
# values-prod.yaml
image:
  tag: "v1.2.3"

resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi

autoscaling:
  minReplicas: 4
  maxReplicas: 20

monitoring:
  enabled: true
  serviceMonitor:
    enabled: true
    namespace: monitoring
```

### Kubernetes Manifest Structure

```
apps/webapp/
├── Chart.yaml
├── values.yaml
├── values-prod.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── serviceaccount.yaml
    ├── hpa.yaml
    └── configmap.yaml
```

## Advanced Examples

### Application with Pre-Sync and Post-Sync Hooks

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-with-hooks
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/app.git
    targetRevision: main
    path: manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: app-with-hooks
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
    automated:
      prune: true
      selfHeal: true
    retry:
      limit: 2
      backoff:
        duration: 5s
        maxDuration: 3m
        factor: 2
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

### Hook Manifest Example

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: pre-sync-job-
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: pre-sync
        image: busybox
        command: ["/bin/sh", "-c", "echo Pre-sync hook executed"]
      restartPolicy: Never
```

### Application with Sync Waves

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-with-waves
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/app.git
    targetRevision: main
    path: manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: app-with-waves
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
    syncWave: 0
```

### Wave Manifest Example

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
data:
  config.yaml: |
    app: version-1
---
apiVersion: v1
kind: Service
metadata:
  name: app-service
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
spec:
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: myapp
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: app
          image: myapp:latest
          ports:
            - containerPort: 8080
```

## Managing Secrets

### SealedSecrets Example

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-with-secrets
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/sealed-secrets.git
    targetRevision: main
    path: sealed-secrets
  destination:
    server: https://kubernetes.default.svc
    namespace: app-with-secrets
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

SealedSecret manifest:
```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: my-secret
  namespace: app-with-secrets
spec:
  encryptedData:
    password: AgBy3i4OJSWK+PiTySY...
    username: AgBy3i4OJSWK+PiTySY...
  template:
    type: Opaque
```

These declarative examples demonstrate how to implement GitOps principles with Argo CD, managing all configuration through version-controlled manifests rather than imperative CLI commands.