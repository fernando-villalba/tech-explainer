# Apimachinery: The Type System Under Everything

Every Kubernetes object -- a Pod, a Deployment, your custom `MultigresCluster` -- is a Go struct that embeds two types from apimachinery: `TypeMeta` (what kind of object is this?) and `ObjectMeta` (what is its name, namespace, labels, annotations, ownership, and version?). Every error from the API server -- not found, conflict, forbidden -- is an apimachinery `StatusError` that you check with `errors.IsNotFound()`. Every label selector, every resource quantity, every status condition, every `NamespacedName` your Reconcile function receives -- apimachinery.

This is the lowest layer in the operator stack. client-go builds on it. controller-runtime builds on it. Your CRD types embed it. You use it in every file of your operator without thinking about it. When something goes wrong -- a type isn't recognized, a scheme registration is missing, an error check doesn't match -- you need to understand what's down here.

## GVK and GVR: How Kubernetes Identifies Things

Every resource in Kubernetes is identified by two coordinate systems.

**GroupVersionKind (GVK)** identifies a *type*. Group is the API group (empty string for core types like Pod, `apps` for Deployment, `multigres.com` for CRDs like MultigresCluster). Version is the API version (`v1`, `v1beta1`, `v1alpha1`). Kind is the type name (`Pod`, `Deployment`, `MultigresCluster`). A GVK tells you *what something is*.

```go
schema.GroupVersionKind{
    Group:   "multigres.com",
    Version: "v1alpha1",
    Kind:    "MultigresCluster",
}
```

**GroupVersionResource (GVR)** identifies a *REST endpoint*. Group and version are the same. Resource is the lowercase plural form (`pods`, `deployments`, `multigresclusters`). A GVR tells you *where to find it* on the API server. `GET /apis/multigres.com/v1alpha1/namespaces/default/multigresclusters` -- that URL is the GVR plus namespace.

GVK and GVR are related but not identical. The Kind is `MultigresCluster`. The Resource is `multigresclusters`. The mapping between them (pluralization, aliasing) is handled by the REST mapper. Most of the time you work with GVKs in Go code and GVRs are resolved automatically.

## The Scheme: A Registry of Known Types

The `runtime.Scheme` is the central type registry. It's a bidirectional map between Go types and GVKs. When you write:

```go
utilruntime.Must(multigresv1alpha1.AddToScheme(scheme))
```

in your `main.go`, you're registering every type in your `api/v1alpha1` package with the scheme. The scheme now knows that `*multigresv1alpha1.MultigresCluster` corresponds to `multigres.com/v1alpha1, Kind=MultigresCluster`, and vice versa.

Two maps power this:

```go
type Scheme struct {
    gvkToType map[schema.GroupVersionKind]reflect.Type  // GVK -> Go type
    typeToGVK map[reflect.Type][]schema.GroupVersionKind // Go type -> GVK(s)
    // ...
}
```

When client-go receives JSON from the API server with `"apiVersion": "multigres.com/v1alpha1"` and `"kind": "MultigresCluster"`, it looks up the GVK in the scheme to find the Go type, allocates an instance, and deserializes into it. When controller-runtime's cache needs to start an informer for `&multigresv1alpha1.MultigresCluster{}`, it looks up the Go type in the scheme to find the GVK, then constructs the REST URL from the GVR.

If a type isn't in the scheme, nothing works. You get "no kind is registered for the type" panics. This is why every operator's `init()` function registers both the built-in types (`clientgoscheme.AddToScheme`) and its custom types. Miss one and the client can't serialize or deserialize objects of that type.

The `SchemeBuilder` pattern -- used in every `groupversion_info.go` file kubebuilder generates -- collects registration functions and applies them all at once:

```go
var (
    SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
    AddToScheme   = SchemeBuilder.AddToScheme
)
```

## TypeMeta and ObjectMeta: The Shape of Every Object

Every Kubernetes object embeds two structs.

**TypeMeta** carries the GVK as wire-format fields:

```go
type TypeMeta struct {
    Kind       string `json:"kind,omitempty"`
    APIVersion string `json:"apiVersion,omitempty"`
}
```

`APIVersion` encodes group and version as `"group/version"` (or just `"v1"` for core types). These fields appear in every YAML manifest. In Go code, they're usually empty on objects returned by the client -- the client knows the GVK from context. They're populated during serialization.

**ObjectMeta** carries the identity and metadata:

