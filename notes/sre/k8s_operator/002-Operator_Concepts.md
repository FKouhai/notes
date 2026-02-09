---
id: 006-Operator_Concepts
aliases:
  - kubernetes-operator-concepts
  - crd-concepts
  - controller-concepts
tags:
  - kubernetes
  - operator
  - controller
  - crd
  - golang
  - sre
---

# Kubernetes Operator Essential Concepts

## Custom Resource Definitions (CRDs)

CRDs extend the Kubernetes API by adding custom resource types.

### CRD Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                    CRD YAML Structure                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  apiVersion: apiextensions.k8s.io/v1                           │
│  kind: CustomResourceDefinition                                 │
│  metadata:                                                      │
│    name: myapps.example.com       # <plural>.<group>            │
│  spec:                                                          │
│    group: example.com                                           │
│    versions:                                                    │
│      - name: v1                                                │
│        served: true                                             │
│        storage: true                                            │
│        schema:                                                  │
│          openAPIV3Schema:                                      │
│            type: object                                        │
│            properties:                                          │
│              spec:                                              │
│                type: object                                    │
│                properties:                                      │
│                  replicas:                                      │
│                    type: integer                                │
│                    minimum: 1                                   │
│                  image:                                         │
│                    type: string                                 │
│                  port:                                          │
│                    type: integer                                │
│                    minimum: 1                                   │
│                    maximum: 65535                               │
│              status:                                            │
│                type: object                                    │
│                properties:                                      │
│                  readyReplicas:                                 │
│                    type: integer                                │
│                  phase:                                         │
│                    type: string                                │
│                    enum: [Pending, Running, Failed]            │
│    scope: Namespaced                                             │
│    names:                                                        │
│      plural: myapps                                             │
│      singular: myapp                                            │
│      kind: MyApp                                                │
│      shortNames:                                                │
│        - ma                                                     │
│    subresources:                                                 │
│      status: {}                                                 │
│    additionalPrinterColumns:                                   │
│      - name: Replicas                                           │
│        type: integer                                            │
│        jsonPath: .spec.replicas                                 │
│      - name: Ready                                               │
│        type: string                                             │
│        jsonPath: .status.phase                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Creating a CRD

```bash
# Apply CRD definition
kubectl apply -f crd.yaml

# Verify CRD is created
kubectl get crd

# Get CRD details
kubectl describe crd myapps.example.com

# List available CRDs
kubectl get crds
```

### CRD Validation

CRDs support OpenAPI v3 validation to ensure data integrity.

```yaml
spec:
  versions:
    - name: v1
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: ["image"]  # Required field
              properties:
                replicas:
                  type: integer
                  minimum: 1        # Validation
                  maximum: 100      # Validation
                  default: 1        # Default value
                image:
                  type: string
                  pattern: '^[a-z0-9]+(-[a-z0-9]+)*:[a-z0-9]+$'  # Regex
                port:
                  type: integer
                  format: int32
                  minimum: 1
                  maximum: 65535
                labels:
                  type: object
                  additionalProperties:
                    type: string
                    pattern: '^[a-z0-9]([\-a-z0-9]*[a-z0-9])?$'
```

### CRD Versions

Support multiple versions with conversion strategies.

```yaml
spec:
  versions:
    - name: v1beta1
      served: true
      storage: false
      deprecated: true
      deprecationWarning: "myapp.example.com/v1beta1 is deprecated"
      schema:
        openAPIV3Schema:
          # v1beta1 schema
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          # v1 schema
  conversion:
    strategy: Webhook
    webhook:
      conversionReviewVersions: ["v1", "v1beta1"]
      clientConfig:
        service:
          name: myapp-webhook
          namespace: myapp-system
```

## Custom Resources (CRs)

Custom Resources are instances of CRDs that represent desired state.

### Creating a CR

```yaml
apiVersion: example.com/v1
kind: MyApp
metadata:
  name: myapp-instance
  namespace: default
  labels:
    app: myapp
    environment: production
spec:
  replicas: 3
  image: myapp:v1.2.0
  port: 8080
  resources:
    requests:
      cpu: "100m"
      memory: "128Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
  env:
    - name: LOG_LEVEL
      value: "info"
    - name: DEBUG
      value: "false"
status:
  readyReplicas: 0
  phase: Pending
  observedGeneration: 0
```

### Managing CRs

```bash
# Create CR
kubectl apply -f myapp-instance.yaml

# Get CR
kubectl get myapps
kubectl get myapp myapp-instance

# Get with wide output
kubectl get myapp myapp-instance -o wide

# Describe CR
kubectl describe myapp myapp-instance

# Edit CR
kubectl edit myapp myapp-instance

# Delete CR
kubectl delete myapp myapp-instance

# Get YAML
kubectl get myapp myapp-instance -o yaml

# Get JSON
kubectl get myapp myapp-instance -o json
```

