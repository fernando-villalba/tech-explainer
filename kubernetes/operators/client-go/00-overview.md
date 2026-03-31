# Client-go: The Machinery Under Controller-Runtime

When you write a Kubernetes operator with controller-runtime, you call `r.Get()` to read an object, `r.List()` to list them, `r.Update()` to write changes back. You call `ctrl.NewControllerManagedBy(mgr).For(&MyType{}).Owns(&appsv1.Deployment{}).Complete(r)` and the framework handles the rest. Events flow in. Reconcile functions run. Objects converge.

None of that is magic. It's six client-go subsystems wired together.

controller-runtime's Cache is a set of SharedInformers from `client-go/tools/cache`. The workqueue that deduplicates events and backs off on failures is `client-go/util/workqueue`. The Client that reads from cache and writes to the API server uses `client-go/rest` underneath. Leader election is `client-go/tools/leaderelection`. Event recording is `client-go/tools/record`. Every abstraction you use in controller-runtime maps directly to a client-go primitive.

Understanding these primitives won't change how you write Reconcile functions. But it will change how you debug them. When the cache serves stale data, you'll know it's because the Reflector's watch stream hasn't delivered the event yet. When an object gets requeued with increasing delays, you'll know it's the rate-limiting workqueue applying exponential backoff. When leader election takes 15 seconds to fail over, you'll know it's the LeaseDuration minus the RenewDeadline.

## The REST Client: Where Every Request Starts

Everything in client-go starts with `rest.Config`. This struct holds the connection to the API server: host URL, authentication (bearer token, client certificates, or auth plugins), TLS settings, rate limiting, and timeout configuration.

```go
config, err := ctrl.GetConfigOrDie()
```

That one-liner from controller-runtime calls `rest.InClusterConfig()` when running in a pod (reads the service account token from `/var/run/secrets/kubernetes.io/serviceaccount/`) or falls back to the kubeconfig file on your laptop. Either way, you get a `rest.Config`.

Every client in client-go -- the typed clientset, the dynamic client, controller-runtime's client -- builds on top of a `rest.RESTClient` constructed from this config. The RESTClient handles HTTP verb mapping (GET, POST, PUT, PATCH, DELETE), content negotiation (JSON or protobuf), request retry, and rate limiting. You almost never use it directly, but it's the bottom of the stack.

## The Clientset: Type-Safe Access to Built-in Resources

The `kubernetes` package provides a generated, type-safe client for every built-in Kubernetes resource:

```go
clientset, err := kubernetes.NewForConfig(config)
pods, err := clientset.CoreV1().Pods("default").List(ctx, metav1.ListOptions{})
```

One method per resource per verb. `CoreV1().Pods()` returns a `PodInterface` with `Get`, `List`, `Create`, `Update`, `Delete`, `Patch`, `Watch`. The types are generated from the OpenAPI schema of the Kubernetes API. Every field is typed. Every response is deserialized into the correct Go struct.

controller-runtime replaces this with its generic `client.Client` interface -- `r.Get(ctx, key, &pod)` works for any type, built-in or custom. But the multigres-operator uses the clientset directly in specific situations: webhook handlers that need a raw client before the Manager's cache is running, and test utilities that create typed clients for cluster setup.

## The Dynamic Client: When You Don't Have Go Types

The dynamic client operates on `unstructured.Unstructured` -- a wrapper around `map[string]interface{}`. No Go types needed. You specify the resource by its GVR (GroupVersionResource) and work with raw data:

```go
dynamicClient, err := dynamic.NewForConfig(config)
result, err := dynamicClient.Resource(schema.GroupVersionResource{
    Group:    "multigres.com",
    Version:  "v1alpha1",
    Resource: "multigresclusters",
}).Namespace("default").Get(ctx, "my-cluster", metav1.GetOptions{})
```

This is useful for generic tools that operate on arbitrary resource types, for controllers that manage CRDs they don't have compile-time types for, and for kubectl plugins. controller-runtime's client can also work with `unstructured.Unstructured`, but the dynamic client is what's underneath.

## The Informer Pipeline: From API Server to Local Cache

This is the core of client-go and the most important thing to understand. When controller-runtime's cache serves you an object from `r.Get()`, it's reading from an informer's local store. The informer filled that store using a four-stage pipeline.

**Stage 1: The Reflector.** A Reflector is responsible for keeping a local copy of a resource type in sync with the API server. It does this in two steps. First, it performs an initial LIST to get every existing object. Then it starts a long-lived WATCH connection that streams every subsequent change. The LIST provides a `resourceVersion`, and the WATCH starts from that version, so no changes are missed between the two operations.

The Reflector pushes every object it receives -- from the initial list and from subsequent watch events -- into a DeltaFIFO queue.

**Stage 2: The DeltaFIFO.** This queue stores `(key, []Delta)` pairs. Each Delta has a type -- `Added`, `Updated`, `Deleted`, `Replaced`, or `Sync` -- and the full object. The queue is FIFO (first-in, first-out), but it deduplicates by key: if a Pod changes three times before the consumer processes it, the queue holds all three deltas under one key, and the consumer sees them all at once.

