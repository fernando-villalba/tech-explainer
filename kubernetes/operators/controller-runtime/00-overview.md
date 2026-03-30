# Controller-Runtime: The Machine That Makes Kubernetes Operators Work

Most Kubernetes tutorials teach you that controllers "watch for events and react to them." Create a Pod, your controller gets a create event. Delete a Service, your controller gets a delete event. Event in, action out.

That model is wrong. And it's the reason most hand-rolled controllers have subtle bugs -- missed events, duplicate processing, race conditions during restarts. Controller-runtime exists because reacting to events is the wrong abstraction for building reliable controllers.

A controller doesn't react to events. It converges reality toward desired state. The entire architecture of controller-runtime -- every queue, every cache, every handler -- exists to make that convergence reliable, even when the world is chaotic.

## The Reconciler: Why You Only Get a Name

Here is the core interface. Everything else exists to serve it:

```go
type Reconciler interface {
    Reconcile(ctx context.Context, req Request) (Result, error)
}
```

And `Request` is:

```go
type Request struct {
    types.NamespacedName  // just Name + Namespace
}
```

No event type. No old object. No new object. No diff. Just a name.

This shocks people the first time they see it. You don't know *what* happened. You don't know *if* something happened. You just know: "something about this object may have changed. Go figure it out."

This is the single most important design decision in controller-runtime. Every architectural choice flows from it.

Why? Because when you get a name instead of an event, you are forced to read the current state and compare it to the desired state. You can't react to a stale event. You can't process events out of order. You can't accidentally skip a delete because you were processing a create. You read reality. You compare it to what should be. You fix the difference.

This is level-triggered reconciliation, not edge-triggered. A thermostat doesn't care *how many times* the temperature changed or *which direction* it went. It reads the current temperature, compares it to the target, and turns the heater on or off. Your Reconcile function is a thermostat.

## The Event Pipeline: From API Server to Workqueue

If Reconcile only gets a name, something has to translate Kubernetes events into those names. That pipeline has four stages.

**Stage 1: The Informer.** Controller-runtime doesn't poll the API server. It uses Kubernetes informers -- long-lived watch connections that stream every change for a given resource type. When a Pod is created, updated, or deleted, the informer sees it immediately. It also maintains a local cache of every object it's watching, which we'll come back to.

**Stage 2: The Predicate.** Before an event goes anywhere, predicates decide whether it matters. `GenerationChangedPredicate` drops status-only updates. `LabelSelectorPredicate` filters by labels. You can compose them with `And`, `Or`, `Not`. A predicate returning `false` means the event is silently dropped -- no reconciliation happens.

**Stage 3: The Handler.** This is where events become reconcile requests. Three built-in handlers cover almost every case:

- `EnqueueRequestForObject` -- "The Pod changed. Reconcile that Pod." Enqueues the object's own name.
- `EnqueueRequestForOwner` -- "A Pod changed. Reconcile the ReplicaSet that owns it." Walks the OwnerReferences.
- `EnqueueRequestsFromMapFunc` -- "A ConfigMap changed. Reconcile every Deployment that references it." Runs your custom mapping function.

The handler pushes `reconcile.Request` values -- just names -- onto the workqueue.

**Stage 4: The Workqueue.** This is where the magic happens. The queue de-duplicates. If an object generates 50 events while you're processing it, you don't reconcile 50 times. You reconcile once more after you finish. The queue also handles rate limiting with exponential backoff -- 5ms, 10ms, 20ms, up to ~17 minutes -- for objects that keep failing. And since v0.19, the default queue is a priority queue that processes real changes before initial-list and resync events.

If this still feels abstract, trace a concrete scenario. A user updates a Deployment's image tag. The Deployment informer fires an Update event. The predicate checks that `.metadata.generation` changed (it did -- spec changed). The handler enqueues `{Namespace: "default", Name: "my-app"}`. Your worker goroutine dequeues it. Your Reconcile function runs. You `Get` the Deployment from the cache, see the desired image, check whether the current Pods match, and if not, kick off a rollout.

## The Cache: A Local Mirror of the Cluster

Every `client.Get()` and `client.List()` in your Reconcile function hits the cache, not the API server. The cache is backed by those same informers -- it holds a complete, indexed copy of every object you've told it to watch.

This is why controllers can handle thousands of events per second without overwhelming the API server. Reads are local memory lookups. Only writes (`Create`, `Update`, `Patch`, `Delete`) go to the API server.

The default client is a split client: reads from the cache, writes to the API server. If you need a guaranteed-fresh read (rare), the Manager exposes `GetAPIReader()` which bypasses the cache entirely.

One subtlety: the cache is eventually consistent. After you `Update` an object, a subsequent `Get` might return the old version until the informer receives the watch event. Your Reconcile function should handle this gracefully -- and it does, naturally, because it will be called again when the watch event arrives.

