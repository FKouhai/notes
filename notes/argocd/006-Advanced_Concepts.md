---
id: 006-Advanced_Concepts
aliases:
  - argocd-advanced
  - argocd-scale
  - argocd-strategies
tags:
  - argocd
  - kubernetes
  - gitops
  - advanced
  - scale
  - strategies
---

# Argo CD Advanced Concepts

## Sync Waves

### Understanding Sync Waves

Sync waves in Argo CD allow you to control the order in which resources are synchronized during a deployment operation. Resources are synchronized in ascending order based on their sync wave annotation value. Resources without a sync wave annotation default to wave 0.

### How Sync Waves Work

1. Argo CD sorts all resources by their sync wave value
2. Resources with negative sync waves are synchronized first
3. Resources with sync wave 0 (default) are synchronized next
4. Resources with positive sync waves are synchronized last
5. Within each wave, resources are synchronized alphabetically

### Sync Wave Examples

#### Basic Database-App Pattern
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  annotations:
    argocd.argoproj.io/sync-wave: "-3"
data:
  password: cGFzc3dvcmQ=
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-pvc
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 10Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
        - name: postgres
          image: postgres:14
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
          volumeMounts:
            - name: db-storage
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: db-storage
          persistentVolumeClaim:
            claimName: db-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: database-service
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  selector:
    app: database
  ports:
    - port: 5432
      targetPort: 5432
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  annotations:
    argocd.argoproj.io/sync-wave: "1"
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
      containers:
        - name: webapp
          image: myapp:latest
          env:
            - name: DB_HOST
              value: database-service
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  selector:
    app: webapp
  ports:
    - port: 80
      targetPort: 8080
```

#### Microservices with Complex Dependencies
```yaml
# Wave -3: Infrastructure and Secrets
apiVersion: v1
kind: ConfigMap
metadata:
  name: global-config
  annotations:
    argocd.argoproj.io/sync-wave: "-3"
data:
  environment: production
---
apiVersion: v1
kind: Secret
metadata:
  name: api-keys
  annotations:
    argocd.argoproj.io/sync-wave: "-3"
type: Opaque
data:
  api-key: c2VjcmV0LWtleQ==

# Wave -2: Shared Services
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
spec:
  selector:
    app: redis
  ports:
    - port: 6379

# Wave 0: Core Services
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: auth-service
  template:
    metadata:
      labels:
        app: auth-service
    spec:
      containers:
        - name: auth
          image: auth-service:latest
          env:
            - name: REDIS_HOST
              value: redis
            - name: CONFIG_MAP
              value: global-config
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
        - name: users
          image: user-service:latest
          env:
            - name: REDIS_HOST
              value: redis
            - name: AUTH_SERVICE
              value: auth-service
---
apiVersion: v1
kind: Service
metadata:
  name: auth-service
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  selector:
    app: auth-service
  ports:
    - port: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  selector:
    app: user-service
  ports:
    - port: 8080

# Wave 2: Edge Services and API Gateway
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
        - name: gateway
          image: gateway:latest
          env:
            - name: AUTH_SERVICE
              value: auth-service
            - name: USER_SERVICE
              value: user-service
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    argocd.argoproj.io/sync-wave: "3"
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-gateway
                port:
                  number: 8080
```

### Sync Wave Best Practices

1. **Wave Planning**: Plan your waves carefully to avoid circular dependencies
2. **Health Checks**: Use readiness probes to ensure services are ready before dependent services start
3. **Separation of Concerns**: Group related resources together with similar wave numbers
4. **Documentation**: Document your wave strategy for team understanding
5. **Testing**: Test wave order in staging before production deployment

## Deployment Strategies

### Rolling Update Strategy

The default Kubernetes deployment strategy that gradually replaces old pods with new ones.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Number of additional pods allowed during update
      maxUnavailable: 1  # Number of pods that can be unavailable during update
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: webapp
          image: webapp:v2
```

### Blue-Green Deployment with Argo CD

Create two separate environments and switch traffic between them.