### Watch CRs

```bash
# Watch for changes
kubectl get myapps --watch

# Watch with labels
kubectl get myapps -l environment=production --watch
```

## Reconciliation Pattern

The core concept of Kubernetes operators is the reconciliation loop.

### Reconciliation Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                   Reconciliation Loop                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Controller receives reconcile request                       │
│     - CR created/updated/deleted                                 │
│     - Related resources changed                                 │
│     - Timer triggered                                           │
│                                                                 │
│     │                                                           │
│     ▼                                                           │
│                                                                 │
│  2. Fetch CR from cache                                         │
│     - Check if CR exists                                        │
│     - Check if deletion timestamp set                           │
│     - Parse spec and status                                     │
│                                                                 │
│     │                                                           │
│     ▼                                                           │
│                                                                 │
│  3. Fetch current state from cluster                            │
│     - List related resources (Deployments, Services, etc.)      │
│     - Compare with desired state                                │
│                                                                 │
│     │                                                           │
│     ▼                                                           │
│                                                                 │
│  4. Determine actions needed                                    │
│     - Create missing resources                                 │
│     - Update mismatched resources                               │
│     - Delete obsolete resources                                 │
│     - Update CR status                                         │
│                                                                 │
│     │                                                           │
│     ▼                                                           │
│                                                                 │
│  5. Execute actions                                             │
│     - Apply changes via client                                 │
│     - Handle errors and retries                                 │
│     - Update status                                             │
│                                                                 │
│     │                                                           │
│     ▼                                                           │
│                                                                 │
│  6. Return result                                               │
│     - Result{Ok, Requeue}: Reconcile again                      │
│     - Result{Ok, Don'tRequeue}: Wait for next event            │
│     - Result{Error}: Requeue with backoff                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Reconciliation Code Example

```go
package controllers

import (
    "context"
    "fmt"

    "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/log"

    appv1 "myapp/api/v1"
)

type MyAppReconciler struct {
    client.Client
    Scheme *runtime.Scheme
    Log    log.Logger
}

func (r *MyAppReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := r.Log.WithValues("myapp", req.NamespacedName)

    // 1. Fetch CR from cache
    var myapp appv1.MyApp
    if err := r.Get(ctx, req.NamespacedName, &myapp); err != nil {
        if errors.IsNotFound(err) {
            // CR was deleted, clean up resources
            logger.Info("MyApp resource not found. Ignoring since object must be deleted")
            return ctrl.Result{}, nil
        }
        // Error reading object, requeue
        logger.Error(err, "Failed to get MyApp")
        return ctrl.Result{}, err
    }

    // 2. Check if CR is being deleted
    if !myapp.DeletionTimestamp.IsZero() {
        // CR is being deleted, handle finalizers
        return r.handleDeletion(ctx, &myapp)
    }

    // 3. Fetch current state
    deployment, err := r.getDeployment(ctx, &myapp)
    if err != nil {
        return ctrl.Result{}, err
    }

    // 4. Determine actions
    if deployment == nil {
        // Create deployment
        logger.Info("Creating deployment")
        if err := r.createDeployment(ctx, &myapp); err != nil {
            return ctrl.Result{}, err
        }
    } else {
        // Update deployment if needed
        if !deploymentEqual(deployment, &myapp) {
            logger.Info("Updating deployment")
            if err := r.updateDeployment(ctx, deployment, &myapp); err != nil {
                return ctrl.Result{}, err
            }
        }
    }

    // 5. Update status
    if err := r.updateStatus(ctx, &myapp); err != nil {
        return ctrl.Result{}, err
    }

    // 6. Requeue periodically for health checks
    return ctrl.Result{RequeueAfter: time.Minute * 5}, nil
}
```

### Idempotency

Reconciliation must be idempotent - safe to run multiple times.

```go
// ✅ Good: Idempotent
func (r *MyAppReconciler) ensureDeployment(ctx context.Context, myapp *appv1.MyApp) error {
    deployment := &appsv1.Deployment{}
    err := r.Get(ctx, client.ObjectKey{
        Name:      myapp.Name,
        Namespace: myapp.Namespace,
    }, deployment)

    if errors.IsNotFound(err) {
        // Create if doesn't exist
        deployment = r.newDeployment(myapp)
        return r.Create(ctx, deployment)
    } else if err != nil {
        return err
    }

    // Update if spec changed
    if !deploymentEqual(deployment, myapp) {
        deployment.Spec = r.deploymentSpec(myapp)
        return r.Update(ctx, deployment)
    }

    return nil
}

// ❌ Bad: Not idempotent (would fail on second run)
func (r *MyAppReconciler) ensureDeployment(ctx context.Context, myapp *appv1.MyApp) error {
    deployment := r.newDeployment(myapp)
    return r.Create(ctx, deployment)  // Fails if already exists
}
```

## Informers and Caches

Informers watch resources and maintain local caches for efficient access.

### Informer Components

```
┌─────────────────────────────────────────────────────────────────┐
│                     Informer Components                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  API Server                                                       │
│      │                                                           │
│      │ Watch (stream)                                            │
│      │ ADD, UPDATE, DELETE events                                │
│      ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Controller Informer                                     │  │
│  │  ┌─────────────────────────────────────────────────┐   │  │
│  │  │  1. Establish watch with API Server              │   │  │
│  │  │  2. Handle watch events                          │   │  │
│  │  │  3. Update cache                                  │   │  │
│  │  │  4. Distribute to event handlers                │   │  │
│  │  └─────────────────────────────────────────────────┘   │  │
│  │             │                                             │  │
│  │             ▼                                             │  │
│  │  ┌─────────────────────────────────────────────────┐   │  │
│  │  │              Local Cache                         │   │  │
│  │  │  - Thread-safe map                              │   │  │
│  │  │  - Fast reads (no API calls)                    │   │  │
│  │  │  - Automatic updates from watch                 │   │  │
│  │  │  - Indexes for queries                          │   │  │
│  │  └─────────────────────────────────────────────────┘   │  │
│  │             │                                             │  │
│  │             │ Event handlers                             │  │
│  │             ▼                                             │  │
│  │  ┌─────────────────────────────────────────────────┐   │  │
│  │  │  OnAdd(obj)    - When object added               │   │  │
│  │  │  OnUpdate(old, new) - When object updated       │   │  │
│  │  │  OnDelete(obj) - When object deleted             │   │  │
│  │  └─────────────────────────────────────────────────┘   │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Using Informers

```go
package main

import (
    "time"

    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/informers"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/tools/cache"
)

func setupInformer(clientset *kubernetes.Clientset, stopCh <-chan struct{}) {
    // Create shared informer factory
    factory := informers.NewSharedInformerFactory(clientset, time.Minute)

    // Get informer for pods
    podInformer := factory.Core().V1().Pods().Informer()

    // Add event handlers
    podInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            pod := obj.(*corev1.Pod)
            log.Printf("Pod added: %s/%s", pod.Namespace, pod.Name)
        },
        UpdateFunc: func(oldObj, newObj interface{}) {
            oldPod := oldObj.(*corev1.Pod)
            newPod := newObj.(*corev1.Pod)
            log.Printf("Pod updated: %s/%s", newPod.Namespace, newPod.Name)
            log.Printf("Old phase: %s, New phase: %s", oldPod.Status.Phase, newPod.Status.Phase)
        },
        DeleteFunc: func(obj interface{}) {
            pod := obj.(*corev1.Pod)
            log.Printf("Pod deleted: %s/%s", pod.Namespace, pod.Name)
        },
    })

    // Start informers
    factory.Start(stopCh)

    // Wait for cache sync
    if !cache.WaitForCacheSync(stopCh, podInformer.HasSynced) {
        log.Fatal("Failed to sync cache")
    }

    log.Println("Informer cache synced and ready")
}
```

### Filtered Informers

```go
// Create informer with label selector
lister := factory.Core().V1().Pods().Lister()
podList, err := lister.Pods("default").List(labels.SelectorFromSet(labels.Set{
    "app": "myapp",
}))
```

### Indexes

Add indexes to queries for better performance.

```go
// Add index to informer
func (r *MyAppReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&appv1.MyApp{}).
        Watches(
            &source.Kind{Type: &appsv1.Deployment{}},
            handler.EnqueueRequestsFromMapFunc(r.findObjectsForDeployment),
        ).
        Complete(r)
}