`Replaced` happens during the initial LIST and periodic resyncs -- the entire set of objects is replaced. `Sync` is a periodic re-notification of existing objects, used to ensure handlers eventually see every object even if a watch event was missed.

**Stage 3: The Indexer.** The consumer pops items from the DeltaFIFO and applies them to the Indexer -- a thread-safe, indexed in-memory store. The default index is `MetaNamespaceKeyFunc`, which indexes objects by `namespace/name`. You can add custom indexes for efficient lookups by label, field, or any computed key.

This is the store that controller-runtime's cache reads from. When you call `r.Get(ctx, types.NamespacedName{Name: "foo", Namespace: "bar"}, &obj)`, you're doing a lookup in this indexer. No network call.

**Stage 4: The Event Handlers.** After updating the store, the informer calls registered `ResourceEventHandler` callbacks:

```go
type ResourceEventHandler interface {
    OnAdd(obj interface{}, isInInitialList bool)
    OnUpdate(oldObj, newObj interface{})
    OnDelete(obj interface{})
}
```

controller-runtime registers handlers that extract the object's key (or the owner's key, depending on the watch configuration) and push it onto the workqueue. This is how "Pod changed" becomes "reconcile the ReplicaSet that owns this Pod."

**The SharedInformer.** A SharedInformer wraps all four stages into one unit. "Shared" means one informer per GVK is shared across all controllers that watch that type. If three controllers all watch Pods, they share one Pod informer -- one LIST, one WATCH connection, one store. Each controller registers its own event handler on the shared informer.

controller-runtime's cache creates and manages these shared informers. When you call `For(&MyType{})` in the builder, the cache ensures a SharedInformer exists for `MyType`. When you call `Owns(&appsv1.Deployment{})`, it ensures a SharedInformer exists for Deployments.

## The Workqueue: Deduplication, Delay, and Backoff

The workqueue is what sits between "an event happened" and "Reconcile is called." client-go provides three queue interfaces, each layering on the previous:

**TypedInterface** -- the base queue. `Add(item)`, `Get()`, `Done(item)`, `ShutDown()`. The critical property: *deduplication*. If you `Add("default/my-pod")` while that key is already in the queue or being processed, the item isn't duplicated. This is why controller-runtime can safely enqueue the same object many times -- rapid-fire events don't cause rapid-fire reconciliations.

**TypedDelayingInterface** -- adds `AddAfter(item, duration)`. The item enters the queue after the delay. This powers `Result{RequeueAfter: 5 * time.Second}` in your Reconcile function.

**TypedRateLimitingInterface** -- adds `AddRateLimited(item)`, `Forget(item)`, `NumRequeues(item)`. The rate limiter decides how long to delay based on failure count. This powers the exponential backoff when your Reconcile returns an error.

The default rate limiter for controllers combines two strategies:

1. **Per-item exponential backoff.** Starts at 5 milliseconds, doubles each failure, caps at 1000 seconds (~17 minutes). If object "default/my-pod" fails reconciliation, it's requeued after 5ms, then 10ms, then 20ms, and so on. Each object tracks its own failure count independently.

2. **Overall token bucket.** 10 requests per second with a burst of 100. This prevents a flood of failing objects from consuming all processing capacity.

When Reconcile succeeds (returns `nil, nil`), controller-runtime calls `Forget(item)`, which resets the exponential backoff counter for that item. The next failure starts back at 5ms. When Reconcile returns `RequeueAfter`, it also calls `Forget` -- the explicit delay replaces the backoff.

This is why a single broken object doesn't take down your operator. It backs off exponentially while other objects process normally.

## Event Recording: Making Reconciliation Visible

`tools/record` provides the `EventRecorder` interface:

```go
type EventRecorder interface {
    Event(object runtime.Object, eventtype, reason, message string)
    Eventf(object runtime.Object, eventtype, reason, messageFmt string, args ...interface{})
    AnnotatedEventf(object runtime.Object, annotations map[string]string, eventtype, reason, messageFmt string, args ...interface{})
}
```

Events are Kubernetes objects. They appear in `kubectl describe` output. They have a type (`Normal` or `Warning`), a reason (short UpperCamelCase like `FailedScheduling` or `Reconciled`), and a human-readable message.

controller-runtime's Manager creates recorders via `mgr.GetEventRecorderFor("controller-name")`. The multigres-operator passes these into every controller:

```go
if err = (&shardcontroller.ShardReconciler{
    Client:   mgr.GetClient(),
    Scheme:   mgr.GetScheme(),
    Recorder: mgr.GetEventRecorderFor("shard-controller"),
}).SetupWithManager(mgr); err != nil { ... }
```

Then inside reconciliation:

```go
r.Recorder.Eventf(shard, "Warning", "ConfigError", "Failed to generate pg_hba: %v", err)
```

This creates a Kubernetes Event tied to the Shard object. `kubectl describe shard my-shard` shows it. Events aggregate automatically -- the same reason/message combination on the same object doesn't create thousands of Event objects.