```go
type ObjectMeta struct {
    Name              string            // unique within namespace
    Namespace         string            // scoping
    UID               types.UID         // unique across time and space
    ResourceVersion   string            // optimistic concurrency token
    Generation        int64             // incremented on spec changes
    Labels            map[string]string // queryable key-value pairs
    Annotations       map[string]string // non-queryable metadata
    OwnerReferences   []OwnerReference  // garbage collection and ownership
    Finalizers        []string          // pre-deletion hooks
    DeletionTimestamp *Time             // set when delete is requested
    // ...
}
```

The fields that matter most for operator development:

`ResourceVersion` is the optimistic concurrency token. Every update must include it. If it doesn't match the server's version, you get a Conflict error. This is how Kubernetes prevents lost updates.

`Generation` increments only when the spec changes, not on status-only updates. `GenerationChangedPredicate` in controller-runtime uses this to skip reconciliation when only the status was updated.

`OwnerReferences` establish the ownership tree. When a parent is deleted, Kubernetes garbage-collects all objects that have an OwnerReference pointing to it. `controllerutil.SetControllerReference()` in controller-runtime sets this. The multigres-operator's entire resource tree -- MultigresCluster owns Cells, Cells own Shards, Shards own Pods -- is wired through OwnerReferences.

`DeletionTimestamp` is set when someone deletes the object but finalizers are blocking actual removal. If your Reconcile function sees a non-nil `DeletionTimestamp`, the object is being deleted.

`Finalizers` are strings that prevent deletion until removed. Add a finalizer to run cleanup logic before an object is garbage-collected. Remove it when cleanup is complete. If a finalizer is never removed, the object gets stuck in `Terminating` state.

## The Error Taxonomy: Every Error Has a Reason

The API server returns structured errors as `StatusError` objects. apimachinery provides predicate functions to classify them:

```go
import apierrors "k8s.io/apimachinery/pkg/api/errors"

if apierrors.IsNotFound(err) {
    // object doesn't exist
}
if apierrors.IsConflict(err) {
    // resourceVersion mismatch -- someone else modified the object
}
if apierrors.IsAlreadyExists(err) {
    // tried to create something that already exists
}
```

The full set of predicates:

| Predicate | HTTP Status | When it happens |
|---|---|---|
| `IsNotFound` | 404 | Object doesn't exist. GET after deletion, or GET for a name that was never created. |
| `IsAlreadyExists` | 409 | Create failed because an object with that name already exists. |
| `IsConflict` | 409 | Update/patch failed because `resourceVersion` doesn't match (optimistic concurrency). |
| `IsInvalid` | 422 | Object failed validation (CRD schema, webhook, or built-in validation). |
| `IsForbidden` | 403 | RBAC denied the request. Missing ClusterRole rules. |
| `IsUnauthorized` | 401 | Authentication failed. Bad token or expired credentials. |
| `IsTimeout` | 504 | API server timed out processing the request. |
| `IsServerTimeout` | 500 | Server asked the client to retry (typically during API server restarts). |
| `IsTooManyRequests` | 429 | Rate limited. Back off and retry. |
| `IsGone` | 410 | ResourceVersion is too old for a watch -- the history has been compacted. Reflectors handle this by re-listing. |
| `IsServiceUnavailable` | 503 | API server is overloaded or shutting down. |

The two you'll use most in operator code are `IsNotFound` (the canonical "object was deleted between event and reconciliation" check) and `IsConflict` (the signal to retry a read-modify-write). The multigres-operator checks `IsNotFound` in every reconciler's initial Get and `IsAlreadyExists` when creating child resources idempotently.

## Status Conditions: Standardized Health Reporting

`metav1.Condition` is the standard way to report status in Kubernetes:

```go
type Condition struct {
    Type               string          // "Available", "Progressing", "Degraded"
    Status             ConditionStatus // "True", "False", "Unknown"
    ObservedGeneration int64           // which spec generation this condition reflects
    LastTransitionTime Time            // when Status last changed
    Reason             string          // machine-readable, UpperCamelCase
    Message            string          // human-readable detail
}
```

Conditions are a list, not a single field. An object can have multiple conditions simultaneously: `Available=True`, `Progressing=False`, `Degraded=False`. Each condition type tracks independently.

`ObservedGeneration` is subtle but important. It records which `metadata.generation` the controller was looking at when it set this condition. If `observedGeneration` is less than `metadata.generation`, the condition is stale -- the spec changed but the controller hasn't reconciled yet. Tools and dashboards use this to distinguish "the controller says it's healthy" from "the controller hasn't processed the latest change yet."

The `api/meta` package provides helpers:

