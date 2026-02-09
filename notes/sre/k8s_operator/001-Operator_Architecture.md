---
id: 005-Operator_Architecture
aliases:
  - kubernetes-operator-architecture
  - controller-architecture
tags:
  - kubernetes
  - operator
  - controller
  - golang
  - sre
---

# Kubernetes Operator Architecture

## What is a Kubernetes Operator

A Kubernetes operator is a method of packaging, deploying, and managing a Kubernetes application. It extends the Kubernetes API to create, configure, and manage instances of complex applications on behalf of Kubernetes users.

### Core Concept

An operator uses custom resources to represent application state and implements control loops to reconcile actual state with desired state, just like the built-in Kubernetes controllers.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Traditional Deployment                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐        │
│  │    MySQL    │    │   Redis     │    │    App      │        │
│  │   Deploy    │    │   Deploy    │    │   Deploy    │        │
│  └─────────────┘    └─────────────┘    └─────────────┘        │
│                                                                 │
│  Problem: No intelligent management, manual operations         │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    Operator-Based Deployment                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                  MySQL Operator                        │  │
│  │  - Automatic provisioning                              │  │
│  │  - Backup management                                   │  │
│  │  - Failover handling                                    │  │
│  │  - Scaling                                              │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                  Redis Operator                        │  │
│  │  - Cluster management                                   │  │
│  │  - Rebalancing                                          │  │
│  │  - Persistence                                          │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Benefit: Intelligent, automated application management       │
└─────────────────────────────────────────────────────────────────┘
```

## Operator Pattern

The operator pattern codifies operational knowledge into software, enabling applications to manage themselves.

### Operator Capabilities

**Deployment:**
- Install application components
- Configure networking and storage
- Set up dependencies

**Lifecycle Management:**
- Upgrade and rollback
- Backup and restore
- Configuration updates

**Health Management:**
- Health checks and monitoring
- Automatic recovery from failures
- Disaster recovery

**Scaling:**
- Horizontal and vertical scaling
- Resource optimization
- Load balancing

## Controller Architecture

### Control Loop

The core of every operator is the reconciliation loop that continuously works to bring the actual state to the desired state.

```
Desired State (YAML)
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│                     Reconciliation                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Watch CR for changes                                    │
│  2. Fetch current state from API Server                      │
│  3. Compare desired vs actual                               │
│  4. Take corrective actions                                 │
│  5. Update status                                           │
│                                                             │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
                 Actual State (Cluster)
```

### Reconciliation Cycle

```
┌─────────────────────────────────────────────────────────────────┐
│                      Reconciliation Cycle                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐        │
│  │  Watch  │──▶│  Fetch  │──▶│ Compare │──▶│  Act    │        │
│  │ Events  │   │  State  │   │  Diff   │   │ Changes │        │
│  └────┬────┘   └────┬────┘   └────┬────┘   └────┬────┘        │
│       │             │             │             │              │
│       │             │             │             ▼              │
│       │             │             │      ┌─────────────┐       │
│       │             │             │      │  Apply      │       │
│       │             │             │      │  Changes    │       │
│       │             │             │      └──────┬──────┘       │
│       │             │             │             │              │
│       │             │             │             ▼              │
│       │             │             │      ┌─────────────┐       │
│       │             │             │      │  Update     │       │
│       │             │             │      │  Status     │       │
│       │             │             │      └──────┬──────┘       │
│       │             │             │             │              │
│       │             │             │             ▼              │
│       │             └─────────────┴─────────────┘              │
│       │                   Requeue for next cycle             │
│       │                                                     │
│       └─────────────────────────────────────────────────────┘
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Controller Components

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                   Kubernetes Controller                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐         ┌───────────────┐                   │
│  │  Informer   │         │   Work Queue  │                   │
│  │  & Cache    │◀────────│   (Channel)   │                   │
│  └──────┬──────┘         └───────┬───────┘                   │
│         │                         │                            │
│         │ Watch                  │ Enqueue                     │
│         │ Events                 │ Events                      │
│         ▼                         │                            │
│  ┌─────────────┐                 │                            │
│  │ API Server  │                 │                            │
│  │   (Watch)   │                 │                            │
│  └─────────────┘                 │                            │
│         ▲                         │                            │
│         │ Read/Write              │                            │
│         ▼                         ▼                            │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              Controller Logic                          │    │
│  │  ┌────────────────────────────────────────────────┐  │    │
│  │  │  ProcessNextWorkItem()                         │  │    │
│  │  │  1. Dequeue from work queue                     │  │    │
│  │  │  2. Get object from cache                       │  │    │
│  │  │  3. Run Reconcile logic                        │  │    │
│  │  │  4. Apply changes to API Server                │  │    │
│  │  │  5. Update status                               │  │    │
│  │  │  6. Requeue if needed                           │  │    │
│  │  └────────────────────────────────────────────────┘  │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Informers and Caches

