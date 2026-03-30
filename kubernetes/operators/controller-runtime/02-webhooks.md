# Webhooks: The Other Half of Controller-Runtime Nobody Explains

Controllers reconcile. Webhooks intercept. Together they form the complete operator pattern -- controllers converge the world toward desired state, while webhooks enforce constraints at the gate before objects ever reach etcd.

Most controller-runtime documentation treats webhooks as an appendix. They're not. They run on a fundamentally different model than controllers, have different failure modes, and require different mental models to debug. A controller that panics retries later. A webhook that fails blocks the entire API server from accepting that resource type.

## Two Kinds of Webhooks

Kubernetes admission webhooks come in two flavors, and controller-runtime implements both.

**Mutating webhooks** intercept API requests and modify objects before they're persisted. Your defaulter receives the incoming object, changes fields (set defaults, inject sidecars, add labels), and returns JSON patches describing the diff. The API server applies the patches to the original request.

**Validating webhooks** intercept API requests and accept or reject them. Your validator receives the object and returns allowed or denied. A denied response blocks the API call with an error message the user sees in their `kubectl` output.

There's also a third kind -- **conversion webhooks** -- for CRDs with multiple API versions. These convert objects between versions on the fly. We'll get to those.

## The Webhook Builder

Controller-runtime exposes webhooks through a builder pattern that mirrors the controller builder:

```go
ctrl.NewWebhookManagedBy(mgr, &myv1.MyResource{}).
    WithDefaulter(&myDefaulter{}).
    WithValidator(&myValidator{}).
    Complete()
```

This registers two webhooks at conventional paths derived from the GVK:

- `/mutate-mygroup-example-com-v1-myresource` for the defaulter
- `/validate-mygroup-example-com-v1-myresource` for the validator

The path convention matters because your `MutatingWebhookConfiguration` and `ValidatingWebhookConfiguration` resources in the cluster must reference these exact paths. The builder generates them deterministically: `/mutate-` or `/validate-` followed by the group (dots replaced with dashes), version, and lowercase kind.

## The Defaulter Interface

```go
type Defaulter[T runtime.Object] interface {
    Default(ctx context.Context, obj T) error
}
```

One method. You receive the object, mutate it in place, return nil or an error. Controller-runtime handles everything else: decoding the admission request, deep-copying the original, calling your function, computing the JSON patches between original and mutated, and encoding the response.

The JSON patch computation is the clever part. You never think about patches. You just modify the Go struct and the framework diffs it.

One subtlety: if your defaulter returns the object unchanged, no patches are generated and the response is a simple "allowed." This short-circuit avoids sending empty patch arrays back to the API server.

DELETE operations skip the defaulter entirely. There's no object to default when it's being removed.

## The Validator Interface

```go
type Validator[T runtime.Object] interface {
    ValidateCreate(ctx context.Context, obj T) (admission.Warnings, error)
    ValidateUpdate(ctx context.Context, oldObj, newObj T) (admission.Warnings, error)
    ValidateDelete(ctx context.Context, obj T) (admission.Warnings, error)
}
```

Three methods, one per operation type. The framework dispatches based on the admission request's `Operation` field. Each method returns warnings (displayed to the user but don't block the request) and an error (blocks the request if non-nil).

For Update validation, you get both the old and new objects. This lets you enforce immutability constraints: "this field cannot change after creation." You compare `oldObj.Spec.Region` to `newObj.Spec.Region` and reject if they differ.

The error handling is worth understanding. If your error implements `apierrors.APIStatus` (like errors from `apierrors.NewForbidden()` or `apierrors.NewInvalid()`), the framework uses its status code and message directly. Otherwise it wraps your error message in a 403 Forbidden response. Use `apierrors.NewInvalid()` with field errors for structured validation messages that kubectl renders cleanly.

## The Request Flow

When the API server sends an admission request to your webhook, here's every step that happens:

1. The TLS handshake completes. The `CertWatcher` provides the current certificate, which may have been rotated since the server started.

2. The HTTP mux routes the request to the instrumented webhook handler. Prometheus metrics record the in-flight count, latency, and total requests.

3. The body is read with a 7MB limit. This matches the Kubernetes API server's maximum request size. If the body exceeds this, the webhook returns 400.

4. Content-Type is verified as `application/json`. If it's anything else, 400.

5. The `AdmissionReview` is deserialized. Controller-runtime handles both `v1` and `v1beta1` versions transparently -- it detects which version was sent and responds with the same version.

6. Panic recovery wraps the handler call. If your defaulter or validator panics, the panic is caught, a 500 error is returned, and a panic counter metric is incremented. The process doesn't crash.

7. Your handler runs.

8. The response is finalized: the UID from the request is copied to the response, the status code defaults to 200, and any JSON patches are serialized.

9. The response is written back as an `AdmissionReview` JSON body.

The entire round trip -- TLS, deserialization, your logic, serialization -- must complete before the API server's webhook timeout expires (default 10 seconds, configurable per webhook). If your handler takes too long, the API server treats it as a failure and applies the webhook's `failurePolicy` (Fail or Ignore).

## Why Webhooks Run on All Replicas

Controllers run behind leader election. Only one replica reconciles at a time. Webhooks are the opposite. They run on every replica.

The reason is architectural. The Kubernetes API server distributes admission webhook calls across all endpoints listed in the webhook configuration's Service. If you have three replicas and only the leader runs the webhook server, two-thirds of admission requests fail with connection refused. The API server doesn't know or care about your leader election.

This is why `webhook.Server.NeedLeaderElection()` returns `false`. The manager starts the webhook server unconditionally, regardless of whether this replica is the leader. Controllers wait for election. Webhooks don't.

