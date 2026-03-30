# The Cache and Client: Your Reads Never Hit the API Server

When you call `client.Get()` inside a Reconcile function, you probably think it talks to the Kubernetes API server. It doesn't. It reads from a local in-memory copy of every object your controller is watching. That local copy is the cache, and understanding how it works -- its topology, its consistency model, and its failure modes -- is the difference between a controller that handles 10 objects and one that handles 10,000.

## The Split Client

The default client that the Manager gives you is a split client. It routes reads and writes to different backends.

```
client.Get()    --> cache (local memory)
client.List()   --> cache (local memory)
client.Create() --> API server (direct HTTP)
client.Update() --> API server (direct HTTP)
client.Patch()  --> API server (direct HTTP)
client.Delete() --> API server (direct HTTP)
```

Reads are fast. Microsecond-level local lookups against an indexed store. Writes are the real thing -- REST calls to the Kubernetes API server, with all the latency, admission, and validation that implies.

The split client decides where to route each read based on three rules, checked in order:

1. If no cache was configured, go to the API server.
2. If the object's GVK is in the `DisableFor` list, go to the API server.
3. If the object is `Unstructured` and `cacheUnstructured` is false (the default), go to the API server.

Everything else goes to the cache.

If you ever need a guaranteed-fresh read -- for example, to check a condition that must be current before a destructive action -- the Manager exposes `GetAPIReader()`. This returns a client that always talks to the API server directly.

The [multigres-operator](https://github.com/multigres/multigres-operator) uses `GetAPIReader()` for a specific and instructive reason. The Shard controller needs to read external Secrets -- TLS certificates and credentials created by cert-manager or other external systems. These Secrets don't carry the operator's labels, so they're invisible to the label-filtered cache (more on that below). The [Shard reconciler stores the API reader](https://github.com/multigres/multigres-operator/blob/main/pkg/resource-handler/controller/shard/shard_controller.go) and uses it for these external reads:

```go
type ShardReconciler struct {
    client.Client
    APIReader client.Reader  // uncached, bypasses label filter
    // ...
}

// In reconciliation:
secret := &corev1.Secret{}
if err := r.APIReader.Get(ctx, types.NamespacedName{
    Name:      secretName,
    Namespace: shard.Namespace,
}, secret); err != nil { ... }
```

This is not about freshness. It's about visibility. The cached client can't see objects that were filtered out at the informer level. The API reader bypasses the cache entirely and hits the API server directly.

## What the Cache Actually Is

The cache is a collection of informers. One informer per GVK (GroupVersionKind). Each informer maintains a long-lived WATCH connection to the API server and holds a complete copy of every object of that type in its local store.

When you call `client.Get(ctx, key, &corev1.Pod{})`, the cache resolves `Pod` to its GVK, finds the Pod informer, and reads from its local indexer. No network call. When a Pod is created, updated, or deleted in the cluster, the informer's watch connection delivers the event, and the local store is updated.

Three things follow from this design.

First, the cache is eventually consistent. After you `client.Update()` an object, the next `client.Get()` might return the old version. The update goes to the API server, but the cache won't see the change until the informer's watch stream delivers it. In practice this delay is milliseconds, but it's real. Your Reconcile function handles this naturally -- it will be called again when the watch event arrives.

Second, the cache is complete for watched types. It holds every object of every type you're watching, across all namespaces (unless you've configured namespace restrictions or label filters). This means `client.List()` against the cache returns the full set, not a paginated subset. There is no `Continue` token support in cache reads.

Third, the cache creates API server load proportional to the number of watched types, not the number of reads. Watching 5 types means 5 watch connections, regardless of whether you read those objects 10 times per second or 10,000 times per second. Reads are free.

## How Informers Are Created

By default, informers are created lazily. The first time you `Get` or `List` a type that has no informer, the cache silently creates one, starts a LIST+WATCH, waits for sync, and serves the result. Convenient for development. Dangerous for production.

The danger is memory. Every informer holds a complete copy of every object of that type in the cluster. One stray `client.Get(ctx, key, &corev1.Secret{})` in a utility function, and you've just created a permanent informer that caches every Secret in the cluster. If you have 50,000 Secrets with large data payloads, you've just pinned hundreds of megabytes of memory.

The fix is `ReaderFailOnMissingInformer`:

```go
cache.Options{
    ReaderFailOnMissingInformer: true,
}
```

With this enabled, a `Get` or `List` for a type without an informer returns `ErrResourceNotCached` immediately instead of creating one. This forces explicit informer setup -- either by watching the type in a controller, or by calling `GetInformer()` directly. No surprises, no runaway memory.

## Filtering the Cache: Label Selectors Per Type

For operators that own child resources in shared namespaces, caching *every* object of a given type is wasteful. If your operator creates 50 Pods but the namespace has 5,000, you're caching 4,950 objects you'll never read.

