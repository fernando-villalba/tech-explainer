# Manager Lifecycle: Startup Order Matters More Than You Think

The Manager is [~650 lines of carefully sequenced initialization code](https://github.com/kubernetes-sigs/controller-runtime/blob/main/pkg/manager/internal.go). It looks like boilerplate. It isn't. Every line exists to prevent a specific race condition, deadlock, or startup failure that someone hit in production.

If you understand the startup order, you can debug "my controller isn't starting" in minutes instead of hours. If you understand the shutdown order, you can explain why your Pod keeps getting killed during rolling updates. If you understand runnable groups, you can predict exactly when any component in the system will start and stop.

## What the Manager Owns

The Manager is not just a controller registry. It coordinates the lifecycle of every component in the operator:

- The **Cluster** -- which contains the cache, client, scheme, REST mapper, and API reader
- The **webhook server** -- TLS, cert rotation, admission and conversion endpoints
- **Health probe server** -- `/healthz` and `/readyz` endpoints
- **Metrics server** -- Prometheus endpoint
- **pprof server** -- Go profiling endpoints
- **Leader election** -- Kubernetes Lease-based leader election
- All **controllers** -- your reconciliation logic
- Any **custom runnables** you add via `mgr.Add()`

The Manager doesn't just start these things. It starts them in a specific order, monitors them for failures, and shuts them down in the reverse order.

Here's how the [multigres-operator constructs its manager](https://github.com/multigres/multigres-operator/blob/main/cmd/multigres-operator/main.go), with several non-default options:

```go
mgr, err := ctrl.NewManager(cfg, ctrl.Options{
    Scheme: scheme,
    LeaderElection:                true,
    LeaderElectionID:              "multigres-operator.multigres.com",
    LeaderElectionReleaseOnCancel: true,  // fast failover
    Cache: cache.Options{
        ByObject: map[client.Object]cache.ByObject{
            &corev1.Secret{}:      {Label: managedBySelector},
            &appsv1.StatefulSet{}: {Label: managedBySelector},
            // ... label-filtered cache per type
        },
    },
    WebhookServer: webhook.NewServer(webhook.Options{
        Port:    9443,
        CertDir: certDir,
    }),
})
```

The API server client is configured with `QPS: 50, Burst: 100` -- higher than the defaults (20/30), reflecting that a database operator managing pods, PVCs, and services across multiple shards makes many API calls per reconciliation.

## The Six Runnable Groups

When you call `mgr.Add(runnable)`, the Manager classifies the runnable into one of six groups based on type detection:

| Group | What goes in it | When it starts | Leader-gated? |
|---|---|---|---|
| **HTTPServers** | Health probes, metrics, pprof | Phase 1 | No |
| **Webhooks** | Webhook servers | Phase 2 | No |
| **Caches** | Cluster (informers) | Phase 3 | No |
| **Others** | Runnables with `NeedLeaderElection() == false` | Phase 4 | No |
| **Warmup** | Runnables with a `Warmup()` method | Phase 5 | Partial |
| **LeaderElection** | Controllers and anything defaulting here | Phase 6 | Yes |

The classification is automatic. If your runnable is a `webhook.Server`, it goes to Webhooks. If it implements `hasCache` (has a `GetCache()` method), it goes to Caches. If it implements `LeaderElectionRunnable` with `NeedLeaderElection() == false`, it goes to Others. Everything else -- including all controllers -- goes to LeaderElection by default.

This means a controller you register via `mgr.Add()` won't start until leader election completes. That's intentional. Two replicas reconciling the same objects simultaneously would conflict. Only the leader reconciles.

## The Startup Sequence

When you call `mgr.Start(ctx)`, the following happens in strict order. Each phase must complete before the next begins.

### Phase 1: HTTP Servers

Health probes, metrics, and pprof servers start first. This might seem premature -- why serve health checks before anything is ready? -- but there's a good reason.

Kubernetes checks Pod readiness before routing traffic. If the health probe server isn't up, the kubelet reports the Pod as not ready. If the Pod isn't ready for too long, Kubernetes kills it and starts a new one. You get a crash loop before your operator even starts initializing.

By starting health probes immediately, the Pod reports as alive (even if not yet ready -- the readyz endpoint can report "not yet" while the healthz endpoint reports "alive"). This gives the rest of the startup sequence time to complete without the Pod being terminated.

Multigres runs its [metrics server on port 8443 with TLS and authentication filters](https://github.com/multigres/multigres-operator/blob/main/cmd/multigres-operator/main.go) -- a production-grade setup that prevents unauthorized scraping of sensitive metrics.

### Phase 2: Webhook Servers

Webhook servers start next, before the cache. This ordering prevents the conversion webhook deadlock.

Here's the deadlock in detail. Suppose your CRD has a conversion webhook (v1alpha1 -> v1). The cache starts and sends a LIST request to the API server for your CRD type. The API server sees that the stored version and the requested version differ, so it calls your conversion webhook to convert the objects. But the conversion webhook hasn't started yet. The LIST call blocks. The cache sync blocks. The manager startup blocks. Nothing ever proceeds.

Starting webhooks before caches eliminates this entirely. By the time the cache sends its first LIST, the conversion webhook is already serving.

### Phase 3: Caches

The Cluster starts, which starts the cache, which starts all informers. Each informer sends an initial LIST request (to populate its local store) followed by a WATCH (to stream updates). The manager blocks here until all informers report synced.

If an informer can't sync -- the CRD doesn't exist, RBAC denies the LIST, the API server is unreachable -- the manager blocks until `CacheSyncTimeout` (default 2 minutes). After that, the source reports an error and the controller startup fails.

This is also where you'll see high memory usage during startup for operators that watch many resource types. Each LIST pulls every object of that type into memory. Multigres mitigates this with its label-filtered cache -- during startup, the LIST calls include label selectors, so the API server only sends objects managed by the operator.

### Phase 4: Non-Leader-Election Runnables

Runnables that explicitly opt out of leader election via `NeedLeaderElection() == false` start here. These run on every replica, regardless of leader status. Common examples: custom health check watchers, metrics collectors, or runnables that need to monitor the cluster but not modify it.

Multigres adds its [internal certificate rotation manager](https://github.com/multigres/multigres-operator/blob/main/cmd/multigres-operator/main.go) as a custom runnable. Certificate rotation must happen on every replica (all replicas serve webhooks), so the cert manager returns `NeedLeaderElection() == false`. This ensures webhook certificates are rotated regardless of which replica is the leader.

### Phase 5: Warmup Runnables

Warmup is a special phase for controllers that implement the `warmupRunnable` interface. A warmup-capable controller can start populating its caches and queues before leader election completes. The actual reconciliation doesn't start -- workers aren't launched -- but the event sources and queue begin accumulating work.

This means that when leader election is won, the controller starts with a pre-populated queue and synced caches instead of starting from scratch. In large clusters, this can shave minutes off the time between leader election and the first reconciliation.

### Phase 6: Leader Election

If leader election is configured, the manager runs `leaderElector.Run()`. This blocks until this replica either wins the election or the context is cancelled.

When this replica becomes the leader, three things happen:

1. All LeaderElection runnables start. This includes your controllers. Workers begin dequeuing from the workqueue and calling your Reconcile function.
2. The `elected` channel is closed. Any code waiting on `mgr.Elected()` unblocks.
3. If warmup was enabled, the warmup phase has already populated the queue, so reconciliation begins immediately.

If leader election is not configured (single-replica operators), the manager skips the election and immediately starts all LeaderElection runnables.

## Registering Controllers

After the manager is constructed, controllers are registered via `SetupWithManager`. The [multigres-operator registers five controllers](https://github.com/multigres/multigres-operator/blob/main/cmd/multigres-operator/main.go), each with `MaxConcurrentReconciles: 20`:

```go
if err := (&multigrescluster.MultigresClusterReconciler{
    Client:   mgr.GetClient(),
    Scheme:   mgr.GetScheme(),
    Recorder: mgr.GetEventRecorderFor("multigrescluster-controller"),
}).SetupWithManager(mgr, 20); err != nil { ... }

if err := (&shard.ShardReconciler{
    Client:    mgr.GetClient(),
    Scheme:    mgr.GetScheme(),
    Recorder:  mgr.GetEventRecorderFor("shard-controller"),
    APIReader: mgr.GetAPIReader(),   // uncached reader for external secrets
    RPCClient: rpcClient,            // for topology operations
}).SetupWithManager(mgr, 20); err != nil { ... }
```

`MaxConcurrentReconciles: 20` means 20 goroutines per controller process reconcile requests concurrently. The default is 1. For an operator managing hundreds of shards across multiple clusters, a single worker would serialize all reconciliation and fall behind. 20 workers let the controller reconcile 20 different objects simultaneously.

The tradeoff: higher concurrency means more concurrent API server writes and more concurrent resource consumption. If your reconciliation is CPU-intensive or makes many API calls, too many workers can overwhelm the API server or the operator's own Pod. Start with the default, measure reconciliation latency, and increase if your queue depth grows.

Notice that the `ShardReconciler` receives `mgr.GetAPIReader()` -- the uncached client -- in addition to the standard cached client. Most controllers only need the cached client. The Shard controller needs direct API access for external resources that fall outside the label-filtered cache.

## Leader Election Internals

Controller-runtime uses Kubernetes Lease objects for leader election. The leader holds a lease and renews it periodically. If it fails to renew before the deadline, another replica takes over.

The default timing:

- **Lease duration**: 15 seconds. How long a lease is valid after the last renewal.
- **Renew deadline**: 10 seconds. How long the leader tries to renew before giving up.
- **Retry period**: 2 seconds. How often non-leaders check if the lease is available.

The identity is `hostname_<UUID>`, so each process instance has a unique identity even across restarts.

One important safety measure: the HTTP client used for leader election has a timeout of `max(RenewDeadline/2, 1 second)`. This prevents a hung API server call from consuming the entire renew deadline. Without this timeout, a single slow API call could cause the leader to lose its lease because it couldn't send the renewal in time.

### What Happens When the Leader Loses Its Lease

This is one of the most important behaviors to understand.

When the leader loses its lease, the manager immediately sets the graceful shutdown timeout to zero and exits. Not "tries to drain," not "finishes in-progress reconciles." Exits. Fast.

Why? Because losing the lease means another replica might already be the new leader. If the old leader continues reconciling, you have two processes making concurrent writes to the same objects. The whole point of leader election is to prevent this. So when the lease is lost, the only safe action is to stop immediately.

The in-memory workqueue is lost. Anything queued but not yet processed is abandoned. This sounds dangerous, but it's fine because of level-triggered reconciliation. The new leader starts up, its cache syncs (re-listing all objects), and every object is re-enqueued. Nothing is permanently lost. The system converges.

Multigres enables [`LeaderElectionReleaseOnCancel: true`](https://github.com/multigres/multigres-operator/blob/main/cmd/multigres-operator/main.go). On a clean shutdown (SIGTERM during a rolling update), the outgoing leader voluntarily releases the lease. The next replica acquires leadership immediately instead of waiting for the full lease duration to expire. In practice, this reduces failover time from ~15 seconds (lease duration) to ~2 seconds (retry period). For a database operator, minimizing the window where no controller is running is critical.

## The Shutdown Sequence

Shutdown is the reverse of startup, with one crucial twist: it's subject to a graceful shutdown timeout (default 30 seconds).

When the context passed to `mgr.Start()` is cancelled (typically from a SIGTERM signal), the manager begins the stop procedure:

1. **Warmup and Others** stop in parallel.
2. **LeaderElection runnables** (controllers) stop. Each controller's context is cancelled, workers drain, and the workqueue shuts down.
3. **Caches** stop. Informers close their watch connections.
4. **Webhooks** stop. The TLS server shuts down gracefully (1-minute HTTP server shutdown timeout within the overall graceful shutdown window).
5. **HTTP Servers** stop last.
6. Leader election is cancelled and the lease is optionally released.

The ordering matters. Caches stop after controllers because a controller might be mid-reconciliation when shutdown begins. If the cache stopped first, the in-progress reconciliation would get errors from `client.Get()`. By stopping controllers first (cancelling their contexts and draining workers), the cache is only shut down after all reconciliation has ceased.

Webhooks stop after caches but before HTTP servers. This ensures that if the API server is still sending admission requests during shutdown, the webhook can still respond. The health probe server stops last because Kubernetes uses readiness probes to stop routing traffic -- the probe should report "not ready" as long as possible to let the load balancer drain connections.

If the graceful shutdown timeout expires before all groups have stopped, the manager proceeds with the remaining stops forcefully via context cancellation.

## Health Checks and Readiness

The Manager exposes two health endpoints:

```go
mgr.AddHealthzCheck("ping", healthz.Ping)            // /healthz
mgr.AddReadyzCheck("webhook", mgr.GetWebhookServer().StartedChecker()) // /readyz
```

`/healthz` reports whether the process is alive. Failing this causes the kubelet to restart the Pod.

`/readyz` reports whether the process is ready to serve. Failing this causes the service to stop routing traffic. Common readiness checks:

- `healthz.Ping` -- always healthy, just proves the HTTP server is responding
- `mgr.GetWebhookServer().StartedChecker()` -- healthy once the webhook server is listening
- Custom checks that verify cache sync, external service connectivity, etc.

The health probe server starts in Phase 1, before everything else. But individual checks can report "not ready" until their respective components start. This is the right pattern: the health endpoint itself is always available, but it reports the actual readiness of downstream components.

There's a subtle interaction with readiness and leader election. If your readyz check includes the webhook started checker, and your webhook takes time to start (waiting for certificates), the Pod reports not-ready during that window. If you have a PodDisruptionBudget or a rolling update strategy that waits for readiness, this can slow down deployments. Monitor your readiness probe latency and adjust timeouts accordingly.

## Adding Custom Runnables

Any type that implements `Start(ctx context.Context) error` can be added to the manager:

```go
type myRunnable struct{}

func (r *myRunnable) Start(ctx context.Context) error {
    <-ctx.Done()
    return nil
}

mgr.Add(&myRunnable{})
```

By default, custom runnables go to the LeaderElection group and only run on the leader. To run on all replicas, implement `LeaderElectionRunnable`:

```go
func (r *myRunnable) NeedLeaderElection() bool {
    return false
}
```

Multigres uses this for its [internal certificate manager](https://github.com/multigres/multigres-operator/blob/main/cmd/multigres-operator/main.go), which rotates TLS certificates for the webhook server. Certificate rotation must happen on every replica because every replica serves webhooks. The cert manager implements `NeedLeaderElection() == false` and is added to the manager as a regular runnable -- the framework routes it to the Others group, which starts in Phase 4, unconditionally.

```go
// Simplified: how multigres adds its cert rotation runnable
certManager := cert.NewManager(caKey, caCert, certDir, rotationInterval)
if err := mgr.Add(certManager); err != nil { ... }
```

If your runnable needs to do setup before leader election (pre-populating state, establishing connections), implement the `warmupRunnable` interface with a `Warmup(ctx context.Context) error` method. The warmup phase runs in Phase 5, before leader election completes.

The Manager guarantees that if `Start` returns an error, the entire manager shuts down. This is deliberate -- a failed component shouldn't silently continue while the rest of the system assumes it's healthy. If your runnable can tolerate transient failures, handle retries internally and only return an error for unrecoverable situations.

## Debugging Startup Issues

When the manager hangs during startup, it's almost always one of these:

**Cache sync timeout.** An informer can't LIST a resource type. Check RBAC (does your ServiceAccount have `list` and `watch` permissions?), check CRDs (is the CRD installed?), check network (can the Pod reach the API server?). The manager logs which informers it's waiting for. Look for "cache did not sync" messages. If you're using label-filtered caches (like multigres), verify that your label selectors don't inadvertently exclude the resources your controller needs.

**Conversion webhook deadlock.** You have a CRD with conversion webhooks, but the webhook server isn't starting before the cache. This shouldn't happen with the default manager, but can happen with custom lifecycle management. Check that you're not adding the webhook server to a group that depends on cache sync.

**Health probe kills the Pod.** The startup takes longer than the readiness probe timeout. Increase `initialDelaySeconds` or `failureThreshold` on your readiness probe. The manager starts health probes first specifically to avoid this, but if cache sync takes 3 minutes and your readiness probe fails after 60 seconds, the Pod gets killed.

**Leader election takes too long.** In fresh deployments, the first leader acquisition is fast (no existing lease). But if a previous leader left a lease with 15 seconds remaining, the new leader waits up to `LeaseDuration` before it can acquire. During this window, the manager is running but no controllers are active. This is normal. If it's too slow, reduce `LeaseDuration` -- but be careful, because shorter leases increase the risk of split-brain during API server partitions. `LeaderElectionReleaseOnCancel` (as multigres uses) eliminates this wait during clean rollouts by releasing the lease before the Pod terminates.

**Certificate bootstrap fails.** If you manage your own webhook certificates (like multigres does internally), the bootstrap must complete before the webhook server starts. If the CA can't be generated or the cert files can't be written, the webhook server starts with no certificates and every TLS handshake fails. Ensure your cert bootstrap runs before `mgr.Start()`, not inside a runnable.