This has implications for your webhook logic. If your defaulter or validator reads from the cache, the cache must also be running on non-leader replicas. It is -- the cache also starts unconditionally, before leader election. But your controller's reconciliation logic only runs on the leader. So a non-leader replica has a populated cache (it's watching the same resources) but no active reconciliation loop.

## The Startup Order Problem

Webhook servers must start before the cache. This sounds backward -- why would the webhook need to be ready before the cache syncs?

The answer is conversion webhooks. If your CRD has multiple versions (say v1alpha1 and v1), and you've defined a conversion webhook, the API server calls that webhook whenever it needs to convert between versions. This includes the LIST calls that populate the cache.

Picture the deadlock: the cache starts, sends a LIST request to the API server, the API server needs to convert the response objects via your conversion webhook, the conversion webhook hasn't started yet, the LIST hangs, the cache never syncs, the manager never starts.

Controller-runtime prevents this by starting the webhook server in phase 2 (after health probes but before caches). The cache starts in phase 3, by which time the webhook is ready to serve conversion requests.

## Certificate Management

Webhooks require TLS. The Kubernetes API server only communicates with webhooks over HTTPS, and it verifies the server certificate against the CA bundle in the webhook configuration.

Controller-runtime's `DefaultServer` handles TLS setup with two mechanisms for certificate management:

The `CertWatcher` monitors certificate files on disk using a dual strategy: filesystem notifications (fsnotify) for immediate detection, plus polling every 10 seconds as a fallback. When you mount certificates from a Kubernetes Secret (managed by cert-manager, for example), the Secret update triggers a file change, the CertWatcher detects it, re-reads the cert and key, and the next TLS handshake uses the new certificate. No restart required.

The default cert directory is `/tmp/k8s-webhook-server/serving-certs/`, looking for `tls.crt` and `tls.key`. In production, cert-manager's `Certificate` resource writes to this location when configured correctly.

If you supply your own `tls.Config.GetCertificate` function in the server options, the `CertWatcher` is skipped entirely. This is the escape hatch for custom certificate management -- PKCS#11 hardware tokens, Vault integration, or any other scheme that doesn't use files.

## Conversion Webhooks

Conversion webhooks solve a specific problem: you released v1alpha1 of your CRD, users created objects, and now you're shipping v1 with a different schema. The API server needs to convert between versions on the fly.

Controller-runtime supports two patterns for conversion:

**Hub-and-Spoke.** One version is the "hub" (the canonical internal representation), and all other versions are "spokes." Each spoke knows how to convert to and from the hub. Converting between two spokes goes through the hub.

```go
// v1 is the hub
func (v1 *MyResourceV1) Hub() {}

// v1alpha1 is a spoke
func (v1a1 *MyResourceV1Alpha1) ConvertTo(hub conversion.Hub) error { ... }
func (v1a1 *MyResourceV1Alpha1) ConvertFrom(hub conversion.Hub) error { ... }
```

**Explicit converters.** Register a converter function in the manager's converter registry. This gives you full control over the conversion logic without implementing interfaces on your types.

All conversion requests go to a single path: `/convert`. The conversion handler decodes the `ConversionReview`, iterates over the objects, converts each one, and returns the results. Unlike admission webhooks, conversion webhooks don't modify objects in-flight -- they transform them between equivalent representations.

## When Webhooks Break

**Your webhook panics.** The panic is caught. A 500 Internal Server Error is returned. The API server applies the `failurePolicy`. If it's `Fail` (the default for most production setups), the API request is rejected. If it's `Ignore`, the request proceeds without your webhook's mutations or validation. Choose your failure policy carefully.

**Your webhook is slow.** The API server has a timeout per webhook (default 10 seconds). If your handler exceeds it, the request is treated as a failure. This blocks the user's kubectl command. Unlike controller reconciliation, where slowness just means delayed convergence, webhook slowness directly degrades the API server experience for everyone.

**Your cert expires.** TLS handshakes fail. Every admission request to your webhook returns an error. The API server may reject all creates/updates for your resource type, depending on the `failurePolicy`. The `CertWatcher` polls every 10 seconds, so if cert-manager renews the cert, the new one is picked up quickly -- but only if the cert files are updated *before* the old cert expires.

**Your webhook rejects a valid object.** Unlike a controller bug (which just delays convergence), a webhook bug blocks users from creating or modifying resources. There is no retry, no backoff, no eventual consistency. The request fails and the user sees an error. Test your validation logic thoroughly -- a false rejection in production is much harder to recover from than a controller bug.

**Your conversion webhook is down during cache sync.** This is the deadlock scenario described above. The cache sends a LIST, the API server tries to convert, the conversion webhook is unreachable, and the entire startup sequence hangs. Controller-runtime's startup ordering prevents this, but if you're running conversion webhooks outside the standard manager lifecycle, you need to handle this yourself.

## Admission vs. Reconciliation: Different Guarantees

The fundamental difference: webhooks are synchronous and blocking; controllers are asynchronous and convergent.

A webhook must respond immediately. It can't say "I'll get back to you." It can't retry. It can't process things out of order. The API server is waiting, the user is waiting, and the timeout is ticking.

A controller operates on its own schedule. It reads current state, makes progress toward desired state, and if it fails, it tries again later. The workqueue buffers. The rate limiter backs off. The system converges eventually.

Use webhooks for things that must be enforced at admission time: defaults that other systems depend on, validation rules that can't be fixed after the fact, conversion between API versions. Use controllers for everything else.

If you find yourself putting complex business logic in a webhook, pause. Ask whether the same constraint could be enforced by a controller that watches objects after creation and either fixes them or marks them as invalid. Controllers are more resilient, more testable, and more debuggable than webhooks. Webhooks are for gates. Controllers are for convergence.