```go
import "k8s.io/apimachinery/pkg/api/meta"

meta.SetStatusCondition(&obj.Status.Conditions, metav1.Condition{
    Type:               "Available",
    Status:             metav1.ConditionTrue,
    Reason:             "AllShardsReady",
    Message:            "All 3 shards are running and registered",
    ObservedGeneration: obj.Generation,
})

if meta.IsStatusConditionTrue(obj.Status.Conditions, "Available") {
    // object is healthy
}
```

The multigres-operator uses conditions on every CRD: `Available`, `Progressing`, `Degraded`, `ReadyForDeletion`. The `ReadyForDeletion` condition is part of its graceful deletion protocol -- the parent checks this condition before deleting a child resource.

## Labels and Selectors: Querying by Metadata

Labels are key-value pairs on ObjectMeta. Selectors match objects by their labels. This is how Kubernetes connects Services to Pods, how Deployments find their ReplicaSets, and how operators filter resources.

apimachinery provides two selector types:

**Equality-based**: `environment=production`, `tier!=frontend`. Simple key-value matching.

**Set-based**: `environment in (production, staging)`, `tier notin (frontend)`, `!partition` (key doesn't exist). More expressive.

```go
import (
    "k8s.io/apimachinery/pkg/labels"
    "k8s.io/apimachinery/pkg/selection"
)

requirement, _ := labels.NewRequirement(
    "multigres.com/cluster",
    selection.Equals,
    []string{"my-cluster"},
)
selector := labels.NewSelector().Add(*requirement)
```

In controller-runtime, you pass selectors to list operations:

```go
podList := &corev1.PodList{}
r.List(ctx, podList, client.MatchingLabelsSelector{Selector: selector})
```

The multigres-operator uses label selectors extensively for cache filtering. Its Manager configures per-type cache rules: Secrets in the operator namespace are unfiltered (cert-manager creates unlabeled secrets), Secrets in all other namespaces are filtered by the operator's label. This reduces memory by only caching objects the operator manages.

## NamespacedName and types.UID

`types.NamespacedName` is the identifier you see everywhere in controller-runtime:

```go
type NamespacedName struct {
    Namespace string
    Name      string
}
```

This is what your Reconcile function receives in the `Request`. It's the key for cache lookups. It's the argument to `client.Get()`. It's simple, but having a dedicated type prevents accidentally mixing names, UIDs, and namespace/name pairs -- a class of bug that's easy to introduce with bare strings.

`types.UID` is the globally unique identifier for an object. Unlike names (which can be reused after deletion), UIDs are unique across time and space. OwnerReferences include the UID to ensure they point to the specific owner instance, not a different object that happens to have the same name.

## Resource Quantities: CPU and Memory

`api/resource.Quantity` represents CPU and memory values:

```go
import "k8s.io/apimachinery/pkg/api/resource"

memory := resource.MustParse("256Mi")  // 256 mebibytes
cpu := resource.MustParse("500m")      // 500 millicores (half a core)
```

These appear in container resource requests and limits. The multigres-operator uses them when building Pod specs for PostgreSQL instances -- setting memory limits based on the shard's configuration, CPU requests based on the workload profile.

Quantities handle the unit parsing (Ki, Mi, Gi for binary; k, M, G for decimal; m for milli) and provide comparison and arithmetic methods. Using them instead of raw integers prevents unit conversion bugs.

## Utilities: wait, intstr, runtime

Three utility packages from apimachinery show up frequently in operators:

**`util/wait`** provides polling and backoff primitives. `wait.PollUntilContextTimeout` polls a condition function at intervals until it returns true or the context expires. `wait.Backoff` configures exponential backoff with jitter. client-go's retry utilities build on this.

**`util/intstr`** provides `IntOrString` -- a type that holds either an integer or a string, used in Kubernetes for port specifications (port 80 or port name "http") and rolling update strategies (25% or 5 pods).

**`util/runtime`** provides `HandleCrash` (panic recovery) and `Must` (panic on error for init-time registration). The `utilruntime.Must(scheme.AddToScheme(...))` pattern in every operator's init function uses this -- if scheme registration fails at startup, panicking is the right behavior because nothing will work.

## The Foundation

apimachinery is the type system that makes Kubernetes extensible. The Scheme maps Go types to API coordinates. TypeMeta and ObjectMeta give every object a standard shape. The error taxonomy turns HTTP status codes into actionable predicates. Conditions standardize health reporting. Labels and selectors connect resources. These aren't implementation details you can ignore. They're the vocabulary every other layer speaks.

Next time you see `IsNotFound`, you'll know it's checking a `StatusReason` inside a `StatusError`. When you set a Condition with `ObservedGeneration`, you'll know why tools can tell whether the controller is caught up. When you get "no kind registered for the type," you'll check the scheme registration. The foundation is visible once you know where to look.
