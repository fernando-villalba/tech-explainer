# Controller-Runtime: The Machine That Makes Kubernetes Operators Work

Most Kubernetes tutorials teach you that controllers "watch for events and react to them." Create a Pod, your controller gets a create event. Delete a Service, your controller gets a delete event. Event in, action out.

That model is wrong. And it's the reason most hand-rolled controllers have subtle bugs -- missed events, duplicate processing, race conditions during restarts. Controller-runtime exists because reacting to events is the wrong abstraction for building reliable controllers.

A controller doesn't react to events. It converges reality toward desired state. The entire architecture of controller-runtime -- every queue, every cache, every handler -- exists to make that convergence reliable, even when the world is chaotic.

## The Reconciler: Why You Only Get a Name ([pkg/reconcile](https://github.com/kubernetes-sigs/controller-runtime/tree/main/pkg/reconcile))

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

## What a Real Reconcile Function Looks Like

To make this concrete, look at the [multigres-operator](https://github.com/multigres/multigres-operator) -- an operator that manages sharded PostgreSQL clusters on Kubernetes. Its resource hierarchy looks like this:

```
MultigresCluster
 +-- TopoServer          (etcd for topology)
 +-- Cell                 (per failure domain)
 |    +-- MultiGateway    (Deployment + Service)
 +-- TableGroup           (per database.tablegroup)
      +-- Shard           (per shard)
           +-- Pool Pods  (PostgreSQL replicas)
           +-- MultiOrch  (Deployment)
```

The [MultigresCluster reconciler](https://github.com/multigres/multigres-operator/blob/main/pkg/cluster-handler/controller/multigrescluster/multigrescluster_controller.go) follows the exact pattern the framework expects. It starts by fetching the current state:

```go
func (r *MultigresClusterReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    cluster := &multigresv1alpha1.MultigresCluster{}
    err := r.Get(ctx, req.NamespacedName, cluster)
    if err != nil {
        if errors.IsNotFound(err) {
            return ctrl.Result{}, nil  // deleted, nothing to do
        }
        return ctrl.Result{}, fmt.Errorf("failed to get MultigresCluster: %w", err)
    }
    // ...
}
```

The `NotFound` return is the canonical pattern. The reconciler was triggered, but the object no longer exists. Maybe it was deleted between the event and the reconciliation. Maybe it never existed (queue artifact). Either way, return success and move on. The thermostat checked the temperature and there's no thermostat anymore. Done.

After the Get, the reconciler doesn't inspect any event. It reads the cluster's spec, computes what should exist (Cells, TableGroups, TopoServer, global services), compares to what does exist, and closes the gap. This is the same function whether it was triggered by a user edit, a child object change, a template update, or a resync.

## The Event Pipeline: From API Server to Workqueue

If Reconcile only gets a name, something has to translate Kubernetes events into those names. That pipeline has four stages.

**Stage 1: The Informer.** Controller-runtime doesn't poll the API server. It uses Kubernetes informers -- long-lived watch connections that stream every change for a given resource type. When a Pod is created, updated, or deleted, the informer sees it immediately. It also maintains a local cache of every object it's watching, which we'll come back to.

**Stage 2: The Predicate.** Before an event goes anywhere, predicates decide whether it matters. `GenerationChangedPredicate` drops status-only updates. `LabelSelectorPredicate` filters by labels. You can compose them with `And`, `Or`, `Not`. A predicate returning `false` means the event is silently dropped -- no reconciliation happens.

In multigres, the `MultigresCluster` controller [uses `GenerationChangedPredicate`](https://github.com/multigres/multigres-operator/blob/main/pkg/cluster-handler/controller/multigrescluster/multigrescluster_controller.go) on its primary type. When a user updates the cluster's status (or another controller patches conditions), the generation doesn't change, and the reconciler isn't called. Only spec changes trigger reconciliation. But the `Shard` controller deliberately omits this predicate -- it needs to reconcile on status changes from its children too.

**Stage 3: The Handler.** This is where events become reconcile requests. Three built-in handlers cover almost every case:

- `EnqueueRequestForObject` -- "The Pod changed. Reconcile that Pod." Enqueues the object's own name.
- `EnqueueRequestForOwner` -- "A Pod changed. Reconcile the ReplicaSet that owns it." Walks the OwnerReferences.
- `EnqueueRequestsFromMapFunc` -- "A ConfigMap changed. Reconcile every Deployment that references it." Runs your custom mapping function.

The handler pushes `reconcile.Request` values -- just names -- onto the workqueue.

**Stage 4: The Workqueue.** This is where the magic happens. The queue de-duplicates. If an object generates 50 events while you're processing it, you don't reconcile 50 times. You reconcile once more after you finish. The queue also handles rate limiting with exponential backoff -- 5ms, 10ms, 20ms, up to ~17 minutes -- for objects that keep failing. And since v0.19, the default queue is a priority queue that processes real changes before initial-list and resync events.

## The Builder: Wiring It All Together ([pkg/builder](https://github.com/kubernetes-sigs/controller-runtime/tree/main/pkg/builder))

In practice, you rarely interact with sources, handlers, and predicates directly. The builder pattern handles the wiring. Here's how multigres [sets up the MultigresCluster controller](https://github.com/multigres/multigres-operator/blob/main/pkg/cluster-handler/controller/multigrescluster/multigrescluster_controller.go):

```go
return ctrl.NewControllerManagedBy(mgr).
    For(&multigresv1alpha1.MultigresCluster{},
        builder.WithPredicates(predicate.GenerationChangedPredicate{})).
    Owns(&multigresv1alpha1.Cell{}).
    Owns(&multigresv1alpha1.TableGroup{}).
    Owns(&multigresv1alpha1.TopoServer{}).
    Owns(&appsv1.Deployment{}).
    Owns(&corev1.Service{}).
    Watches(
        &multigresv1alpha1.CoreTemplate{},
        handler.EnqueueRequestsFromMapFunc(r.enqueueRequestsFromTemplate),
    ).
    Watches(
        &multigresv1alpha1.CellTemplate{},
        handler.EnqueueRequestsFromMapFunc(r.enqueueRequestsFromTemplate),
    ).
    Watches(
        &multigresv1alpha1.ShardTemplate{},
        handler.EnqueueRequestsFromMapFunc(r.enqueueRequestsFromTemplate),
    ).
    WithOptions(controllerOpts).
    Complete(r)
```

Read this from top to bottom and the whole event pipeline becomes visible.

`For(&MultigresCluster{})` -- watch the primary type. Internally wires `EnqueueRequestForObject`. When a MultigresCluster is created or its spec changes (generation predicate), enqueue its own name.

`Owns(&Cell{})` -- watch child types. Internally wires `EnqueueRequestForOwner`. When a Cell changes, walk its OwnerReferences, find the MultigresCluster that owns it, and enqueue that cluster's name. Same for TableGroups, TopoServers, Deployments, and Services. The parent reconciles because a child changed -- but the parent never sees the child event. It reads current state and converges.

`Watches(&CoreTemplate{}, ...)` -- watch a related type with a custom handler. Templates aren't owned by clusters (many clusters can reference the same template). So `Owns()` won't work. Instead, [the custom mapper](https://github.com/multigres/multigres-operator/blob/main/pkg/cluster-handler/controller/multigrescluster/multigrescluster_controller.go) lists all clusters and checks which ones reference the changed template:

```go
func (r *MultigresClusterReconciler) enqueueRequestsFromTemplate(
    ctx context.Context, o client.Object,
) []reconcile.Request {
    templateKind := templateKindFromObject(o)
    clusters := &multigresv1alpha1.MultigresClusterList{}
    r.List(ctx, clusters, client.InNamespace(o.GetNamespace()))
    var requests []reconcile.Request
    for _, c := range clusters.Items {
        if referencesTemplate(c.Status.ResolvedTemplates, templateKind, o.GetName()) {
            requests = append(requests, reconcile.Request{
                NamespacedName: client.ObjectKeyFromObject(&c),
            })
        }
    }
    return requests
}
```

A CoreTemplate changes. The mapper finds every MultigresCluster that references it. Each of those clusters gets re-enqueued. Their Reconcile functions run, re-resolve the template, and if the resolved spec changed, update the child resources. The template author doesn't know how many clusters use their template. The cluster controller doesn't know what changed in the template. Each side reads current state and converges.

Now compare this to [the Shard controller's setup](https://github.com/multigres/multigres-operator/blob/main/pkg/resource-handler/controller/shard/shard_controller.go) -- the most complex controller in the operator:

```go
return ctrl.NewControllerManagedBy(mgr).
    For(&multigresv1alpha1.Shard{}).       // no GenerationChangedPredicate!
    Owns(&appsv1.Deployment{}).
    Owns(&corev1.Pod{}).
    Owns(&corev1.Service{}).
    Owns(&corev1.ConfigMap{}).
    Owns(&corev1.Secret{}).
    Owns(&corev1.PersistentVolumeClaim{}).
    Owns(&policyv1.PodDisruptionBudget{}).
    Watches(
        &corev1.ConfigMap{},
        handler.EnqueueRequestsFromMapFunc(r.enqueueFromPostgresConfigMap),
    ).
    WithOptions(controllerOpts).
    Complete(r)
```

Notice: no `GenerationChangedPredicate` on the `For()`. The Shard controller needs to reconcile on status-only changes too -- when pod roles update, when drain state transitions, when backup health changes. It also watches ConfigMaps with a custom mapper that [finds Shards referencing a specific postgres config](https://github.com/multigres/multigres-operator/blob/main/pkg/resource-handler/controller/shard/shard_controller.go):

```go
func (r *ShardReconciler) enqueueFromPostgresConfigMap(
    ctx context.Context, o client.Object,
) []reconcile.Request {
    shards := &multigresv1alpha1.ShardList{}
    r.List(ctx, shards, client.InNamespace(o.GetNamespace()))
    var requests []reconcile.Request
    for _, s := range shards.Items {
        if s.Spec.PostgresConfigRef != nil && s.Spec.PostgresConfigRef.Name == o.GetName() {
            requests = append(requests, reconcile.Request{
                NamespacedName: client.ObjectKeyFromObject(&s),
            })
        }
    }
    return requests
}
```

When a DBA updates a postgres config ConfigMap, every Shard that references it gets reconciled. The Shard controller computes a config hash, detects the drift, and triggers a rolling restart of the affected pods. The DBA never touched the Shard objects. The ConfigMap change cascaded through the mapper, through the queue, and into a reconciliation that reads current state and converges.

## The Cache: A Local Mirror of the Cluster ([pkg/cache](https://github.com/kubernetes-sigs/controller-runtime/tree/main/pkg/cache))

Every `client.Get()` and `client.List()` in your Reconcile function hits the cache, not the API server. The cache is backed by those same informers -- it holds a complete, indexed copy of every object you've told it to watch.

This is why controllers can handle thousands of events per second without overwhelming the API server. Reads are local memory lookups. Only writes (`Create`, `Update`, `Patch`, `Delete`) go to the API server.

The default client is a split client: reads from the cache, writes to the API server. If you need a guaranteed-fresh read (rare), the Manager exposes `GetAPIReader()` which bypasses the cache entirely. Multigres uses this for reading [external Secrets that fall outside the cache's label filter](https://github.com/multigres/multigres-operator/blob/main/pkg/resource-handler/controller/shard/reconcile_shared_infra.go) -- secrets created by other systems that don't carry the operator's labels.

One subtlety: the cache is eventually consistent. After you write an object, a subsequent `Get` might return the old version until the informer receives the watch event. Your Reconcile function should handle this gracefully -- and it does, naturally, because it will be called again when the watch event arrives.

There's much more to the cache -- multi-namespace routing, per-GVK delegation, metadata-only watches, field indexers, and transform functions that strip data before it enters memory. See the [cache deep-dive](01-cache-and-client.md) for the full picture.

## The Manager: Orchestrating the Lifecycle ([pkg/manager](https://github.com/kubernetes-sigs/controller-runtime/tree/main/pkg/manager))

The Manager is the top-level coordinator. It owns the cache, the client, the webhook server, health probes, metrics, and every controller. When you call `mgr.Start(ctx)`, it boots the entire system in a precise order:

1. HTTP servers (health probes, metrics, pprof) start first.
2. Webhook servers start next -- before the cache, to prevent deadlocks with conversion webhooks.
3. The cache starts. Every informer syncs its initial list. The system blocks until all caches report synced.
4. Non-leader-election runnables start.
5. Leader election runs. Only one replica wins. The rest block.
6. The winner starts controllers. Workers begin dequeuing and reconciling.

This ordering prevents an entire class of startup bugs. The [manager lifecycle deep-dive](03-manager-lifecycle.md) covers why each step matters, what happens during shutdown, and the failure modes you need to understand for production.

## Writing Resources: Client-Side vs Server-Side Apply

Controllers need to create and update child resources. There are three approaches, each with different trade-offs.

### The Traditional Pattern: Read-Modify-Write

The most common approach in older operators is the read-modify-write loop. Read the object, check if it exists, create it or update it:

```go
deployment := &appsv1.Deployment{}
err := r.Get(ctx, types.NamespacedName{Name: "my-app", Namespace: ns}, deployment)
if apierrors.IsNotFound(err) {
    // Doesn't exist yet -- create it
    deployment = buildDesiredDeployment()
    return r.Create(ctx, deployment)
}
// Exists -- update it
deployment.Spec.Replicas = ptr.To(int32(3))
return r.Update(ctx, deployment)
```

controller-runtime's `controllerutil` package wraps this into `CreateOrUpdate` and `CreateOrPatch`, which handle the exists-or-not branching:

```go
dep := &appsv1.Deployment{ObjectMeta: metav1.ObjectMeta{Name: "my-app", Namespace: ns}}
_, err := controllerutil.CreateOrUpdate(ctx, r.Client, dep, func() error {
    dep.Spec.Replicas = ptr.To(int32(3))
    dep.Spec.Template.Spec.Containers = []corev1.Container{{
        Name:  "app",
        Image: "my-app:v2",
    }}
    return controllerutil.SetControllerReference(owner, dep, r.Scheme)
})
```

This works, but it has a fundamental problem: the **read-modify-write race condition**. Between the `Get` and the `Update`, another actor -- another controller, a human running `kubectl edit`, a webhook, a horizontal pod autoscaler -- can modify the same object. The `Update` call sends the full object back to the API server, overwriting whatever the other actor changed. If the HPA set replicas to 5 and the operator immediately writes replicas back to 3, the HPA's change is silently lost.

Kubernetes protects against this with `resourceVersion`. If the version doesn't match, the API server returns a `Conflict` error and the operator must retry. But `CreateOrUpdate` replaces the *entire* object, so even fields the operator doesn't care about must be correct. This creates coupling between the operator and every other system that touches the same object.

### Server-Side Apply: Declare What You Own

[Server-Side Apply (SSA)](https://kubernetes.io/docs/reference/using-api/server-side-apply/) solves this by shifting the merge logic to the API server. Instead of reading the current state and deciding what to change, the operator builds the desired state and sends it. The API server figures out what's different, applies only the fields the operator manages, and leaves everything else untouched:

```go
desired := buildDesiredDeployment()
desired.SetGroupVersionKind(appsv1.SchemeGroupVersion.WithKind("Deployment"))
if err := r.Patch(ctx, desired, client.Apply,
    client.ForceOwnership,
    client.FieldOwner("my-operator"),
); err != nil {
    return fmt.Errorf("failed to apply deployment: %w", err)
}
```

One call handles create-if-missing, update-if-changed, and no-op-if-identical. No `Get` first. No `resourceVersion` to manage. No read-modify-write race.

`FieldOwner` declares which fields this caller manages. The API server tracks ownership per field. If the operator manages `spec.replicas` and the HPA also manages `spec.replicas`, SSA detects the conflict. With `ForceOwnership`, the operator wins and takes ownership of the field. Without it, the API server returns a conflict, letting the operator decide how to handle it.

This is what makes SSA declarative at the API level. The operator says "this is what I want the object to look like." The API server computes the diff. Other actors' fields are preserved. The multigres-operator uses SSA for almost all resource management -- Cells, TableGroups, TopoServers, Deployments, Services, ConfigMaps.

### When to Use Which

**SSA** is the better default for most child resource management. It eliminates race conditions, handles create-or-update in one call, and plays well with multiple actors managing different fields of the same object. Use it for any resource the operator "owns" and wants to keep converged.

**`CreateOrUpdate` / `CreateOrPatch`** still makes sense for simple cases where the operator is the only writer and the read-modify-write pattern is easier to reason about. Some operators prefer it because the mutation function in `CreateOrUpdate` explicitly shows what's being changed.

**`r.Create()` with `AlreadyExists` handling** is the right pattern for immutable resources. Pods can't be updated after creation -- they must be deleted and recreated. The multigres-operator uses this for PostgreSQL pods: create the pod, and if it already exists, move on.

**`client.MergeFrom()` patches** are for surgical changes to specific fields without touching anything else. The multigres-operator uses this for annotation updates during drain operations -- setting `DrainState` without overwriting anything else on the pod:

```go
patch := client.MergeFrom(pod.DeepCopy())
pod.Annotations[metadata.AnnotationDrainState] = metadata.DrainStateRequested
if err := r.Patch(ctx, pod, patch); err != nil { ... }
```

This is lighter than SSA when the change is a single field and you don't want to construct the full desired state.

## Ownership and Garbage Collection ([pkg/controller/controllerutil](https://github.com/kubernetes-sigs/controller-runtime/tree/main/pkg/controller/controllerutil))

Controller-runtime provides `controllerutil.SetControllerReference()` to declare that one object owns another. Multigres uses this in every builder function:

```go
func BuildCell(cluster *multigresv1alpha1.MultigresCluster, ..., scheme *runtime.Scheme,
) (*multigresv1alpha1.Cell, error) {
    cellCR := &multigresv1alpha1.Cell{ /* ... */ }
    if err := controllerutil.SetControllerReference(cluster, cellCR, scheme); err != nil {
        return nil, err
    }
    return cellCR, nil
}
```

This sets an `OwnerReference` on the Cell pointing back to the MultigresCluster, with `Controller: true`. Two things happen as a result. First, when the Cell changes, the `Owns()` handler in the builder automatically enqueues the owning MultigresCluster for reconciliation. Second, when the MultigresCluster is deleted, Kubernetes garbage collection cascades the deletion to all owned Cells.

The ownership hierarchy flows through the entire resource tree: MultigresCluster owns Cells and TableGroups, TableGroups own Shards, Shards own Pods and PVCs. Delete the root, and the entire tree is cleaned up by Kubernetes GC -- no finalizers needed.

Multigres takes this further with [dynamic PVC ownership](https://github.com/multigres/multigres-operator/blob/main/pkg/resource-handler/controller/shard/shard_controller.go). When the PVC deletion policy is `Delete`, it adds an OwnerReference so GC cascades. When the policy is `Retain`, it removes the OwnerReference so PVCs survive shard deletion. The ownership graph is not static -- it's managed as part of reconciliation.

## What Happens When Your Reconcile Returns

The return value from `Reconcile` drives the retry machinery:

| Return | What happens |
|---|---|
| `nil, nil` | Success. Item removed from queue. Rate limiter reset. |
| `Result{RequeueAfter: 5*time.Minute}, nil` | Requeue after fixed delay. Rate limiter reset. Good for polling external state. |
| `Result{}, err` | Failure. Requeued with exponential backoff (5ms -> 10ms -> 20ms -> ... -> 1000s). |
| `Result{}, reconcile.TerminalError(err)` | Permanent failure. Not requeued. Logged and counted, but the system gives up. |

The rate limiter is per-item. Object A failing doesn't slow down Object B. And `RequeueAfter` resets the backoff counter, so a temporary failure followed by a timed requeue starts fresh.

Multigres uses `RequeueAfter` extensively for different situations, each with a carefully chosen delay:

- **2 seconds** -- drain state machine in progress (pods draining, [checking every 2s](https://github.com/multigres/multigres-operator/blob/main/pkg/resource-handler/controller/shard/reconcile_data_plane.go))
- **5 seconds** -- pending graceful deletions (waiting for children to report [`ReadyForDeletion`](https://github.com/multigres/multigres-operator/blob/main/pkg/cluster-handler/controller/multigrescluster/multigrescluster_controller.go))
- **10 seconds** -- shard healthy but no primary detected yet in topology
- **30 seconds** -- child not healthy, [crash-loop detection](https://github.com/multigres/multigres-operator/blob/main/pkg/resource-handler/controller/cell/cell_controller.go)

Notice the pattern: shorter delays for active state transitions (draining), longer delays for waiting on external conditions (health). Fast enough to converge quickly, slow enough to avoid API server churn.

The operator also has a clever [grace period pattern](https://github.com/multigres/multigres-operator/blob/main/pkg/cluster-handler/controller/multigrescluster/reconcile_topology.go) for topology server unavailability. During the first 2 minutes after cluster creation, topo unavailability is silently requeued (no error, so no noisy metrics). After the grace period, it returns an error, which triggers exponential backoff and shows up in dashboards. This prevents false alerts during normal cluster bootstrapping.

## Graceful Deletion Without Finalizers

Multigres demonstrates an advanced deletion pattern that avoids finalizers entirely. Instead of using finalizers to block deletion and run cleanup, it uses a three-step annotation-based protocol:

1. The parent sets a `PendingDeletion` annotation on the child via a merge patch.
2. The child controller sees the annotation, initiates its cleanup (drains pods, unregisters from topology), and sets a `ReadyForDeletion` status condition when done.
3. The parent polls the condition and only calls `r.Delete()` when the child reports ready.

```go
// Step 1: Mark for deletion
patch := client.MergeFrom(item.DeepCopy())
item.Annotations[multigresv1alpha1.AnnotationPendingDeletion] = metav1.Now().UTC().Format(time.RFC3339)
r.Patch(ctx, item, patch)

// Step 2 (in child controller): Drain, then set condition
status.SetCondition(&shard.Status.Conditions, metav1.Condition{
    Type:   multigresv1alpha1.ConditionReadyForDeletion,
    Status: metav1.ConditionTrue,
})

// Step 3: Parent checks and deletes
if meta.IsStatusConditionTrue(item.Status.Conditions, multigresv1alpha1.ConditionReadyForDeletion) {
    r.Delete(ctx, item)
}
```

This works because the level-triggered model lets the parent poll repeatedly (via `RequeueAfter: 5*time.Second`) until the child is ready. No finalizer, no risk of a stuck finalizer blocking namespace deletion, and the cleanup logic lives in the child controller where it belongs. Kubernetes GC handles the rest via OwnerReferences.

## When Things Break

**Your Reconcile panics.** Controller-runtime catches it. The panic becomes an error. The item requeues with backoff. Your other workers keep running. The process doesn't crash. Panic recovery is on by default.

**The leader loses its lease.** The manager immediately sets the graceful shutdown timeout to zero and exits. No draining. No finishing in-progress reconciles. The binary dies fast so another replica can take over. The work queue is in-memory -- anything in it is lost. This is safe because the new leader will re-list everything and process it again. Level-triggered, not edge-triggered. The system converges. Multigres enables [`LeaderElectionReleaseOnCancel`](https://github.com/multigres/multigres-operator/blob/main/cmd/multigres-operator/main.go) so the outgoing leader voluntarily releases the lease on clean shutdown, cutting failover time from ~15 seconds to ~2 seconds.

**A CRD isn't installed yet.** The `source.Kind` poller retries `GetInformer()` every 10 seconds. Your controller blocks at startup, waiting. The moment the CRD appears, the informer syncs and workers begin. No crash loop.

**The cache serves stale data.** You update an object, immediately read it back, and get the old version. Your Reconcile function returns success because it thinks nothing changed. Then the watch event arrives, the object is re-enqueued, and Reconcile runs again with fresh data. Convergence, not correctness-on-first-try. Multigres sidesteps this for most writes by using SSA -- it doesn't read-then-write, it just applies the desired state, so cache staleness doesn't affect write correctness.

## The Unifying Idea

Controller-runtime is a convergence engine. Events trigger reconciliation, but reconciliation doesn't process events. It reads current state, compares it to desired state, and closes the gap. The queue de-duplicates so you never do redundant work. The cache keeps reads local so you never overwhelm the API server. The manager sequences startup so you never hit initialization races. The rate limiter backs off so a broken object doesn't consume all your capacity.

Desired state in, reconciliation loop running, reality converging. That's the whole machine.

Next time you write a Reconcile function, you won't be guessing what events triggered it or worrying about missed notifications. You'll read the world as it is, compare it to how it should be, and fix the difference. The framework handles everything else.

## Further Reading

- [The Cache and Client](01-cache-and-client.md) -- how reads actually work, split client, field indexers, memory pitfalls
- [Webhooks](02-webhooks.md) -- admission and conversion webhooks, the other half of controller-runtime
- [Manager Lifecycle](03-manager-lifecycle.md) -- startup ordering, shutdown, leader election, runnable groups
- [Testing with envtest](04-testing-with-envtest.md) -- running a real control plane in your tests