// Index function
func (r *MyAppReconciler) findObjectsForDeployment(deployment client.Object) []reconcile.Request {
    myapps := &appv1.MyAppList{}
    if err := r.List(context.Background(), myapps, client.InNamespace(deployment.GetNamespace())); err != nil {
        return []reconcile.Request{}
    }

    requests := []reconcile.Request{}
    for _, myapp := range myapps.Items {
        if deployment.Name == myapp.Name {
            requests = append(requests, reconcile.Request{
                NamespacedName: types.NamespacedName{
                    Name:      myapp.Name,
                    Namespace: myapp.Namespace,
                },
            })
        }
    }
    return requests
}
```

## Work Queues

Work queues buffer events and provide rate limiting for controllers.

### Work Queue Types

```go
import "k8s.io/client-go/util/workqueue"

// 1. Rate limiting queue (default)
rateLimitingQueue := workqueue.NewRateLimitingQueue(
    workqueue.DefaultControllerRateLimiter(),
)

// 2. Named rate limiting queue
namedQueue := workqueue.NewNamedRateLimitingQueue(
    workqueue.DefaultControllerRateLimiter(),
    "my-controller",
)

// 3. Delaying queue (schedule future processing)
delayingQueue := workqueue.NewDelayingQueue()

// 4. Basic queue (no rate limiting)
basicQueue := workqueue.New()
```

### Rate Limiting

Rate limiting prevents thundering herd problems and handles retries with backoff.

```go
import "k8s.io/client-go/util/workqueue"