```yaml
# Application Set for Blue-Green
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: blue-green-app
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - environment: blue
            color: "blue"
            weight: 100
          - environment: green
            color: "green"
            weight: 0
  template:
    metadata:
      name: webapp-{{environment}}
    spec:
      project: default
      source:
        repoURL: https://github.com/your-org/webapp.git
        targetRevision: main
        path: manifests
        helm:
          values: |
            replicaCount: 3
            environment: {{environment}}
            color: {{color}}
      destination:
        server: https://kubernetes.default.svc
        namespace: webapp-{{environment}}
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true

# Service for traffic routing (updated manually or via automation)
apiVersion: v1
kind: Service
metadata:
  name: webapp-active
  namespace: webapp-blue-green
spec:
  selector:
    app: webapp
    color: blue  # Change this to "green" to switch traffic
  ports:
    - port: 80
      targetPort: 8080
```

### Canary Deployment Strategy

Gradually roll out new versions to a subset of users.

```yaml
# Canary and Stable Deployments
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-stable
  namespace: production
spec:
  replicas: 9
  selector:
    matchLabels:
      app: webapp
      version: stable
  template:
    metadata:
      labels:
        app: webapp
        version: stable
    spec:
      containers:
        - name: webapp
          image: webapp:v1
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-canary
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp
      version: canary
  template:
    metadata:
      labels:
        app: webapp
        version: canary
    spec:
      containers:
        - name: webapp
          image: webapp:v2
---
apiVersion: v1
kind: Service
metadata:
  name: webapp
  namespace: production
spec:
  selector:
    app: webapp
  ports:
    - port: 80
      targetPort: 8080
```

### A/B Testing Strategy

Deploy multiple versions simultaneously and route traffic based on user attributes.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "X-Canary-Test"
    nginx.ingress.kubernetes.io/canary-by-header-value: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  rules:
    - host: webapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: webapp-canary
                port:
                  number: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-canary
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp
      variant: canary
  template:
    metadata:
      labels:
        app: webapp
        variant: canary
    spec:
      containers:
        - name: webapp
          image: webapp:v2-test
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-main
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
      variant: main
  template:
    metadata:
      labels:
        app: webapp
        variant: main
    spec:
      containers:
        - name: webapp
          image: webapp:v1
```

### Progressive Delivery with Argo Rollouts

```yaml
apiVersion: argoproj.io/v1
kind: Rollout
metadata:
  name: webapp
  namespace: production
spec:
  replicas: 5
  strategy:
    canary:
      steps:
        - setWeight: 20
        - pause: {duration: 10m}
        - setWeight: 40
        - pause: {duration: 10m}
        - setWeight: 60
        - pause: {duration: 10m}
        - setWeight: 80
        - pause: {duration: 10m}
      analysis:
        templates:
          - templateName: success-rate
        args:
          - name: service-name
            value: webapp
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: webapp
          image: webapp:v2
          ports:
            - containerPort: 8080
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: production
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 5m
      count: 3
      successCondition: result[0] >= 0.95
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus-server.monitoring.svc.cluster.local
          query: |
            sum(rate(http_requests_total{service="{{args.service-name}}",status!~"5.."}[5m])) /
            sum(rate(http_requests_total{service="{{args.service-name}}"}[5m]))
```

## Running Argo CD at Scale

### Architecture Considerations

#### High Availability Setup

For production environments running Argo CD at scale, use the HA deployment mode:

```yaml
# HA deployment values for Helm
redis-ha:
  enabled: true
  persistentVolume:
    enabled: true
  exporter:
    enabled: true

controller:
  replicas: 2
  resources:
    limits:
      cpu: 2000m
      memory: 2Gi
    requests:
      cpu: 1000m
      memory: 1Gi

server:
  replicas: 2
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 5
  resources:
    limits:
      cpu: 1000m
      memory: 1Gi
    requests:
      cpu: 500m
      memory: 512Mi