The `ByObject` cache option lets you filter per GVK. The [multigres-operator demonstrates this well](https://github.com/multigres/multigres-operator/blob/main/cmd/multigres-operator/main.go). It applies a label selector so that only objects managed by multigres are cached:

```go
managedBySelector := labels.SelectorFromSet(labels.Set{
    "app.kubernetes.io/managed-by": "multigres-operator",
})

mgr, err := ctrl.NewManager(cfg, ctrl.Options{
    Cache: cache.Options{
        ByObject: map[client.Object]cache.ByObject{
            &corev1.Secret{}:      {Label: managedBySelector},
            &appsv1.StatefulSet{}: {Label: managedBySelector},
            &corev1.Service{}:     {Label: managedBySelector},
            &corev1.Pod{}:         {Label: managedBySelector},
            // ConfigMaps intentionally left unfiltered
        },
    },
})
```

The informer for Secrets only watches Secrets with `app.kubernetes.io/managed-by: multigres-operator`. The LIST and WATCH calls include this label selector, so the API server only sends matching objects. Memory usage drops from "all Secrets in the cluster" to "Secrets this operator created."

Notice that ConfigMaps are intentionally left unfiltered. The Shard controller watches external postgres config ConfigMaps that are created by users, not by the operator. These don't carry the operator's label, so filtering them out would break the watch. This is a deliberate architectural decision -- not an oversight.

The trade-off of label filtering: if your code tries to read an object that exists in the cluster but doesn't match the selector, the cache returns NotFound. The object is real, but the cache can't see it. This is exactly why multigres uses `GetAPIReader()` for external Secrets -- they exist, they just don't match the label filter.

## Cache Topology: Three Architectures

The cache construction function looks at your options and picks one of three architectures.

**Plain informerCache.** The default. One informer per GVK, all watching all namespaces. Simple, effective, and the right choice for most controllers.

**Multi-namespace cache.** Activated when you set `DefaultNamespaces`. Creates a separate informer per namespace per GVK, plus a cluster-scoped cache for non-namespaced resources. Each namespace's informer only watches that namespace, so the API server sends less data.

The catch: cross-namespace `List` calls aggregate results from all per-namespace caches. The `resourceVersion` on the aggregated list comes from whichever namespace the map iteration hits last -- effectively arbitrary. And `Limit` is distributed across caches in iteration order, so some namespaces dominate. For most use cases this doesn't matter. If it does, you need to know.

**Delegating-by-GVK cache.** Activated when you set `ByObject` (as multigres does). Routes each GVK to its own cache with its own configuration -- different label selectors, different transforms, different namespace restrictions per type. GVKs without explicit configuration fall through to a default cache.

You can combine these. A `ByObject` entry can itself have namespace-specific configuration, producing a per-GVK multi-namespace cache nested inside the delegating layer.

Multigres takes the `ByObject` approach further by [applying different configurations per namespace](https://github.com/multigres/multigres-operator/blob/main/cmd/multigres-operator/main.go). The operator's own namespace gets an unfiltered cache (it needs to see its own Secrets, ConfigMaps, etc.), while all other namespaces get the label-filtered cache. This hybrid strategy minimizes memory while preserving full visibility in the operator's namespace.

## Field Indexers: Making List Fast

By default, `client.List()` with a field selector does a full scan of the cache. To make it index-backed, you register a field indexer before the cache starts:

```go
mgr.GetFieldIndexer().IndexField(ctx, &corev1.Pod{}, ".spec.nodeName",
    func(o client.Object) []string {
        return []string{o.(*corev1.Pod).Spec.NodeName}
    },
)
```

Now `client.List(ctx, &podList, client.MatchingFields{".spec.nodeName": "node-1"})` does an index lookup instead of a scan.

Three things to know about field indexers.

First, only exact-match field selectors are supported. If you try `!=` or `in`, the cache returns an error. This is a hard limitation of the indexer, not something that will be fixed later.

Second, multi-field selectors are partially indexed. If you filter on two fields, only the first one uses the index. The second is a linear scan over the first result set. If you need to filter on two fields efficiently, register a composite index that concatenates both values.

Third, for namespaced objects, the indexer stores each value twice: once prefixed with the namespace (`ns/value`) and once with `__all_namespaces/value`. This lets both namespace-scoped and cross-namespace queries hit the index. You don't need to handle this yourself -- it's automatic. But it does mean index memory is roughly 2x what you'd expect for namespaced objects.

## Transforms: Stripping Data Before It Enters the Cache

Transform functions run on every object before it enters the cache. The most common use case is stripping `managedFields`:

```go
cache.Options{
    DefaultTransform: cache.TransformStripManagedFields(),
}
```

Managed fields track which field manager owns which field. For objects managed by multiple tools (kubectl, server-side apply, Helm), this metadata can be 2-10x the size of the object itself. Stripping it before caching cuts memory dramatically.

The trade-off: if your reconciler reads `obj.GetManagedFields()`, it will get `nil`. The transform removes the data before your code ever sees it. This is a silent failure -- no error, no warning, just missing data.

You can set transforms per-GVK via `ByObject[obj].Transform` or per-namespace. The transform runs inside the informer, so it affects every reader of the cache, not just your controller.

## Metadata-Only Watches: Caching Less

For types where you only need metadata (name, namespace, labels, annotations, owner references), you can watch with `OnlyMetadata`:

```go
ctrl.NewControllerManagedBy(mgr).
    For(&myv1.MyResource{}).
    Watches(&corev1.ConfigMap{}, handler, builder.OnlyMetadata).
    Complete(reconciler)
```

This tells the informer to use the metadata API, which returns only the metadata portion of each object. For ConfigMaps with large data payloads or Secrets with binary content, the memory savings can be 90%+.

There is one critical trap. If you watch ConfigMaps with `OnlyMetadata` but then call `client.Get(ctx, key, &corev1.ConfigMap{})` with a full typed object, the cache sees a structured type, looks in the structured tracker (not the metadata tracker), and auto-creates a *second* informer watching full ConfigMaps. You now have two informers for the same resource, doubling memory. Worse, the two caches sync independently -- you can get stale reads because the metadata cache and the structured cache are at different points in the watch stream.

The fix: when reading objects watched metadata-only, use `&metav1.PartialObjectMetadata{}` with the GVK set:

```go
obj := &metav1.PartialObjectMetadata{}
obj.SetGroupVersionKind(schema.GroupVersionKind{Group: "", Version: "v1", Kind: "ConfigMap"})
client.Get(ctx, key, obj)
```

Note that multigres doesn't use `OnlyMetadata` -- it needs full ConfigMap data (postgres config contents) in its reconciliation logic. If it only needed to detect ConfigMap changes (not read their contents), switching to metadata-only watches and reading the full object via `APIReader` only when needed would cut memory significantly. This is a common optimization pattern: watch metadata-only for change detection, read full objects on demand.

## The Resync Period

The cache has a `SyncPeriod` (default 10 hours). When it fires, the informer sends artificial Update events for every cached object, with `ObjectOld` and `ObjectNew` set to the same value. This is a safety net -- if a watch event was dropped, the resync ensures your controller eventually sees every object.

If you use `GenerationChangedPredicate`, resync events are silently dropped because the generation didn't change. This is usually what you want. Multigres uses `GenerationChangedPredicate` on its MultigresCluster, Cell, TableGroup, and TopoServer controllers -- they only reconcile on spec changes, so resync events are noise. But the Shard controller omits this predicate, meaning it receives resync events and re-runs its full reconciliation periodically. This acts as a drift-detection mechanism for pod roles, backup health, and drain state.

The priority queue deprioritizes resync events (priority -100), so they're processed after real changes. This prevents a resync wave from blocking actual work.

## Eventual Consistency: The Practical Implications

The gap between writing to the API server and seeing the result in the cache is usually milliseconds. But it's not zero, and it creates patterns you need to internalize.

**Pattern 1: Update, then Get returns old data.** You call `client.Update()`, then immediately `client.Get()`. The Get returns the pre-update version. Your code sees "nothing changed" and returns success. Then the watch event arrives, the object is re-enqueued, and the next reconciliation sees the correct state. This is fine. It's an extra reconciliation, not a bug.

**Pattern 2: Create, then Get returns NotFound.** Same mechanism. You create a child object, then check if it exists. It doesn't -- yet. Your code creates it again and gets an `AlreadyExists` error. Handle that error gracefully (treat it as success) and the system converges. Multigres does [exactly this for Pods and PVCs](https://github.com/multigres/multigres-operator/blob/main/pkg/resource-handler/controller/shard/reconcile_pool_pods.go):

```go
if createErr := r.Create(ctx, desiredPVC); createErr != nil && !errors.IsAlreadyExists(createErr) {
    return fmt.Errorf("failed to create PVC %s: %w", pvcName, createErr)
}
// AlreadyExists is silently accepted -- convergence handles the rest
```

**Pattern 3: Delete, then List still includes the object.** You delete an object, then list all objects and see it still present. The watch event hasn't arrived yet. If your logic depends on the object being gone, add a `DeletionTimestamp` check -- `DeletionTimestamp` is set by the API server before the delete event is even sent.

**Pattern 4: SSA sidesteps the problem.** Multigres avoids most eventual consistency pitfalls by using Server-Side Apply instead of read-modify-write. It builds the desired object in memory and applies it. No read-before-write means no stale-read bugs. The API server itself determines what changed. If the applied state matches current state, it's a no-op. If not, it patches. The cache staleness is irrelevant because the write path never consulted the cache.

The common thread: never assume that a write you just performed is visible in the cache. Design your Reconcile function to be correct regardless of which version of the data it sees. If it sees stale data, the worst case is an extra reconciliation -- which is exactly what level-triggered design gives you for free.