Informers watch Kubernetes resources and maintain a local cache to reduce API server load.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Informer and Cache                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  API Server                                                       │
│      │                                                           │
│      │ Watch                                                     │
│      ▼                                                           │
│  ┌─────────────┐                                                 │
│  │  Informer   │                                                 │
│  │             │                                                 │
│  │  ┌─────────┴─────────────────────────────────────────────┐   │
│  │  │  1. Establish watch with API Server                   │   │
│  │  │  2. Receive events (ADD, UPDATE, DELETE)            │   │
│  │  │  3. Update local cache                               │   │
│  │  │  4. Dispatch to event handlers                      │   │
│  │  └─────────────────────────────────────────────────────┘   │
│  │             │                                                 │
│  │             ▼                                                 │
│  │  ┌─────────────────────────────────────────────────────┐   │
│  │  │              Local Cache (Thread-safe)               │   │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │   │
│  │  │  │   Object A  │  │   Object B  │  │   Object C  │  │   │
│  │  │  └─────────────┘  └─────────────┘  └─────────────┘  │   │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │   │
│  │  │  │   Object D  │  │   Object E  │  │   Object F  │  │   │
│  │  │  └─────────────┘  └─────────────┘  └─────────────┘  │   │
│  │  └─────────────────────────────────────────────────────┘   │
│  │             │                                                 │
│  │             │ Get/List                                        │
│  │             │ (No API calls, reads from cache)                │
│  │             ▼                                                 │
│  │  ┌─────────────────────────────────────────────────────┐   │
│  │  │            Controller Logic                          │   │
│  │  └─────────────────────────────────────────────────────┘   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Benefits:**
- Reduces API server load
- Faster reads (local cache)
- Automatic watch reconnection
- Event filtering and distribution
- Thread-safe operations

**Code Example:**
```go
import (
    "k8s.io/client-go/informers"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/tools/cache"
    "k8s.io/client-go/tools/clientcmd"
)

func setupInformer() {
    config, _ := clientcmd.BuildConfigFromFlags("", kubeconfig)
    clientset := kubernetes.NewForConfigOrDie(config)

    // Create a shared informer factory
    factory := informers.NewSharedInformerFactory(clientset, time.Second*30)

    // Get an informer for pods
    podInformer := factory.Core().V1().Pods().Informer()

    // Add event handler
    podInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            pod := obj.(*v1.Pod)
            log.Printf("Pod added: %s/%s", pod.Namespace, pod.Name)
        },
        UpdateFunc: func(oldObj, newObj interface{}) {
            oldPod := oldObj.(*v1.Pod)
            newPod := newObj.(*v1.Pod)
            log.Printf("Pod updated: %s/%s", newPod.Namespace, newPod.Name)
        },
        DeleteFunc: func(obj interface{}) {
            pod := obj.(*v1.Pod)
            log.Printf("Pod deleted: %s/%s", pod.Namespace, pod.Name)
        },
    })

    // Start informers
    factory.Start(stopCh)
    factory.WaitForCacheSync(stopCh)
}
```

### Work Queues

Work queues buffer events and process them in order, with rate limiting.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Work Queue Processing                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌─────────────────────────────────────┐     │
│  │  Informer   │    │        Work Queue                  │     │
│  │  Events     │────▶│    (Rate Limiting)                │     │
│  └─────────────┘    │                                     │     │
│                     │  [event1, event2, event3, ...]      │     │
│                     │                                     │     │
│                     └──────────────┬──────────────────────┘     │
│                                    │                            │
│                                    │ Dequeue                     │
│                                    │                            │
│                                    ▼                            │
│                     ┌─────────────────────────────────────┐  │
│                     │    Worker Pool                       │  │
│                     │  ┌─────────┐ ┌─────────┐           │  │
│                     │  │Worker 1 │ │Worker 2 │ ...       │  │
│                     │  └────┬────┘ └────┬────┘           │  │
│                     │       │          │                 │  │
│                     │       ▼          ▼                 │  │
│                     │  ┌─────────────────────────────┐   │  │
│                     │  │  ProcessNextWorkItem()      │   │  │
│                     │  │  - Get item from queue      │   │  │
│                     │  │  - Run reconcile logic      │   │  │
│                     │  │  - Handle errors             │   │  │
│                     │  │  - Requeue if needed        │   │  │
│                     │  └─────────────────────────────┘   │  │
│                     └─────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Queue Types:**