// Custom rate limiter
limiter := workqueue.NewMaxOfRateLimiter(
    workqueue.NewItemExponentialFailureRateLimiter(5*time.Millisecond, 1000*time.Second),
    &workqueue.BucketRateLimiter{Limiter: rate.NewLimiter(rate.Limit(10), 100)},
)

queue := workqueue.NewRateLimitingQueue(limiter)
```

### Processing Work Items

```go
func (r *MyAppReconciler) processNextWorkItem() bool {
    // Dequeue item
    obj, shutdown := r.queue.Get()
    if shutdown {
        return false
    }

    defer r.queue.Done(obj)

    // Process item
    key, ok := obj.(string)
    if !ok {
        r.queue.Forget(obj)
        r.logger.Errorf("expected string in workqueue but got %#v", obj)
        return true
    }

    if err := r.syncHandler(key); err != nil {
        r.logger.Errorf("error syncing '%s': %s, requeuing", key, err.Error())
        r.queue.AddRateLimited(key)
        return true
    }

    r.queue.Forget(obj)
    r.logger.Infof("successfully synced '%s'", key)
    return true
}

func (r *MyAppReconciler) syncHandler(key string) error {
    // Parse key
    namespace, name, err := cache.SplitMetaNamespaceKey(key)
    if err != nil {
        return err
    }

    // Get CR
    myapp := &appv1.MyApp{}
    if err := r.client.Get(context.Background(), client.ObjectKey{
        Name:      name,
        Namespace: namespace,
    }, myapp); err != nil {
        if errors.IsNotFound(err) {
            r.logger.Infof("MyApp '%s' has been deleted", key)
            return nil
        }
        return err
    }

    // Reconcile
    return r.reconcile(context.Background(), myapp)
}
```

## Finalizers

Finalizers allow controllers to perform cleanup before resource deletion.

### Finalizer Pattern

```
┌─────────────────────────────────────────────────────────────────┐
│                   Finalizer Deletion Flow                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. User deletes CR                                             │
│     kubectl delete myapp myapp-instance                         │
│                                                                 │
│     │                                                           │
│     ▼                                                           │
│                                                                 │
│  2. Kubernetes sets deletionTimestamp                           │
│     ┌─────────────────────────────────────────────────────────┐ │
│     │ metadata:                                              │ │
│     │   name: myapp-instance                                 │ │
│     │   deletionTimestamp: 2024-01-15T10:30:00Z               │ │
│     │   finalizers:                                          │ │
│     │     - myapp.example.com/finalizer                     │ │
│     └─────────────────────────────────────────────────────────┘ │
│                                                                 │
│     │                                                           │
│     ▼                                                           │
│                                                                 │
│  3. Controller reconciles deletion                             │
│     - Sees deletionTimestamp set                              │
│     - Checks finalizers                                         │
│     - Performs cleanup                                         │
│                                                                 │
│     │                                                           │
│     ▼                                                           │
│                                                                 │
│  4. Controller removes finalizer                               │
│     ┌─────────────────────────────────────────────────────────┐ │
│     │ metadata:                                              │ │
│     │   name: myapp-instance                                 │ │
│     │   deletionTimestamp: 2024-01-15T10:30:00Z               │ │
│     │   finalizers: []  # Removed                            │ │
│     └─────────────────────────────────────────────────────────┘ │
│                                                                 │
│     │                                                           │
│     ▼                                                           │
│                                                                 │
│  5. Kubernetes deletes CR                                      │
│     Object removed from etcd                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Implementing Finalizers