repoServer:
  replicas: 2
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 5
  resources:
    limits:
      cpu: 1000m
      memory: 1Gi
    requests:
      cpu: 500m
      memory: 512Mi

applicationSet:
  replicas: 2
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 250m
      memory: 256Mi
```

#### Multi-Cluster Architecture

```yaml
# ApplicationSet for multi-cluster deployment
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: multi-cluster-apps
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            environment: production
  template:
    metadata:
      name: '{{name}}-webapp'
      labels:
        app: webapp
    spec:
      project: production
      source:
        repoURL: https://github.com/your-org/webapp.git
        targetRevision: main
        path: manifests
        helm:
          values: |
            environment: {{metadata.labels.environment}}
            cluster: {{name}}
            region: {{metadata.labels.region}}
      destination:
        server: '{{server}}'
        namespace: webapp
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

### Performance Optimization

#### Controller Configuration

```yaml
# Application Controller Configuration
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-application-controller
  namespace: argocd
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: argocd-application-controller
          env:
            - name: ARGOCD_RECONCILIATION_TIMEOUT
              value: "180s"
            - name: ARGOCD_REPO_SERVER_TIMEOUT_SECONDS
              value: "60"
            - name: ARGOCD_MAX_SYNC_CONCURRENCY
              value: "20"
            - name: ARGOCD_CONTROLLER_RESOURCE_CUSTOMIZATIONS
              value: |
                - replicas:
                    js: |
                      if (resource.kind === 'Deployment') {
                        return 10;
                      }
```

#### Repository Server Optimization

```yaml
# Repository Server Configuration
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-repo-server
  namespace: argocd
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: argocd-repo-server
          env:
            - name: ARGOCD_REPOTIMEOUT
              value: "180s"
            - name: ARGOCD_REPOMOUNT
              value: "200"
            - name: ARGOCD_MAX_REPOS
              value: "500"
          resources:
            limits:
              cpu: 2000m
              memory: 2Gi
            requests:
              cpu: 1000m
              memory: 1Gi
          volumeMounts:
            - name: repo-cache
              mountPath: /app/argo-cd
      volumes:
        - name: repo-cache
          persistentVolumeClaim:
            claimName: argocd-repo-server-cache
```

#### Redis Configuration

```yaml
# Redis HA Configuration
redis-ha:
  enabled: true
  persistentVolume:
    enabled: true
    size: 10Gi
  configmap:
    redis:
      maxmemory-policy: allkeys-lru
      timeout: 0
      tcp-keepalive: 300
    redisSentinel:
      down-after-milliseconds: 5000
      failover-timeout: 10000
      parallel-syncs: 1
  resources:
    limits:
      cpu: 500m
      memory: 1Gi
    requests:
      cpu: 250m
      memory: 512Mi
```

### Multi-Tenant Architecture

#### Tenant Isolation

```yaml
# Project for each tenant
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: tenant-a
  namespace: argocd
spec:
  description: Project for Tenant A
  sourceRepos:
    - https://github.com/tenant-a/apps.git
  destinations:
    - namespace: tenant-a-*
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: rbac.authorization.k8s.io
      kind: Role
    - group: rbac.authorization.k8s.io
      kind: RoleBinding
  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
  roles:
    - name: tenant-a-admin
      policies:
        - p, proj:tenant-a:tenant-a-admin, applications, *, tenant-a/*, allow
      groups:
        - tenant-a-admin-group
    - name: tenant-a-developer
      policies:
        - p, proj:tenant-a:tenant-a-developer, applications, get, tenant-a/*, allow
        - p, proj:tenant-a:tenant-a-developer, applications, sync, tenant-a/*, allow
      groups:
        - tenant-a-developer-group
```

#### Namespace Restrictions

```yaml
# Application with namespace restrictions
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: tenant-a-webapp
  namespace: argocd
spec:
  project: tenant-a
  source:
    repoURL: https://github.com/tenant-a/apps.git
    targetRevision: main
    path: webapp
  destination:
    server: https://kubernetes.default.svc
    namespace: tenant-a-production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
    managedNamespaceMetadata:
      labels:
        tenant: tenant-a
        environment: production
      annotations:
        owner: tenant-a-team
```