**Rate Limiting Queue:**
```go
import "k8s.io/client-go/util/workqueue"

queue := workqueue.NewRateLimitingQueue(workqueue.DefaultControllerRateLimiter())
```
- Default for controllers
- Exponential backoff for retries
- Prevents thundering herd problems

**Named Queue:**
```go
queue := workqueue.NewNamedRateLimitingQueue(
    workqueue.DefaultControllerRateLimiter(),
    "my-controller",
)
```
- Named for debugging
- Useful for multiple queues

**Delaying Queue:**
```go
delayQueue := workqueue.NewDelayingQueue()

// Add item with delay
delayQueue.AddAfter(obj, time.Second*10)
```
- Schedule future processing
- Good for delayed reconciliation

## Kubernetes Client Libraries

### client-go

The official Go client for Kubernetes, used for building operators.

```go
import (
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/rest"
    "k8s.io/client-go/tools/clientcmd"
)

// In-cluster config
config, err := rest.InClusterConfig()

// Or from kubeconfig
config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)

// Create clientset
clientset := kubernetes.NewForConfigOrDie(config)

// Use clientset to interact with API
pods, err := clientset.CoreV1().Pods(namespace).List(ctx, listOptions)
```

### controller-runtime

Higher-level framework for building controllers on top of client-go.

```
┌─────────────────────────────────────────────────────────────────┐
│                 controller-runtime Architecture                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐                                                │
│  │   Manager   │  Entry point, manages shared resources        │
│  │             │                                                │
│  │  ┌─────────┴─────────────────────────────────────────────┐ │
│  │  │                                                       │ │
│  │  │  Shared Components:                                   │ │
│  │  │  - Client (for API access)                            │ │
│  │  │  - Cache (for resource caching)                       │ │
│  │  │  - Scheme (type registration)                         │ │
│  │  │  - EventBroadcaster (events)                          │ │
│  │  │                                                       │ │
│  │  └───────────────────────────────────────────────────────┘ │
│  │             │                                                │
│  │             │ Manages                                        │
│  │             ▼                                                │
│  │  ┌─────────────────────────────────────────────────────┐  │
│  │  │               Controllers                           │  │
│  │  │  ┌─────────────┐  ┌─────────────┐                  │  │
│  │  │  │ Controller A │  │ Controller B │  ...            │  │
│  │  │  └─────────────┘  └─────────────┘                  │  │
│  │  │                                                     │  │
│  │  │  Each Controller:                                   │  │
│  │  │  - Watches specific CRD types                       │  │
│  │  │  - Runs reconcile logic                             │  │
│  │  │  - Uses shared client and cache                     │  │
│  │  └─────────────────────────────────────────────────────┘  │
│  │                                                           │
│  └───────────────────────────────────────────────────────────┘
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Basic Controller Setup:**
```go
package main

import (
    "context"
    "os"

    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/log/zap"

    myappv1 "myproject/api/v1"
    "myproject/controllers"
)