```go
const myAppFinalizer = "myapp.example.com/finalizer"

func (r *MyAppReconciler) handleDeletion(ctx context.Context, myapp *appv1.MyApp) (ctrl.Result, error) {
    logger := r.Log.WithValues("myapp", myapp.Name)

    // Check if finalizer exists
    if !containsString(myapp.Finalizers, myAppFinalizer) {
        return ctrl.Result{}, nil
    }

    // Perform cleanup
    logger.Info("Performing cleanup")
    if err := r.cleanupResources(ctx, myapp); err != nil {
        return ctrl.Result{}, err
    }

    // Remove finalizer
    myapp.Finalizers = removeString(myapp.Finalizers, myAppFinalizer)
    if err := r.Update(ctx, myapp); err != nil {
        return ctrl.Result{}, err
    }

    logger.Info("Cleanup complete, finalizer removed")
    return ctrl.Result{}, nil
}

func (r *MyAppReconciler) ensureFinalizer(ctx context.Context, myapp *appv1.MyApp) error {
    if !containsString(myapp.Finalizers, myAppFinalizer) {
        myapp.Finalizers = append(myapp.Finalizers, myAppFinalizer)
        return r.Update(ctx, myapp)
    }
    return nil
}

func (r *MyAppReconciler) cleanupResources(ctx context.Context, myapp *appv1.MyApp) error {
    // Delete deployment
    deployment := &appsv1.Deployment{}
    if err := r.Get(ctx, client.ObjectKey{
        Name:      myapp.Name,
        Namespace: myapp.Namespace,
    }, deployment); err == nil {
        if err := r.Delete(ctx, deployment); err != nil {
            return err
        }
    }

    // Delete service
    service := &corev1.Service{}
    if err := r.Get(ctx, client.ObjectKey{
        Name:      myapp.Name,
        Namespace: myapp.Namespace,
    }, service); err == nil {
        if err := r.Delete(ctx, service); err != nil {
            return err
        }
    }

    return nil
}

func containsString(slice []string, s string) bool {
    for _, item := range slice {
        if item == s {
            return true
        }
    }
    return false
}

func removeString(slice []string, s string) []string {
    var result []string
    for _, item := range slice {
        if item == s {
            continue
        }
        result = append(result, item)
    }
    return result
}
```

## Status Subresource

Status subresource allows separate updates of status and spec.

### Status vs Spec

```
┌─────────────────────────────────────────────────────────────────┐
│                    Spec vs Status                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Spec: Desired state (user-controlled)                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ spec:                                                  │   │
│  │   replicas: 3        # User sets this                  │   │
│  │   image: v1.2.0      # User sets this                  │   │
│  │   port: 8080         # User sets this                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Status: Observed state (controller-controlled)                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ status:                                                │   │
│  │   readyReplicas: 3    # Controller updates this         │   │
│  │   phase: Running      # Controller updates this         │   │
│  │   observedGeneration: 5  # Controller updates this      │   │
│  │   conditions:          # Controller updates this         │   │
│  │     - type: Available                                   │
│  │       status: "True"                                     │
│  │     - type: Progressing                                  │
│  │       status: "True"                                     │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Updating Status

```go
func (r *MyAppReconciler) updateStatus(ctx context.Context, myapp *appv1.MyApp) error {
    // Get latest state
    myappClone := myapp.DeepCopy()

    // Update status
    myappClone.Status.ReadyReplicas = r.getReadyReplicas(ctx, myapp)
    myappClone.Status.Phase = r.getPhase(ctx, myapp)
    myappClone.Status.ObservedGeneration = myapp.Generation

    // Update status subresource (doesn't trigger reconcile)
    return r.Status().Update(ctx, myappClone)
}
```

### Status Conditions

Conditions provide standardized status information.

```go
const (
    ConditionAvailable = "Available"
    ConditionProgressing = "Progressing"
    ConditionDegraded = "Degraded"
)