## Leader Election: One Active Replica at a Time

`tools/leaderelection` implements leader election using Kubernetes Lease objects. A `LeaderElector` performs the election and calls three callbacks:

- `OnStartedLeading(ctx)` -- this replica won the election, start doing work
- `OnStoppedLeading()` -- this replica lost the lease, stop immediately
- `OnNewLeader(identity)` -- a new leader was elected (informational)

The leader holds a Lease object and periodically renews it. If it fails to renew within the `LeaseDuration`, other replicas detect the stale lease and one of them acquires it. Three parameters control the timing:

- `LeaseDuration` -- how long a lease is valid (default 15s)
- `RenewDeadline` -- how long the leader tries to renew before giving up (default 10s)
- `RetryPeriod` -- how often non-leaders check the lease (default 2s)

controller-runtime's Manager wraps this entirely. When you set `LeaderElection: true` in the Manager options, it runs `LeaderElector` and only starts controllers after winning the election. The multigres-operator enables `LeaderElectionReleaseOnCancel` so the outgoing leader voluntarily releases the lease on clean shutdown, cutting failover time from ~15 seconds to ~2 seconds.

## Retry Utilities: Handling Optimistic Concurrency

Kubernetes uses optimistic concurrency. Every object has a `resourceVersion`. When you update an object, the API server rejects the write if the `resourceVersion` doesn't match -- someone else modified the object between your read and your write. The error is a `Conflict` (HTTP 409).

`util/retry` provides the pattern for handling this:

```go
err := retry.RetryOnConflict(retry.DefaultRetry, func() error {
    // Read the latest version
    err := r.Get(ctx, key, &obj)
    if err != nil {
        return err
    }
    // Modify it
    obj.Status.Phase = "Ready"
    // Write it back
    return r.Status().Update(ctx, &obj)
})
```

`DefaultRetry` tries 5 times with 10ms intervals and slight jitter. `DefaultBackoff` uses exponential backoff (factor 5.0) for situations where contention is heavy.

Most controller-runtime operators don't need this -- the reconciliation loop naturally handles conflicts by re-reading on the next reconcile. But when you need an atomic read-modify-write within a single reconciliation (status updates that must succeed before proceeding), `RetryOnConflict` is the tool.

## When to Reach Past Controller-Runtime

Most operators never import client-go directly. controller-runtime wraps it well enough. But there are specific situations where you need the lower layer.

**Raw clientset for pre-cache operations.** The multigres-operator creates a `kubernetes.Clientset` in its webhook setup to check for existing resources before the Manager's cache is running. The cache only starts after informers sync. If you need API access during initialization, you need a raw client.

**Port forwarding in tests.** `tools/portforward` and `transport/spdy` provide programmatic port forwarding to pods. The multigres-operator uses this in e2e tests to connect to PostgreSQL instances inside the cluster without exposing them via Services.

**Custom cache indexers.** `tools/cache` exports `MetaNamespaceKeyFunc` and the indexer interfaces. When you add a custom field index to controller-runtime's cache, you're creating an index in client-go's indexer.

**Retry logic.** `util/retry.RetryOnConflict` for atomic status updates within a single reconciliation cycle.

**Event recording types.** Even though controller-runtime provides the recorder, the `EventRecorder` interface and event types come from `tools/record`.

## The Mapping: controller-runtime to client-go

| controller-runtime | client-go |
|---|---|
| `Cache` | `tools/cache.SharedInformer` (one per GVK) |
| `Client.Get()` / `Client.List()` (reads) | `tools/cache.Indexer` lookup |
| `Client.Create()` / `Client.Update()` / `Client.Patch()` (writes) | `rest.RESTClient` HTTP calls |
| Workqueue (internal) | `util/workqueue.TypedRateLimitingInterface` |
| `Manager` leader election | `tools/leaderelection.LeaderElector` |
| `Manager.GetEventRecorderFor()` | `tools/record.EventRecorder` |
| `Builder.For()` / `Owns()` / `Watches()` | `tools/cache.SharedInformer.AddEventHandler()` |
| `ctrl.GetConfigOrDie()` | `rest.InClusterConfig()` / kubeconfig loading |

Every line in that table is a direct wrapping relationship. controller-runtime doesn't reimplement any of these. It configures them, connects them, and exposes a simpler API on top.

## The Unifying Idea

controller-runtime is not an abstraction layer that hides client-go. It's a wiring layer that connects client-go's subsystems. The Reflector fills the cache. The cache serves your reads. Event handlers push keys onto the workqueue. The workqueue feeds your Reconcile function. The rate limiter controls the backoff. The leader elector gates the whole thing behind a single active replica.

Next time your operator behaves unexpectedly -- stale reads, delayed reconciliations, leader election hiccups -- you won't be guessing at abstractions. You'll trace the problem through the pipeline: Reflector to DeltaFIFO to Indexer to event handler to workqueue to Reconcile. Six subsystems, one straight line.
