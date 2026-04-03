# Client-go: The Machinery Under Controller-Runtime

When you write a Kubernetes operator with controller-runtime, you call `r.Get()` to read an object, `r.List()` to list them, `r.Update()` to write changes back. You call `ctrl.NewControllerManagedBy(mgr).For(&MyType{}).Owns(&appsv1.Deployment{}).Complete(r)` and the framework handles the rest. Events flow in. Reconcile functions run. Objects converge.

controller-runtime is a wiring layer on top of client-go. Every abstraction you use in controller-runtime maps directly to a client-go primitive. Understanding these primitives won't change how you write Reconcile functions. But it will change how you debug them.

client-go has three layers:

1. **Clients** -- how you talk to the API server (REST client, typed clientset, dynamic client)
2. **The informer pipeline** -- how the API server's state gets into local memory (Reflector, DeltaFIFO, Indexer, event handlers, workqueue)
3. **Utilities** -- tools operators use directly alongside controller-runtime (event recording, leader election, retry)

After covering these three layers, this explainer maps them to controller-runtime, compares the APIs side by side, and shows what controller-runtime saves you from building yourself.

---

# Part 1: Clients

## The REST Client ([rest](https://github.com/kubernetes/client-go/tree/master/rest))

Everything in client-go starts with `rest.Config`. This struct holds the connection to the API server: host URL, authentication (bearer token, client certificates, or auth plugins), TLS settings (CA certificate to verify the API server, and optionally client certificates for mutual TLS authentication), rate limiting, and timeout configuration.

```go
config, err := ctrl.GetConfigOrDie()
```

That one-liner from controller-runtime calls `rest.InClusterConfig()` when running in a pod (reads the service account token from `/var/run/secrets/kubernetes.io/serviceaccount/`) or falls back to the kubeconfig file on your laptop. Either way, you get a `rest.Config`.

Every client in client-go -- the typed clientset, the dynamic client, controller-runtime's client -- builds on top of a `rest.RESTClient` constructed from this config. The RESTClient handles HTTP verb mapping (GET, POST, PUT, PATCH, DELETE), content negotiation (JSON or protobuf), request retry, and rate limiting. You almost never use it directly, but it's the bottom of the stack.

## The Clientset ([kubernetes](https://github.com/kubernetes/client-go/tree/master/kubernetes))

The `kubernetes` package provides a generated, type-safe client for every built-in Kubernetes resource:

```go
clientset, err := kubernetes.NewForConfig(config)
pods, err := clientset.CoreV1().Pods("default").List(ctx, metav1.ListOptions{})
```

One method per resource per verb. `CoreV1().Pods()` returns a `PodInterface` with `Get`, `List`, `Create`, `Update`, `Delete`, `Patch`, `Watch`. The types are generated from the OpenAPI schema of the Kubernetes API. Every field is typed. Every response is deserialized into the correct Go struct.

controller-runtime replaces this with its generic `client.Client` interface -- `r.Get(ctx, key, &pod)` works for any type, built-in or custom. But the multigres-operator uses the clientset directly in specific situations: webhook handlers that need a raw client before the Manager's cache is running, and test utilities that create typed clients for cluster setup.

## The Dynamic Client ([dynamic](https://github.com/kubernetes/client-go/tree/master/dynamic))

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

---

# Part 2: The Informer Pipeline

This is the core of client-go. When controller-runtime's cache serves you an object from `r.Get()`, it's reading from an informer's local store. The informer filled that store through a pipeline with four stages, plus a workqueue that sits between the pipeline and your Reconcile function.

## Stage 1: The Reflector ([tools/cache](https://github.com/kubernetes/client-go/tree/master/tools/cache))

A Reflector keeps a local copy of a resource type in sync with the API server. It does this in two steps. First, it performs an initial LIST to get every existing object. Then it starts a long-lived WATCH connection that streams every subsequent change.