func (r *MyAppReconciler) setCondition(status *appv1.MyAppStatus, conditionType string, status metav1.ConditionStatus, reason, message string) {
    newCondition := metav1.Condition{
        Type:               conditionType,
        Status:             status,
        Reason:             reason,
        Message:            message,
        LastTransitionTime: metav1.Now(),
    }

    for i, condition := range status.Conditions {
        if condition.Type == conditionType {
            if condition.Status != status {
                newCondition.LastTransitionTime = metav1.Now()
            }
            status.Conditions[i] = newCondition
            return
        }
    }

    status.Conditions = append(status.Conditions, newCondition)
}
```

## Owner References

Owner references establish parent-child relationships between resources.

### Owner Reference Pattern

```
┌─────────────────────────────────────────────────────────────────┐
│                  Owner Reference Hierarchy                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MyApp (CR)                                                     │
│    │                                                            │
│    │ ownerReferences:                                           │
│    │   - apiVersion: example.com/v1                             │
│    │     kind: MyApp                                           │
│    │     name: myapp-instance                                  │
│    │     controller: true                                      │
│    │                                                             │
│    ├────────────────────────────────────────────────────────┐  │
│    │                                                         │  │
│    ▼                                                         │  │
│  Deployment (owned by MyApp)                                 │  │
│    │ ownerReferences:                                        │  │
│    │   - apiVersion: example.com/v1                           │  │
│    │     kind: MyApp                                           │  │
│    │     name: myapp-instance                                  │  │
│    │     controller: true                                      │  │
│    │                                                         │  │
│    │                                                         │  │
│    ├────────────────────────────────────────────────────────┐  │
│    │                                                         │  │
│    ▼                                                         │  │
│  ReplicaSet (owned by Deployment)                           │  │
│    │                                                         │  │
│    └────────────────────────────────────────────────────────┤  │
│                                                              │  │
│  When MyApp is deleted:                                       │  │
│  1. Kubernetes cascades delete to Deployment                 │  │
│  2. Deployment cascades delete to ReplicaSet                 │  │
│  3. All resources are cleaned up automatically               │  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Setting Owner References

```go
func (r *MyAppReconciler) newDeployment(myapp *appv1.MyApp) *appsv1.Deployment {
    return &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      myapp.Name,
            Namespace: myapp.Namespace,
            Labels:    myapp.Labels,
            OwnerReferences: []metav1.OwnerReference{
                {
                    APIVersion: myapp.APIVersion,
                    Kind:       myapp.Kind,
                    Name:       myapp.Name,
                    UID:        myapp.UID,
                    Controller: pointer.BoolPtr(true),
                },
            },
        },
        Spec: r.deploymentSpec(myapp),
    }
}
```

### Background vs Foreground Deletion

```go
// Foreground deletion (default)
// - Owner waits for dependents to be deleted
// - Blocks until cleanup complete

// Background deletion
// - Owner deleted immediately
// - Dependents deleted in background

func (r *MyAppReconciler) ensureOwnerReference(obj client.Object, owner client.Object) error {
    obj.SetOwnerReferences([]metav1.OwnerReference{
        {
            APIVersion: owner.GetObjectKind().GroupVersionKind().GroupVersion().String(),
            Kind:       owner.GetObjectKind().GroupVersionKind().Kind,
            Name:       owner.GetName(),
            UID:        owner.GetUID(),
            Controller: pointer.BoolPtr(true),
        },
    })
    return nil
}
```

## Watchers and Events

Watchers monitor resources and generate events for changes.

### Watching Resources

```go
func (r *MyAppReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        // Watch CR changes
        For(&appv1.MyApp{}).
        // Watch owned Deployments
        Owns(&appsv1.Deployment{}).
        // Watch related Services
        Watches(
            &source.Kind{Type: &corev1.Service{}},
            handler.EnqueueRequestsFromMapFunc(r.findObjectsForService),
        ).
        Complete(r)
}
```

### Events

Events provide audit trail and debugging information.

```go
// Emit event
func (r *MyAppReconciler) emitEvent(ctx context.Context, object client.Object, eventType, reason, message string) {
    r.Event(object, eventType, reason, message)
}

// Usage
r.emitEvent(ctx, myapp, corev1.EventTypeNormal, "Created", "Deployment created successfully")
r.emitEvent(ctx, myapp, corev1.EventTypeWarning, "Failed", "Failed to create deployment")

// Event types
// corev1.EventTypeNormal  - Normal operation
// corev1.EventTypeWarning - Warning/issue
```

## Testing

### Controller Testing