### Monitoring and Observability

#### Prometheus Metrics

```yaml
# ServiceMonitor for Argo CD components
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-metrics
  namespace: argocd
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app.kubernetes.io/part-of: argocd
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

#### Key Metrics to Monitor

```yaml
# Prometheus alerting rules
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: argocd-alerts
  namespace: argocd
spec:
  groups:
    - name: argocd.rules
      rules:
        - alert: ArgoCDApplicationsOutOfSync
          expr: argocd_app_sync_status{phase="Failed"} > 0
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Argo CD application {{ $labels.name }} is out of sync"
            description: "Argo CD application {{ $labels.name }} has been out of sync for more than 5 minutes."
        
        - alert: ArgoCDPodNotReady
          expr: |
            kube_pod_status_ready{namespace="argocd",condition="true"} == 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Argo CD pod {{ $labels.pod }} is not ready"
            description: "Argo CD pod {{ $labels.pod }} has been not ready for more than 5 minutes."
        
        - alert: ArgoCDHighCPUUsage
          expr: |
            rate(container_cpu_usage_seconds_total{namespace="argocd"}[5m]) > 0.8
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Argo CD component {{ $labels.container }} has high CPU usage"
            description: "Argo CD component {{ $labels.container }} has CPU usage above 80% for more than 10 minutes."
```

### GitOps Best Practices at Scale

#### Repository Structure

```
infrastructure/
├── argocd/
│   ├── apps/
│   │   ├── production/
│   │   └── staging/
│   ├── projects/
│   └── clusters/
├── base-configs/
├── tenant-a/
│   ├── services/
│   └── configs/
└── tenant-b/
    ├── services/
    └── configs/
```

#### Application Organization

```yaml
# App of Apps pattern for organized deployments
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: production-apps
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: production
  source:
    repoURL: https://github.com/your-org/infrastructure.git
    targetRevision: main
    path: argocd/apps/production
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

#### Secrets Management

```yaml
# External Secrets Operator integration
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: argocd-repo-server
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets
    kind: SecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
  data:
    - secretKey: password
      remoteRef:
        key: production/db-password
```

### Disaster Recovery

#### Backup Strategy

```yaml
# Backup CronJob for Argo CD configuration
apiVersion: batch/v1
kind: CronJob
metadata:
  name: argocd-backup
  namespace: argocd
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: bitnami/kubectl:latest
              command:
                - /bin/bash
                - -c
                - |
                  kubectl get applications -n argocd -o yaml > /backup/applications.yaml
                  kubectl get appprojects -n argocd -o yaml > /backup/projects.yaml
                  kubectl get clusters -n argocd -o yaml > /backup/clusters.yaml
                  kubectl get cm -n argocd -o yaml > /backup/configmaps.yaml
              volumeMounts:
                - name: backup-volume
                  mountPath: /backup
          volumes:
            - name: backup-volume
              persistentVolumeClaim:
                claimName: argocd-backup
          restartPolicy: OnFailure
```

#### Restore Strategy

```yaml
# Restore Job
apiVersion: batch/v1
kind: Job
metadata:
  name: argocd-restore
  namespace: argocd
spec:
  template:
    spec:
      containers:
        - name: restore
          image: bitnami/kubectl:latest
          command:
            - /bin/bash
            - -c
            - |
              kubectl apply -f /backup/projects.yaml
              kubectl apply -f /backup/clusters.yaml
              kubectl apply -f /backup/applications.yaml
              kubectl apply -f /backup/configmaps.yaml
          volumeMounts:
            - name: backup-volume
              mountPath: /backup
      volumes:
        - name: backup-volume
          persistentVolumeClaim:
            claimName: argocd-backup
      restartPolicy: OnFailure
```

This advanced concepts guide provides the knowledge needed to run Argo CD effectively at scale in production environments, covering sophisticated deployment strategies, performance optimization, and multi-cluster architectures.