The `resourceVersion` threading between LIST and WATCH is critical. The LIST returns a `resourceVersion` representing the snapshot point. The Reflector stores this as `lastSyncResourceVersion` and passes it to the WATCH via `options.ResourceVersion = r.LastSyncResourceVersion()`. The WATCH starts from exactly that version, so every change that happened after the snapshot is delivered. No gap, no duplicates. If the watch connection drops and the server returns `410 Gone` (the version is too old for the watch history), the Reflector falls back to a full re-LIST and starts over.

The Reflector pushes every object it receives -- from the initial list and from subsequent watch events -- into a DeltaFIFO queue.

## Stage 2: The DeltaFIFO

The DeltaFIFO stores `(key, []Delta)` pairs. Each Delta has a type and the full object. The primary types are `Added`, `Updated`, `Deleted`, `Replaced`, and `Sync` (newer versions also define `ReplacedAll` and `SyncAll` for batch operations). The queue is FIFO (first-in, first-out), but it deduplicates by key: if a Pod changes three times before the consumer processes it, the queue holds all three deltas under one key, and the consumer sees them all at once.

`Replaced` and `Sync` are distinct despite looking similar:

- **`Replaced`** fires when `Replace()` is called on the DeltaFIFO -- during the initial LIST and on any full re-list triggered by a watch error (the `410 Gone` case). The entire object set is re-ingested. Objects in the store that aren't in the replacement set get a `Deleted` delta. (Historically, `Replaced` didn't exist as a separate type -- `Sync` was used for both. The `EmitDeltaTypeReplaced` flag controls which type is emitted, for backwards compatibility.)
- **`Sync`** fires on a periodic resync timer. No API server call is made. The Indexer's existing objects are re-pushed through the DeltaFIFO purely in-process. This re-drives the event handlers with the current known state, ensuring handlers eventually process every object even if a watch event was missed or a handler didn't act on a previous notification.

## Stage 3: The Indexer

The consumer pops items from the DeltaFIFO and applies them to the Indexer -- a thread-safe, indexed in-memory store. The default index is `MetaNamespaceKeyFunc`, which indexes objects by `namespace/name`. You can add custom indexes for efficient lookups by label, field, or any computed key.

This is the store that controller-runtime's cache reads from. When you call `r.Get(ctx, types.NamespacedName{Name: "foo", Namespace: "bar"}, &obj)`, you're doing a lookup in this indexer by the `namespace/name` key. No network call. When you call `r.List()` with a field selector, controller-runtime can use custom indexes (added via `mgr.GetFieldIndexer().IndexField(...)`) to do an indexed lookup instead of scanning every object in the store. This is why operator setup code sometimes pre-computes field indexes -- it turns O(n) list operations into O(1) lookups.

## Stage 4: The Event Handlers

After updating the store, the informer calls registered `ResourceEventHandler` callbacks:

```go
type ResourceEventHandler interface {
    OnAdd(obj interface{}, isInInitialList bool)
    OnUpdate(oldObj, newObj interface{})
    OnDelete(obj interface{})
}
```

The `isInInitialList` parameter on `OnAdd` lets handlers distinguish between objects seen during the initial LIST sync and objects that arrived via a live WATCH event. controller-runtime uses this internally to track whether the cache has fully synced -- `cache.WaitForCacheSync()` blocks until every initial-list object has been processed, which is what prevents controllers from reconciling before the cache has a complete picture.

controller-runtime registers handlers that extract the object's key (or the owner's key, depending on the watch configuration) and push it onto the workqueue. This is how "Pod changed" becomes "reconcile the ReplicaSet that owns this Pod."

## Stage 5: The Workqueue ([util/workqueue](https://github.com/kubernetes/client-go/tree/master/util/workqueue))

The handlers don't call Reconcile directly. They push keys onto a rate-limited workqueue. This is the final stage before your code runs.

client-go provides three queue interfaces, each layering on the previous:

**TypedInterface** -- the base queue. `Add(item)`, `Get()`, `Done(item)`, `ShutDown()`. The critical property: *deduplication*. If you `Add("default/my-pod")` while that key is already in the `dirty` set, it won't be queued again (though `Touch()` may be called to adjust priority if the queue implementation supports it). If the item is currently being processed and gets re-added, it's marked dirty and will be re-queued after the current processing completes. This is why controller-runtime can safely enqueue the same object many times -- rapid-fire events don't cause rapid-fire reconciliations.

**TypedDelayingInterface** -- adds `AddAfter(item, duration)`. The item enters the queue after the delay. This powers `Result{RequeueAfter: 5 * time.Second}` in your Reconcile function.

**TypedRateLimitingInterface** -- adds `AddRateLimited(item)`, `Forget(item)`, `NumRequeues(item)`. The rate limiter decides how long to delay based on failure count. This powers the exponential backoff when your Reconcile returns an error.

The default rate limiter for controllers combines two strategies:

1. **Per-item exponential backoff.** Starts at 5 milliseconds, doubles each failure, caps at 1000 seconds (~17 minutes). If object "default/my-pod" fails reconciliation, it's requeued after 5ms, then 10ms, then 20ms, and so on. Each object tracks its own failure count independently.

2. **Overall token bucket.** 10 requests per second with a burst of 100. This prevents a flood of failing objects from consuming all processing capacity.

When Reconcile succeeds (returns `nil, nil`), controller-runtime calls `Forget(item)`, which resets the exponential backoff counter for that item. The next failure starts back at 5ms. When Reconcile returns `RequeueAfter`, it also calls `Forget` -- the explicit delay replaces the backoff.

This is why a single broken object doesn't take down your operator. It backs off exponentially while other objects process normally.

## The SharedInformer

A SharedInformer wraps Stages 1 through 4 into one unit. "Shared" means one informer per GVK is shared across all controllers that watch that type. If three controllers all watch Pods, they share one Pod informer -- one LIST, one WATCH connection, one store. Each controller registers its own event handler on the shared informer.

controller-runtime's cache creates and manages these shared informers. When you call `For(&MyType{})` in the builder, the cache ensures a SharedInformer exists for `MyType`. When you call `Owns(&appsv1.Deployment{})`, it ensures a SharedInformer exists for Deployments.

---

# Part 3: Utilities

These are client-go packages that operators use directly, even when using controller-runtime. controller-runtime wraps some of them (event recording, leader election) but exposes the same interfaces. Others (retry) aren't wrapped at all.

## Event Recording ([tools/record](https://github.com/kubernetes/client-go/tree/master/tools/record))

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

## Leader Election ([tools/leaderelection](https://github.com/kubernetes/client-go/tree/master/tools/leaderelection))

`tools/leaderelection` implements leader election using Kubernetes Lease objects. A `LeaderElector` performs the election and calls three callbacks:

- `OnStartedLeading(ctx)` -- this replica won the election, start doing work
- `OnStoppedLeading()` -- this replica lost the lease, stop immediately
- `OnNewLeader(identity)` -- a new leader was elected (informational)

The leader holds a Lease object and periodically renews it. If it fails to renew within the `LeaseDuration`, other replicas detect the stale lease and one of them acquires it. Three parameters control the timing:

- `LeaseDuration` -- how long a lease is valid (default 15s in controller-runtime)
- `RenewDeadline` -- how long the leader tries to renew before giving up (default 10s)
- `RetryPeriod` -- how often non-leaders check the lease (default 2s)

controller-runtime's Manager wraps this entirely. When you set `LeaderElection: true` in the Manager options, it runs `LeaderElector` and only starts controllers after winning the election. The multigres-operator enables `LeaderElectionReleaseOnCancel` so the outgoing leader voluntarily releases the lease on clean shutdown, cutting failover time from ~15 seconds to ~2 seconds.

## Retry Utilities ([util/retry](https://github.com/kubernetes/client-go/tree/master/util/retry))

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

`DefaultRetry` tries 5 times with flat 10ms intervals and slight jitter (factor 1.0, so no exponential growth -- just 5 quick retries). `DefaultBackoff` uses exponential backoff (factor 5.0, so 10ms, 50ms, 250ms, 1250ms) for situations where contention is heavy and backing off gives other writers time to finish.