```go
package controllers_test

import (
    "testing"

    "github.com/stretchr/testify/require"
    appv1 "myapp/api/v1"
    "myapp/controllers"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/reconcile"
)

func TestMyAppReconciler_Reconcile(t *testing.T) {
    tests := []struct {
        name    string
        myapp   *appv1.MyApp
        want    reconcile.Result
        wantErr bool
    }{
        {
            name: "create deployment",
            myapp: &appv1.MyApp{
                ObjectMeta: metav1.ObjectMeta{
                    Name:      "test-app",
                    Namespace: "default",
                },
                Spec: appv1.MyAppSpec{
                    Replicas: 3,
                    Image:    "test:v1.0.0",
                },
            },
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Setup test environment
            scheme := runtime.NewScheme()
            appv1.AddToScheme(scheme)
            client := fake.NewClientBuilder().
                WithScheme(scheme).
                WithRuntimeObjects(tt.myapp).
                Build()

            r := &controllers.MyAppReconciler{
                Client: client,
                Scheme: scheme,
            }

            // Run reconcile
            req := reconcile.Request{
                NamespacedName: client.ObjectKey{
                    Name:      tt.myapp.Name,
                    Namespace: tt.myapp.Namespace,
                },
            }

            got, err := r.Reconcile(context.Background(), req)

            if (err != nil) != tt.wantErr {
                t.Errorf("Reconcile() error = %v, wantErr %v", err, tt.wantErr)
                return
            }
            if got != tt.want {
                t.Errorf("Reconcile() got = %v, want %v", got, tt.want)
            }

            // Verify deployment was created
            deployment := &appsv1.Deployment{}
            err = client.Get(context.Background(), client.ObjectKey{
                Name:      tt.myapp.Name,
                Namespace: tt.myapp.Namespace,
            }, deployment)
            require.NoError(t, err)
        })
    }
}
```

### EnvTest Testing

```go
package controllers_test

import (
    "testing"
    "time"

    "sigs.k8s.io/controller-runtime/pkg/envtest"
    "sigs.k8s.io/controller-runtime/pkg/envtest/printer"
)

func TestControllers(t *testing.T) {
    testEnv := &envtest.Environment{
        CRDDirectoryPaths:     []string{"../../config/crd/bases"},
        ErrorIfCRDPathMissing: true,
    }

    cfg, err := testEnv.Start()
    if err != nil {
        t.Fatalf("failed to start test environment: %v", err)
    }
    defer testEnv.Stop()

    // Run tests
    t.Run("TestMyAppController", func(t *testing.T) {
        // Use real API server
        // test controller behavior
    })
}
```

## Best Practices

### 1. Idempotent Reconciliation

```go
// ✅ Good: Idempotent
func (r *MyAppReconciler) ensureDeployment(ctx context.Context, myapp *appv1.MyApp) error {
    // Get or create
    deployment := &appsv1.Deployment{}
    if err := r.Get(ctx, client.ObjectKey{Name: myapp.Name, Namespace: myapp.Namespace}, deployment); err != nil {
        if errors.IsNotFound(err) {
            deployment = r.newDeployment(myapp)
            return r.Create(ctx, deployment)
        }
        return err
    }

    // Update if needed
    if !deploymentEqual(deployment, myapp) {
        deployment.Spec = r.deploymentSpec(myapp)
        return r.Update(ctx, deployment)
    }

    return nil
}

// ❌ Bad: Not idempotent
func (r *MyAppReconciler) ensureDeployment(ctx context.Context, myapp *appv1.MyApp) error {
    deployment := r.newDeployment(myapp)
    return r.Create(ctx, deployment)  // Fails on second run
}
```

### 2. Status Updates

```go
// ✅ Good: Separate status update
func (r *MyAppReconciler) updateStatus(ctx context.Context, myapp *appv1.MyApp) error {
    myappClone := myapp.DeepCopy()
    myappClone.Status.ReadyReplicas = r.getReadyReplicas(ctx, myapp)
    return r.Status().Update(ctx, myappClone)
}

// ❌ Bad: Update entire object (triggers reconcile)
func (r *MyAppReconciler) updateStatus(ctx context.Context, myapp *appv1.MyApp) error {
    myapp.Status.ReadyReplicas = r.getReadyReplicas(ctx, myapp)
    return r.Update(ctx, myapp)  // Triggers infinite loop
}
```

### 3. Error Handling

```go
// ✅ Good: Requeue on transient errors
func (r *MyAppReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    if err := r.doWork(ctx); err != nil {
        if isTransientError(err) {
            return ctrl.Result{RequeueAfter: time.Second * 5}, nil
        }
        return ctrl.Result{}, err
    }
    return ctrl.Result{}, nil
}

// ❌ Bad: Always return error
func (r *MyAppReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    if err := r.doWork(ctx); err != nil {
        return ctrl.Result{}, err  // Exponential backoff
    }
    return ctrl.Result{}, nil
}
```

### 4. Finalizers

```go
// ✅ Good: Proper finalizer handling
func (r *MyAppReconciler) handleDeletion(ctx context.Context, myapp *appv1.MyApp) (ctrl.Result, error) {
    if !containsString(myapp.Finalizers, finalizerName) {
        return ctrl.Result{}, nil
    }

    // Cleanup
    if err := r.cleanupResources(ctx, myapp); err != nil {
        return ctrl.Result{}, err
    }

    // Remove finalizer
    myapp.Finalizers = removeString(myapp.Finalizers, finalizerName)
    return ctrl.Result{}, r.Update(ctx, myapp)
}

// ❌ Bad: No finalizer
func (r *MyAppReconciler) handleDeletion(ctx context.Context, myapp *appv1.MyApp) (ctrl.Result, error) {
    // Cleanup
    return ctrl.Result{}, r.cleanupResources(ctx, myapp)
    // Orphaned resources!
}
```