func main() {
    // Setup logger
    logger := zap.New(zap.UseDevMode(true))

    // Setup scheme
    scheme := runtime.NewScheme()
    myappv1.AddToScheme(scheme)

    // Create manager
    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
        Scheme:             scheme,
        MetricsBindAddress: ":8080",
        Port:               9443,
    })
    if err != nil {
        logger.Error(err, "unable to start manager")
        os.Exit(1)
    }

    // Setup controller
    if err := (&controllers.MyAppReconciler{
        Client: mgr.GetClient(),
        Scheme: mgr.GetScheme(),
        Log:    ctrl.Log.WithName("controllers").WithName("MyApp"),
    }).SetupWithManager(mgr); err != nil {
        logger.Error(err, "unable to create controller")
        os.Exit(1)
    }

    // Start manager
    if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
        logger.Error(err, "problem running manager")
        os.Exit(1)
    }
}
```

## CRDs and Operators Relationship

CRDs define the API, operators implement the control logic.

```
┌─────────────────────────────────────────────────────────────────┐
│              CRDs and Operators Relationship                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                   Custom Resource Definition (CRD)      │  │
│  │  Defines:                                               │  │
│  │  - API version (apiVersion: myapp.example.com/v1)       │  │
│  │  - Kind (kind: MyApp)                                   │  │
│  │  - Spec schema (desired state)                          │  │
│  │  - Status schema (observed state)                       │  │
│  │  - Validation rules                                      │  │
│  │  - Subresources (status, scale)                         │  │
│  └────────────────────┬────────────────────────────────────┘  │
│                       │                                        │
│                       │ Registers                               │
│                       │                                        │
│                       ▼                                        │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │               Kubernetes API Server                    │  │
│  │  - Stores CRD definition                                │  │
│  │  - Validates resources against CRD                      │  │
│  │  - Serves API for CRUD operations                        │  │
│  └────────────────────┬────────────────────────────────────┘  │
│                       │                                        │
│                       │ Stores                                 │
│                       │                                        │
│                       ▼                                        │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │               Custom Resource (CR) Instance             │  │
│  │  Example:                                               │  │
│  │    apiVersion: myapp.example.com/v1                     │  │
│  │    kind: MyApp                                          │  │
│  │    metadata:                                            │  │
│  │      name: myapp-instance                               │  │
│  │    spec:                                                │  │
│  │      replicas: 3                                        │  │
│  │      image: myapp:v1.0.0                                │  │
│  │    status:                                              │  │
│  │      readyReplicas: 3                                   │  │
│  └────────────────────┬────────────────────────────────────┘  │
│                       │                                        │
│                       │ Watched by                              │
│                       │                                        │
│                       ▼                                        │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                  Operator (Controller)                   │  │
│  │  - Watches CR changes                                    │  │
│  │  - Reconciles desired vs actual state                   │  │
│  │  - Manages dependent resources                           │  │
│  │  - Updates status                                       │  │
│  │  - Handles lifecycle events                              │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Complete Operator Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                   Complete Operator Flow                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  User                                                           │
│   │                                                             │
│   │ 1. kubectl apply -f myapp.yaml                            │
│   ▼                                                             │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                    CRD Definition                         │  │
│  │  (Installed once)                                         │  │
│  └─────────────────────────────────────────────────────────┘  │
│   │                                                             │
│   │ 2. kubectl apply -f myapp-instance.yaml                   │
│   ▼                                                             │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                    CR Instance                           │  │
│  │  (Desired state)                                         │  │
│  └─────────────────────────────────────────────────────────┘  │
│   │                                                             │
│   │ API Server event                                           │
│   ▼                                                             │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                    Operator                              │  │
│  │  ┌───────────────────────────────────────────────────┐  │  │
│  │  │  1. Informer detects CR change                    │  │  │
│  │  │  2. Work queue enqueues reconcile request          │  │  │
│  │  │  3. Worker picks up request                        │  │  │
│  │  │  4. Reconcile logic:                               │  │  │
│  │  │     a. Fetch CR from cache                         │  │  │
│  │  │     b. Fetch current state (Deployments, etc.)     │  │  │
│  │  │     c. Compare spec vs actual                      │  │  │
│  │  │     d. Create/update/delete resources              │  │  │
│  │  │     e. Update CR status                            │  │  │
│  │  │  5. Requeue if needed                               │  │  │
│  │  └───────────────────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────┘  │
│   │                                                             │
│   │ 3. Manage resources                                       │
│   ▼                                                             │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │            Kubernetes Resources                           │  │
│  │  (Deployments, Services, ConfigMaps, Secrets, etc.)      │  │
│  └─────────────────────────────────────────────────────────┘  │
│   │                                                             │
│   │ 4. Update status                                           │
│   ▼                                                             │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                    CR Status                             │  │
│  │  (Observed state)                                        │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Operator Patterns

### Single-Resource Operator

Manages a single CR type without complex dependencies.

```
CR (MyApp)
    │
    ▼
Deployment
```

### Multi-Resource Operator

Manages multiple Kubernetes resources to implement a single CR.

```
CR (Database)
    │
    ├─▶ StatefulSet
    ├─▶ Service
    ├─▶ Secret
    ├─▶ ConfigMap
    └─▶ ServiceAccount
```

### Composite Operator

Manages other CRs created by other operators.

```
CR (MyApp)
    │
    ├─▶ CR (Database Operator)
    │       │
    │       └─▶ StatefulSet, Service, etc.
    │
    └─▶ CR (Cache Operator)
            │
            └─▶ Deployment, Service, etc.
```

## Summary

Kubernetes operators provide:
- Declarative management of complex applications
- Automated lifecycle operations
- Self-healing capabilities
- Consistent deployment patterns
- Operational knowledge encoded in software

The core components (informers, work queues, controllers) work together to implement the reconciliation pattern that makes operators powerful tools for managing applications in Kubernetes.