Most controller-runtime operators don't need this -- the reconciliation loop naturally handles conflicts by re-reading on the next reconcile. But when you need an atomic read-modify-write within a single reconciliation (status updates that must succeed before proceeding), `RetryOnConflict` is the tool.

---

# Part 4: client-go and controller-runtime

## How Much Do Real Operators Use client-go Directly?

More than you'd think. controller-runtime wraps the core loop, but most production operators still import client-go for things controller-runtime doesn't cover. A survey of major operators:

- **[multigres-operator](https://github.com/multigres/multigres-operator)**: 41 files import client-go directly. Uses `kubernetes.Clientset` for pre-cache operations, `tools/record` for event recording, `tools/cache` for custom indexers, `util/retry` for conflict retries, `tools/portforward` and `transport/spdy` for e2e test port forwarding.
- **[CloudNative-PG](https://github.com/cloudnative-pg/cloudnative-pg)**: 14 client-go packages. Uses `discovery` for API discovery, `dynamic` for unstructured access, `tools/remotecommand` for exec into pods, `util/retry`.
- **[Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator)**: 27 client-go packages. Uses raw `informers`, `listers`, `util/workqueue`, typed clients (`typed/core/v1`, `typed/apps/v1`), `discovery`, `metadata` informers. Large parts of the codebase work directly with client-go alongside controller-runtime.
- **[cert-manager](https://github.com/cert-manager/cert-manager)**: 38 client-go packages. Uses `informers`, `listers`, `tools/leaderelection` directly, `applyconfigurations` for SSA, `util/workqueue`, `metadata` informers and listers. The most extensive direct client-go usage of any operator surveyed.
- **[Zalando Postgres Operator](https://github.com/zalando/postgres-operator)**: Built entirely on client-go without controller-runtime.

The pattern: controller-runtime handles the reconciliation loop and cache, but operators reach into client-go for typed clients, raw informers, discovery, retry utilities, event recording, remote execution, and port forwarding. The more complex the operator, the more client-go surfaces directly in the codebase.

## Common Operations: controller-runtime vs client-go

| Operation | controller-runtime | client-go equivalent |
|---|---|---|
| Get an object | `r.Get(ctx, key, &pod)` | `clientset.CoreV1().Pods(ns).Get(ctx, name, metav1.GetOptions{})` |
| List objects | `r.List(ctx, &podList, client.InNamespace(ns))` | `clientset.CoreV1().Pods(ns).List(ctx, metav1.ListOptions{})` |
| Create an object | `r.Create(ctx, &pod)` | `clientset.CoreV1().Pods(ns).Create(ctx, &pod, metav1.CreateOptions{})` |
| Update an object | `r.Update(ctx, &pod)` | `clientset.CoreV1().Pods(ns).Update(ctx, &pod, metav1.UpdateOptions{})` |
| Patch an object | `r.Patch(ctx, &pod, patch)` | `clientset.CoreV1().Pods(ns).Patch(ctx, name, patchType, data, metav1.PatchOptions{})` |
| Server-Side Apply | `r.Patch(ctx, obj, client.Apply, client.FieldOwner("x"))` | `clientset.CoreV1().Pods(ns).Apply(ctx, applyConfig, metav1.ApplyOptions{FieldManager: "x"})` |
| Delete an object | `r.Delete(ctx, &pod)` | `clientset.CoreV1().Pods(ns).Delete(ctx, name, metav1.DeleteOptions{})` |
| Update status | `r.Status().Update(ctx, &obj)` | `clientset.CoreV1().Pods(ns).UpdateStatus(ctx, &pod, metav1.UpdateOptions{})` |
| Watch for changes | `Builder.For(&Type{}).Owns(&Child{})` | `informer.AddEventHandler(cache.ResourceEventHandlerFuncs{...})` |
| Record an event | `recorder.Eventf(obj, "Normal", "Reason", "msg")` | `recorder.Eventf(obj, "Normal", "Reason", "msg")` (same interface) |
| Retry on conflict | Not built in (use `util/retry`) | `retry.RetryOnConflict(retry.DefaultRetry, func() error { ... })` |
| Get cluster config | `ctrl.GetConfigOrDie()` | `rest.InClusterConfig()` or `clientcmd.BuildConfigFromFlags()` |
| Leader election | `Manager` option: `LeaderElection: true` | `leaderelection.NewLeaderElector(config)` + manual callbacks |

The syntax looks similar in the table, but that hides the real difference. The table shows individual operations. controller-runtime's value is in all the wiring you'd have to build yourself if you only had client-go.

## What controller-runtime Saves You From

Building an operator on raw client-go means writing and maintaining all of the following yourself:

- **Informer lifecycle management.** For each resource type you watch, you need to create a `SharedInformerFactory`, start the informers, wait for cache sync, and handle shutdown. controller-runtime's Manager does this automatically for every type registered through `For()`, `Owns()`, and `Watches()`.

- **Workqueue wiring.** You need to create a rate-limiting workqueue, wire event handlers from each informer to enqueue the right keys (the object itself, its owner, or a custom mapping), handle the `Get`/`Done`/`Forget` lifecycle for each item, and implement the worker loop that dequeues and calls your reconciliation logic. controller-runtime's Builder and internal controller handle all of this.

- **Cache-backed reads with API server writes.** With raw client-go, you either read from the informer's lister (cached, eventually consistent) or from the API server (fresh, expensive). You have to choose per call. controller-runtime's split client does this transparently: `Get` and `List` hit the cache, `Create`/`Update`/`Patch`/`Delete` hit the API server.

- **Startup sequencing.** HTTP servers (health, metrics, pprof) need to start before the cache. Webhooks need to start before the cache (to avoid conversion webhook deadlocks). The cache needs to sync before controllers start. Leader election needs to gate the controllers. With client-go you sequence all of this manually. controller-runtime's Manager handles the six-phase startup in the correct order.

- **Leader election integration.** With client-go you create a `LeaderElector`, configure the lease parameters, write the `OnStartedLeading`/`OnStoppedLeading` callbacks, and make sure your controllers only run when leading. With controller-runtime it's `LeaderElection: true` in the Manager options.

- **Health probes and metrics.** With client-go you set up HTTP servers, register health check handlers, wire Prometheus metrics collectors, and manage TLS. controller-runtime's Manager provides all of this out of the box.

- **Webhook server with TLS.** Running an admission webhook requires a TLS server, certificate loading, request decoding, response encoding, and path routing. controller-runtime's webhook package handles the HTTP plumbing and lets you implement a single `Default()` or `ValidateCreate()` method.

- **Graceful shutdown.** Stopping controllers, draining workqueues, closing informer watch connections, releasing leader leases, and shutting down HTTP servers in the right order. controller-runtime's Manager reverses the startup sequence on context cancellation.

The Zalando Postgres Operator, built directly on client-go, implements all of this manually. It works, but it's a significant amount of infrastructure code that every controller-runtime operator gets for free. For most operators, that trade-off isn't worth it. controller-runtime exists so that the reconciliation logic is the only code you have to write.

## The Architecture Mapping

| controller-runtime abstraction | client-go primitive underneath |
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

---

## The Unifying Idea

controller-runtime is not an abstraction layer that hides client-go. It's a wiring layer that connects client-go's subsystems. The Reflector fills the cache. The cache serves your reads. Event handlers push keys onto the workqueue. The workqueue feeds your Reconcile function. The rate limiter controls the backoff. The leader elector gates the whole thing behind a single active replica.

Next time your operator behaves unexpectedly -- stale reads, delayed reconciliations, leader election hiccups -- you won't be guessing at abstractions. You'll trace the problem through the pipeline: Reflector to DeltaFIFO to Indexer to event handler to workqueue to Reconcile.