### 5. Watch Setup

```go
// ✅ Good: Watch owned resources
func (r *MyAppReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&appv1.MyApp{}).
        Owns(&appsv1.Deployment{}).
        Complete(r)
}

// ❌ Bad: Don't watch resources
func (r *MyAppReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&appv1.MyApp{}).
        Complete(r)
    // Won't reconcile when deployment changes
}
```

### 6. Logging

```go
// ✅ Good: Structured logging
logger.Info("Creating deployment",
    "namespace", myapp.Namespace,
    "name", myapp.Name,
    "replicas", myapp.Spec.Replicas,
)

// ❌ Bad: Unstructured logging
logger.Info(fmt.Sprintf("Creating deployment %s/%s with %d replicas",
    myapp.Namespace, myapp.Name, myapp.Spec.Replicas))
```

### 7. Resource Limits

```go
// ✅ Good: Set resource limits
func (r *MyAppReconciler) newDeployment(myapp *appv1.MyApp) *appsv1.Deployment {
    return &appsv1.Deployment{
        Spec: appsv1.DeploymentSpec{
            Template: corev1.PodTemplateSpec{
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{
                        {
                            Name: "myapp",
                            Resources: corev1.ResourceRequirements{
                                Requests: corev1.ResourceList{
                                    corev1.ResourceCPU:    resource.MustParse("100m"),
                                    corev1.ResourceMemory: resource.MustParse("128Mi"),
                                },
                                Limits: corev1.ResourceList{
                                    corev1.ResourceCPU:    resource.MustParse("500m"),
                                    corev1.ResourceMemory: resource.MustParse("512Mi"),
                                },
                            },
                        },
                    },
                },
            },
        },
    }
}

// ❌ Bad: No resource limits
func (r *MyAppReconciler) newDeployment(myapp *appv1.MyApp) *appsv1.Deployment {
    return &appsv1.Deployment{
        Spec: appsv1.DeploymentSpec{
            Template: corev1.PodTemplateSpec{
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{
                        {
                            Name: "myapp",
                            // No resource limits - can starve the node
                        },
                    },
                },
            },
        },
    }
}
```

### 8. Namespaces

```go
// ✅ Good: Namespace-scoped
func (r *MyAppReconciler) getDeployment(ctx context.Context, myapp *appv1.MyApp) (*appsv1.Deployment, error) {
    deployment := &appsv1.Deployment{}
    err := r.Get(ctx, client.ObjectKey{
        Name:      myapp.Name,
        Namespace: myapp.Namespace,  // Always include namespace
    }, deployment)
    return deployment, err
}

// ❌ Bad: Hardcoded namespace
func (r *MyAppReconciler) getDeployment(ctx context.Context, myapp *appv1.MyApp) (*appsv1.Deployment, error) {
    deployment := &appsv1.Deployment{}
    err := r.Get(ctx, client.ObjectKey{
        Name:      myapp.Name,
        Namespace: "default",  // Hardcoded!
    }, deployment)
    return deployment, err
}
```

### 9. Context Handling

```go
// ✅ Good: Propagate context
func (r *MyAppReconciler) createDeployment(ctx context.Context, myapp *appv1.MyApp) error {
    deployment := r.newDeployment(myapp)
    return r.Create(ctx, deployment)  // Pass context
}

// ❌ Bad: Ignore context
func (r *MyAppReconciler) createDeployment(ctx context.Context, myapp *appv1.MyApp) error {
    deployment := r.newDeployment(myapp)
    return r.Create(context.Background(), deployment)  // Lost context
}
```

### 10. Validation

```go
// ✅ Good: Validate spec
func (r *MyAppReconciler) validateSpec(spec *appv1.MyAppSpec) error {
    if spec.Replicas < 1 {
        return fmt.Errorf("replicas must be at least 1, got %d", spec.Replicas)
    }
    if spec.Image == "" {
        return fmt.Errorf("image is required")
    }
    return nil
}

// ❌ Bad: No validation
func (r *MyAppReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // No validation - could create invalid resources
    myapp := &appv1.MyApp{}
    r.Get(ctx, req.NamespacedName, myapp)
    r.createDeployment(ctx, myapp)
    return ctrl.Result{}, nil
}
```