There's much more to the cache -- multi-namespace routing, per-GVK delegation, metadata-only watches, field indexers, and transform functions that strip data before it enters memory. See the [cache deep-dive](01-cache-and-client.md) for the full picture.

## The Manager: Orchestrating the Lifecycle

The Manager is the top-level coordinator. It owns the cache, the client, the webhook server, health probes, metrics, and every controller. When you call `mgr.Start(ctx)`, it boots the entire system in a precise order:

1. HTTP servers (health probes, metrics, pprof) start first.
2. Webhook servers start next -- before the cache, to prevent deadlocks with conversion webhooks.
3. The cache starts. Every informer syncs its initial list. The system blocks until all caches report synced.
4. Non-leader-election runnables start.
5. Leader election runs. Only one replica wins. The rest block.
6. The winner starts controllers. Workers begin dequeuing and reconciling.

This ordering prevents an entire class of startup bugs. The [manager lifecycle deep-dive](03-manager-lifecycle.md) covers why each step matters, what happens during shutdown, and the failure modes you need to understand for production.

## The Builder: Wiring It All Together

In practice, you rarely interact with sources, handlers, and predicates directly. The builder pattern handles the wiring:

```go
ctrl.NewControllerManagedBy(mgr).
    For(&myv1.MyResource{}).                                    // watch your primary type
    Owns(&corev1.Pod{}).                                        // watch children, reconcile the parent
    Watches(&corev1.ConfigMap{}, handler.EnqueueRequestsFromMapFunc(mapFn)).
    WithEventFilter(predicate.GenerationChangedPredicate{}).    // only spec changes
    Complete(&MyReconciler{})
```

`For()` wires `EnqueueRequestForObject`. `Owns()` wires `EnqueueRequestForOwner`. `Watches()` lets you specify any handler. `Complete()` creates the controller, registers it with the manager, and sets up all the watches.

Four lines of setup replace hundreds of lines of client-go boilerplate -- informer factories, work queues, event handler registrations, cache sync waiting. The builder doesn't hide complexity. It removes it.

## What Happens When Your Reconcile Returns

The return value from `Reconcile` drives the retry machinery:

| Return | What happens |
|---|---|
| `nil, nil` | Success. Item removed from queue. Rate limiter reset. |
| `Result{RequeueAfter: 5*time.Minute}, nil` | Requeue after fixed delay. Rate limiter reset. Good for polling external state. |
| `Result{}, err` | Failure. Requeued with exponential backoff (5ms -> 10ms -> 20ms -> ... -> 1000s). |
| `Result{}, reconcile.TerminalError(err)` | Permanent failure. Not requeued. Logged and counted, but the system gives up. |

The rate limiter is per-item. Object A failing doesn't slow down Object B. And `RequeueAfter` resets the backoff counter, so a temporary failure followed by a timed requeue starts fresh.

## When Things Break

**Your Reconcile panics.** Controller-runtime catches it. The panic becomes an error. The item requeues with backoff. Your other workers keep running. The process doesn't crash. Panic recovery is on by default.

**The leader loses its lease.** The manager immediately sets the graceful shutdown timeout to zero and exits. No draining. No finishing in-progress reconciles. The binary dies fast so another replica can take over. The work queue is in-memory -- anything in it is lost. This is safe because the new leader will re-list everything and process it again. Level-triggered, not edge-triggered. The system converges.

**A CRD isn't installed yet.** The `source.Kind` poller retries `GetInformer()` every 10 seconds. Your controller blocks at startup, waiting. The moment the CRD appears, the informer syncs and workers begin. No crash loop.

**The cache serves stale data.** You update an object, immediately read it back, and get the old version. Your Reconcile function returns success because it thinks nothing changed. Then the watch event arrives, the object is re-enqueued, and Reconcile runs again with fresh data. Convergence, not correctness-on-first-try.

## The Unifying Idea

Controller-runtime is a convergence engine. Events trigger reconciliation, but reconciliation doesn't process events. It reads current state, compares it to desired state, and closes the gap. The queue de-duplicates so you never do redundant work. The cache keeps reads local so you never overwhelm the API server. The manager sequences startup so you never hit initialization races. The rate limiter backs off so a broken object doesn't consume all your capacity.

Desired state in, reconciliation loop running, reality converging. That's the whole machine.

Next time you write a Reconcile function, you won't be guessing what events triggered it or worrying about missed notifications. You'll read the world as it is, compare it to how it should be, and fix the difference. The framework handles everything else.

## Further Reading

- [The Cache and Client](01-cache-and-client.md) -- how reads actually work, split client, field indexers, memory pitfalls
- [Webhooks](02-webhooks.md) -- admission and conversion webhooks, the other half of controller-runtime
- [Manager Lifecycle](03-manager-lifecycle.md) -- startup ordering, shutdown, leader election, runnable groups
- [Testing with envtest](04-testing-with-envtest.md) -- running a real control plane in your tests
