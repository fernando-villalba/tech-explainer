# Testing with envtest: A Real Control Plane, No Cluster Required

Most people test Kubernetes controllers one of two ways. They mock the Kubernetes client and test against fake objects. Or they deploy to a real cluster and run end-to-end tests. Both approaches have serious problems.

Mocks lie. You can make a mock return anything, which means your tests pass even when your code has bugs that only manifest against the real API server -- admission logic, field validation, resource version conflicts, defaulting by the API server, list pagination, field selectors. A test that passes against a mock and fails against the real API server is worse than no test, because it gave you false confidence.

Real clusters are slow, expensive, and flaky. You need infrastructure, you need cleanup, you need to handle parallel test runs, and you need to wait for Pods to schedule (if you're testing anything that creates Pods). Integration test suites against real clusters routinely take 20+ minutes.

envtest is the third option. It starts a real etcd and a real kube-apiserver on your local machine, gives you a `rest.Config` that points at them, and lets you run your controller against actual Kubernetes API behavior. No mocks. No cluster. Tests run in seconds.

## What You Get

envtest gives you a real Kubernetes control plane, minus everything that runs on nodes. Concretely:

**Running (as local processes):**
- **etcd** -- the real key-value store, writing to a temporary directory
- **kube-apiserver** -- the real API server, with full RBAC, admission, CRD support, discovery, conversion webhooks, and field validation

**Not running:**
- **kubelet** -- no nodes exist, so Pods won't be scheduled or run
- **kube-scheduler** -- nothing assigns Pods to nodes
- **kube-controller-manager** -- no built-in controllers: no garbage collection, no ReplicaSet scaling, no namespace lifecycle, no service account creation
- **kube-proxy, CoreDNS, CNI** -- no networking

This absence list is important. If your controller creates a Deployment, the Deployment object exists in etcd, but no ReplicaSet is created (the Deployment controller isn't running). If your controller creates a Pod, the Pod object exists, but it stays in `Pending` forever (no scheduler). If your controller sets an OwnerReference, the child is not automatically deleted when the parent is deleted (no garbage collector).

You're testing your controller's interaction with the Kubernetes API. Not the full Kubernetes lifecycle. This is the right level for unit and integration tests of operator logic.

## Setup

```go
import (
    "testing"
    "sigs.k8s.io/controller-runtime/pkg/envtest"
)

var testEnv *envtest.Environment

func TestMain(m *testing.M) {
    testEnv = &envtest.Environment{
        CRDDirectoryPaths: []string{"path/to/crds"},
    }

    cfg, err := testEnv.Start()
    if err != nil {
        panic(err)
    }

    // cfg is a *rest.Config pointing at the local API server
    // Use it to create clients, managers, etc.

    code := m.Run()

    testEnv.Stop()
    os.Exit(code)
}
```

`testEnv.Start()` does the following:

1. Locates the `kube-apiserver` and `etcd` binaries. It checks, in order: `TEST_ASSET_KUBE_APISERVER` and `TEST_ASSET_ETCD` environment variables, the `KUBEBUILDER_ASSETS` directory, or the default install location (`~/Library/Application Support/io.kubebuilder.envtest/k8s` on macOS, `~/.local/share/kubebuilder-envtest/k8s` on Linux).

2. Starts etcd on a random port. Uses `--unsafe-no-fsync=true` on etcd 3.5+ for faster writes (safe because test data is ephemeral).

3. Generates self-signed CA and serving certificates for the API server.

4. Starts kube-apiserver on a random port, configured with RBAC authorization, the generated certs, and the etcd connection. The ServiceAccount admission plugin is disabled because the SA controller (part of kube-controller-manager) isn't running.

5. Waits for both processes to be healthy by polling their health endpoints every 100ms.

6. Creates an admin user in the `system:masters` group with QPS=1000 and Burst=2000 (high throughput for test speed).

7. Waits for the `default` namespace to appear in etcd.

8. Installs CRDs from the specified paths.

9. Returns a `*rest.Config`.

If you set `DownloadBinaryAssets: true`, envtest will automatically download the correct etcd and kube-apiserver binaries for your platform. It fetches an index from the controller-tools repository, selects the latest stable version, downloads a tarball, verifies the SHA-512 checksum, and caches the binaries locally. Subsequent runs skip the download.

The `setup-envtest` CLI tool is the standalone way to manage these binaries:

```bash
go install sigs.k8s.io/controller-runtime/tools/setup-envtest@latest
eval $(setup-envtest use -p env)
```

This downloads the binaries and prints the environment variables to set. Most CI systems use this approach.

## Installing CRDs

envtest reads CRD manifests from directories and installs them before returning the config:

```go
testEnv = &envtest.Environment{
    CRDDirectoryPaths: []string{
        filepath.Join("..", "config", "crd", "bases"),
    },
    ErrorIfCRDPathMissing: true,
}
```

It walks the directories, reads `.yaml`, `.json`, and `.yml` files, parses multi-document YAML, and extracts `CustomResourceDefinition` objects. Duplicates are de-duplicated by name.

`ErrorIfCRDPathMissing: true` makes `Start()` fail if a specified CRD directory doesn't exist. Without this, a typo in the path silently installs nothing, and your tests fail later with confusing "resource not found" errors.

You can also pass CRDs directly:

```go
testEnv = &envtest.Environment{
    CRDs: []*apiextensionsv1.CustomResourceDefinition{myCRD},
}
```

After installation, envtest polls the API server's discovery endpoint until all CRD resources appear. This ensures the CRDs are fully available before your tests start creating custom resources. The polling creates a fresh clientset each time to avoid discovery caching issues.

If your CRDs use conversion webhooks, envtest automatically patches the webhook configuration to point at your local test webhook server.

## Testing Webhooks

Testing admission webhooks requires a local TLS server. envtest sets this up:

```go
testEnv = &envtest.Environment{
    CRDDirectoryPaths: []string{"path/to/crds"},
    WebhookInstallOptions: envtest.WebhookInstallOptions{
        Paths: []string{"path/to/webhook-manifests"},
    },
}
```

When webhooks are configured, `Start()` does extra work:

1. Generates a test CA and serving certificate for `localhost`.
2. Writes the cert and key to a temporary directory.
3. Reads your `MutatingWebhookConfiguration` and `ValidatingWebhookConfiguration` YAML manifests.
4. Patches them: replaces `Service` references with direct `URL` references pointing to `https://localhost:<random-port>/<path>`, and injects the test CA bundle.
5. Creates or updates the webhook configurations in the API server.
6. Polls until all webhook configurations are visible via the API.

Your test then starts a Manager that points its webhook server at the same host, port, and cert directory:

```go
mgr, _ := ctrl.NewManager(cfg, ctrl.Options{
    WebhookServer: webhook.NewServer(webhook.Options{
        Host:    testEnv.WebhookInstallOptions.LocalServingHost,
        Port:    testEnv.WebhookInstallOptions.LocalServingPort,
        CertDir: testEnv.WebhookInstallOptions.LocalServingCertDir,
    }),
})

// Register your webhooks
ctrl.NewWebhookManagedBy(mgr, &myv1.MyResource{}).
    WithDefaulter(&myDefaulter{}).
    WithValidator(&myValidator{}).
    Complete()

// Start the manager in a goroutine
go mgr.Start(ctx)
```

Now when your test creates a `MyResource` via the API server, the API server sends the admission request to your local webhook server, and your defaulter/validator runs. You're testing the full admission flow -- serialization, deserialization, patch computation, error handling -- not a mock.

## What envtest Can't Test

Because there's no kubelet, scheduler, or controller manager, several Kubernetes behaviors don't happen:

**Garbage collection doesn't work.** If you create an object with an OwnerReference, deleting the owner does not delete the child. Your controller probably sets OwnerReferences and expects GC to clean up child objects. In envtest, you need to test cleanup logic explicitly -- either by verifying that OwnerReferences are set correctly (and trusting that GC works in production), or by running finalizer-based cleanup in your controller.

**Pods don't run.** You can create Pod objects, but they stay in `Pending`. If your controller checks Pod status (Running, Succeeded, Failed), you'll need to manually patch Pod status in your tests to simulate the kubelet.

**Deployments don't create ReplicaSets.** Statefulsets don't create Pods. Jobs don't create Pods. None of the built-in controllers run. If your controller creates a Deployment and then waits for it to become Available, that condition is never set. Mock the status or test at the correct level of abstraction.

**Default service accounts aren't created.** The ServiceAccount admission plugin is disabled. If your code assumes a `default` service account exists in each namespace, that assumption doesn't hold in envtest.

This isn't a limitation -- it's a feature. envtest tests your controller's logic, not Kubernetes internals. You're verifying that your code makes the correct API calls with the correct objects. Whether Kubernetes correctly handles those objects downstream is Kubernetes's problem, tested by Kubernetes's own test suite.

## The komega Library

envtest ships with `komega`, a test assertion library designed for Kubernetes objects. It wraps Gomega matchers with Kubernetes-aware helpers:

```go
import "sigs.k8s.io/controller-runtime/pkg/envtest/komega"

k := komega.New(client)

// Wait for a resource to appear
Eventually(k.Get(&myResource)).Should(Succeed())

// Wait for a specific condition
Eventually(k.Object(&myResource)).Should(HaveField("Status.Ready", BeTrue()))

// Fetch-modify-update in one step
Eventually(k.Update(&myResource, func() {
    myResource.Spec.Replicas = 3
})).Should(Succeed())
```

The `Eventually` pattern is essential for testing controllers. Your Reconcile function runs asynchronously. When your test creates an object, the controller hasn't reconciled it yet. `Eventually` polls until the assertion passes or a timeout expires.

`EqualObject` is a matcher that compares Kubernetes objects while ignoring auto-generated metadata:

```go
Expect(actual).To(komega.EqualObject(expected,
    komega.IgnoreAutogeneratedMetadata,
))
```

This ignores `uid`, `resourceVersion`, `creationTimestamp`, `managedFields`, and other fields that the API server sets automatically. Without this, you'd need to copy those fields from the actual object to the expected object before comparing, which defeats the purpose of the assertion.

## Port Allocation

envtest needs random available ports for etcd and the API server. It uses a file-based port reservation system to prevent collisions between parallel test runs:

1. Binds to port 0 to let the OS assign a free port.
2. Writes a reservation file to `~/.cache/kubebuilder-envtest/` with a 2-minute TTL.
3. On the next allocation, checks existing reservation files and avoids those ports.

This means you can run `go test ./...` with parallel packages and each test suite gets its own etcd and API server on different ports. The reservation files are cleaned up automatically when they expire.

## Using an Existing Cluster

For tests that need the full Kubernetes lifecycle (garbage collection, scheduling, etc.), envtest can point at a real cluster:

```go
testEnv = &envtest.Environment{
    UseExistingCluster: ptr.To(true),
}
```

Or set `USE_EXISTING_CLUSTER=true` in the environment. Instead of starting local processes, envtest loads your kubeconfig and returns a `rest.Config` pointing at the real cluster. CRDs are still installed (and optionally cleaned up), but you get the full Kubernetes behavior.

The trade-off is obvious: slower, needs infrastructure, leaves state in the cluster. Use this for integration tests that genuinely need the full lifecycle. Use the local control plane for everything else.

## Practical Tips

**Keep test suites fast.** Start one `Environment` per package, not per test. The startup cost (2-5 seconds) amortizes well across many tests but adds up if repeated.

**Set `ErrorIfCRDPathMissing: true`.** Always. A missing CRD directory should fail loudly, not silently produce tests that pass for the wrong reason.

**Use `KUBEBUILDER_ATTACH_CONTROL_PLANE_OUTPUT=true` to debug.** This pipes the API server and etcd stdout/stderr to your test output. When a test fails with mysterious API errors, the control plane logs usually explain why.

**Test at the right level.** If your controller creates a Deployment, test that the Deployment was created with the correct spec. Don't try to verify that the Deployment becomes Available -- that requires the Deployment controller, which isn't running. Trust the boundary.

**Clean up between tests.** envtest doesn't reset state between tests. Objects created in one test persist to the next. Use unique namespaces per test, or delete objects in `AfterEach`. The namespace lifecycle controller doesn't run either, so deleting a namespace doesn't delete its contents -- delete individual objects explicitly.
