# E2E Framework: Testing Operators in Real Clusters

envtest runs a real etcd and kube-apiserver on your laptop. It's fast -- seconds to start. Controllers reconcile, webhooks fire, status gets updated. But there are no nodes. No kubelet. No scheduler. No networking. A Deployment's pods are never scheduled. A Service never resolves. A PodDisruptionBudget is never enforced.

For many operators, this is fine. The [envtest explainer](../controller-runtime/04-testing-with-envtest.md) covers how to test controller logic without a full cluster. But some bugs only appear when the operator runs as an actual container in a real cluster: image pull failures, RBAC misconfiguration in the deployed manifest, webhook TLS certificate issues, scheduling constraints, resource limits, DNS resolution between pods.

[e2e-framework](https://github.com/kubernetes-sigs/e2e-framework) fills that gap. It spins up real [Kind](https://kind.sigs.k8s.io/) clusters (or k3d, or kwok) with actual nodes and kubelets, deploys the operator as a container image, and runs black-box tests against it through the Kubernetes API. The tests use `go test`. No custom binary. No special runner. Standard Go testing with real clusters underneath.

## The Testing Spectrum

Before diving into e2e-framework, it helps to see where it fits. Operator testing has three layers:

**Unit tests** -- pure Go. No Kubernetes at all. Test builder functions, validation logic, helper utilities, template rendering. Fast. Run everywhere.

**Integration tests (envtest)** -- real etcd + kube-apiserver, no nodes. Test controller reconciliation logic: does the Reconcile function create the right child resources? Does the webhook reject invalid objects? Does the status get updated correctly? Fast enough for CI on every push.

**E2e tests (e2e-framework)** -- real cluster with nodes. Test the full deployment: the operator container starts, CRDs are installed, webhooks are registered with valid TLS, pods are scheduled and run, services resolve, the whole system converges. Slower (minutes per test suite) but catches an entire class of bugs the other layers can't.

Each layer catches different failures. A test that passes in envtest can fail in e2e because the Makefile forgot to include a CRD in the Kustomize overlay. A test that passes in e2e can fail in production because of network policies that don't exist in Kind. No single layer is sufficient. The question is always which layer is the cheapest place to catch a specific bug.

## The Architecture ([pkg/env](https://github.com/kubernetes-sigs/e2e-framework/tree/main/pkg/env))

e2e-framework has four core concepts: Environment, EnvFuncs, Features, and the klient.

### Environment

The Environment is the test harness. It's created in `TestMain` and manages the lifecycle of the test suite:

```go
var testenv env.Environment

func TestMain(m *testing.M) {
    testenv = env.New()
    kindClusterName := envconf.RandomName("my-cluster", 16)
    namespace := envconf.RandomName("myns", 16)

    testenv.Setup(
        envfuncs.CreateCluster(kind.NewProvider(), kindClusterName),
        envfuncs.CreateNamespace(namespace),
    )

    testenv.Finish(
        envfuncs.DeleteNamespace(namespace),
        envfuncs.DestroyCluster(kindClusterName),
    )

    os.Exit(testenv.Run(m))
}
```

`Setup()` runs before any tests in the package. `Finish()` runs after all tests complete (even on failure). `Run(m)` executes the test suite and returns the exit code. Each setup and finish function receives a `context.Context` and `*envconf.Config`, and returns them -- functions are chained.

### EnvFuncs ([pkg/envfuncs](https://github.com/kubernetes-sigs/e2e-framework/tree/main/pkg/envfuncs))

Pre-built functions for common setup and teardown operations:

- `envfuncs.CreateCluster(provider, name)` -- creates a Kind/k3d/kwok cluster
- `envfuncs.DestroyCluster(name)` -- tears it down
- `envfuncs.CreateNamespace(name)` -- creates a namespace
- `envfuncs.DeleteNamespace(name)` -- deletes it
- `envfuncs.SetupCRDs(path, pattern)` -- applies CRD YAML files to the cluster
- `envfuncs.TeardownCRDs(path, pattern)` -- removes them

These compose into the setup chain. Create cluster, install CRDs, create namespace -- each step builds on the previous one. If any step fails, the test suite aborts.

Custom setup functions follow the same signature. Need to deploy the operator via Helm? Load a container image into Kind? Apply a Kustomize overlay? Write a function that takes `(context.Context, *envconf.Config)` and returns `(context.Context, error)`.

### Features ([pkg/features](https://github.com/kubernetes-sigs/e2e-framework/tree/main/pkg/features))

Features are the actual tests. Each Feature has a name, optional setup/teardown, and one or more assessments:

```go
func TestClusterCreation(t *testing.T) {
    f := features.New("cluster-creates-pods").
        Setup(func(ctx context.Context, t *testing.T, cfg *envconf.Config) context.Context {
            // create test resources
            return ctx
        }).
        Assess("pods are scheduled", func(ctx context.Context, t *testing.T, cfg *envconf.Config) context.Context {
            // verify pods exist and are running
            return ctx
        }).
        Assess("services resolve", func(ctx context.Context, t *testing.T, cfg *envconf.Config) context.Context {
            // verify service DNS works
            return ctx
        }).
        Teardown(func(ctx context.Context, t *testing.T, cfg *envconf.Config) context.Context {
            // clean up test resources
            return ctx
        }).
        Feature()

    testenv.Test(t, f)
}
```

Features integrate with Go's testing. Each assessment is a sub-step within the feature. If an assessment calls `t.Fatal()`, subsequent assessments in that feature are skipped but teardown still runs. Labels on features (`WithLabel("type", "smoke")`) enable filtering with `--labels` flags.

### Klient and Wait Conditions ([klient](https://github.com/kubernetes-sigs/e2e-framework/tree/main/klient))

The klient package wraps client-go into a test-friendly API. The most important part is `klient/wait/conditions`, which provides polling helpers for common resource states:

```go
err := wait.For(
    conditions.New(cfg.Client().Resources()).ResourceScaled(
        deployment, func(obj k8s.Object) int32 {
            return obj.(*appsv1.Deployment).Status.ReadyReplicas
        }, 3,
    ),
    wait.WithTimeout(2 * time.Minute),
)
```

`ResourceScaled` polls until the resource reaches the expected replica count. Other conditions include `ResourceDeleted`, `ResourceMatch`, and `JobCompleted`. These replace raw `time.Sleep` calls with event-driven polling.

## Cluster Providers ([support](https://github.com/kubernetes-sigs/e2e-framework/tree/main/support))

e2e-framework supports multiple cluster backends:

**Kind** (`support/kind`) -- the default. Creates real clusters using Docker containers as nodes. Full Kubernetes including kubelet, scheduler, networking. The multigres-operator uses Kind exclusively.

**k3d** (`support/k3d`) -- lightweight Kubernetes via k3s in Docker. Faster cluster creation than Kind, but slightly different from upstream Kubernetes.

**kwok** (`support/kwok`) -- Kubernetes Without Kubelet. Simulates nodes and pods without actually running containers. Extremely fast but only useful for testing control plane behavior, not workload behavior.

The provider is passed to `envfuncs.CreateCluster()`. Switching providers is a one-line change. The rest of the test code is provider-agnostic.

## Third-Party Integrations ([third_party](https://github.com/kubernetes-sigs/e2e-framework/tree/main/third_party))

e2e-framework includes integrations for deploying software into test clusters:

- **Helm** -- install charts into the cluster during setup
- **Flux** -- deploy via GitOps
- **ko** -- build and deploy Go containers directly (no Dockerfile needed)
- **vcluster** -- create virtual clusters inside the test cluster

These are optional. Many operators deploy via raw manifests or Kustomize in custom setup functions.

## Real-World Patterns from the Multigres Operator

The [multigres-operator](https://github.com/multigres/multigres-operator) has a production-grade e2e test suite that demonstrates several patterns worth understanding.

### Shared vs Dedicated Clusters

Not every test needs its own cluster. Cluster creation takes 1-2 minutes. If you have 10 test packages, that's 10-20 minutes just for setup.

Multigres solves this with two modes:

**Shared cluster** -- one Kind cluster reused across test packages. Each test package gets its own namespace for isolation. The cluster is created once by the first package that needs it and reused by all subsequent packages. Fast. Good for tests that don't need full cluster isolation.

```go
func TestMain(m *testing.M) {
    cluster, err = framework.EnsureSharedCluster()  // idempotent
    if err != nil {
        fmt.Fprintf(os.Stderr, "e2e setup: %v\n", err)
        os.Exit(1)
    }
    os.Exit(m.Run())
}
```

**Dedicated cluster** -- one Kind cluster per test package. Full isolation. Slower but necessary for tests that modify cluster-wide state (CRD changes, webhook configuration, RBAC modifications).

```go
func TestMain(m *testing.M) {
    cluster, err = framework.NewDedicatedCluster("deletion-tests")
    // ...
}
```

The shared cluster handles most tests: webhook rejection, basic creation, scaling, template resolution. Dedicated clusters handle destructive tests: deletion protocols, CRD upgrades, anything that leaves cluster-wide side effects.

### Build Tag Separation

All e2e test files use `//go:build e2e`. This means `go test ./...` skips them entirely. E2e tests only run when explicitly requested:

```bash
go test -tags e2e ./test/e2e/...
```

This keeps the regular test suite fast. CI runs unit and integration tests on every push, e2e tests on merges or on a schedule.

### Cluster Cleanup Control

Failed e2e tests leave behind Kind clusters with valuable debugging state. Multigres uses an environment variable to control cleanup:

```
E2E_KEEP_CLUSTERS=never       # default: always destroy
E2E_KEEP_CLUSTERS=on-failure  # keep clusters from failed tests
E2E_KEEP_CLUSTERS=always      # never destroy (for local debugging)
```

When a cluster is kept, the test logs the cluster name and the `kind delete cluster --name <name>` command to clean up manually. This is critical for debugging -- `kubectl` into the surviving cluster, inspect pod logs, check events, examine the operator's state.

### Image Loading

The Makefile builds the operator container image and loads it into Kind *before* `go test` runs. The Go test code does not build images. This separation keeps the test code focused on assertions and avoids flaky Docker-in-Docker builds during tests.

```makefile
e2e: docker-build
    kind load docker-image $(IMG) --name $(KIND_CLUSTER)
    go test -tags e2e -v ./test/e2e/...
```

### Fixture Loading

Test resources are defined as YAML fixtures, not constructed in Go code:

```go
cr := framework.MustLoadCluster("test/e2e/fixtures/base.yaml", ns)
```

This keeps test definitions declarative and readable. The fixture is a complete MultigresCluster CR. Tests modify specific fields before applying -- adding a cell, changing a pool count, removing a shard -- and assert on the resulting cluster state.

### Wait Helpers

Instead of `time.Sleep`, the framework uses event-driven polling:

```go
cluster.WaitForAllPodsReady(t, ns)
```

This polls the cluster until all pods in the namespace are in `Ready` state or a timeout expires. No sleeping for arbitrary durations. Tests complete as fast as the system converges.

## envtest vs e2e-framework: When to Use Which

| Question | envtest | e2e-framework |
|---|---|---|
| Does the Reconcile function create the right resources? | Yes | Overkill |
| Does the webhook reject invalid objects? | Yes (programmatic) | Yes (CEL + programmatic + TLS) |
| Do pods actually schedule and run? | No | Yes |
| Does the operator's Deployment start correctly? | No | Yes |
| Does service DNS resolve between pods? | No | Yes |
| Are the Kustomize manifests correct? | No | Yes |
| Is the container image buildable and runnable? | No | Yes |
| Is RBAC sufficient for the deployed ServiceAccount? | No | Yes |
| How fast does the test run? | Seconds | Minutes |
| Can it run without Docker? | Yes | No |

The rule of thumb: test controller logic in envtest, test deployment correctness in e2e. Most operators should have both. The multigres-operator runs envtest on every push and e2e on merges.

## The Testing Triangle

Unit tests at the base -- fast, many, testing pure logic. Integration tests (envtest) in the middle -- testing controller behavior against a real API server. E2e tests at the top -- slow, few, testing the full deployment end to end.

Each layer exists because it catches bugs the layer below misses. A controller that creates perfect resources in envtest can fail in production because the RBAC manifest is wrong, the container image doesn't start, or the webhook certificate wasn't provisioned. E2e tests catch those failures before users do.
