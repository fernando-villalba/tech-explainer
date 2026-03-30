# Controller-Runtime Internals

A series of deep-dive explainers on [controller-runtime](https://github.com/kubernetes-sigs/controller-runtime), the Go framework that powers most Kubernetes operators. Written by reading the source, not the docs.

Examples throughout are drawn from the [multigres-operator](https://github.com/multigres/multigres-operator) -- a production operator that manages sharded PostgreSQL clusters on Kubernetes. It exercises most of controller-runtime's surface area: multi-level ownership hierarchies, custom watch handlers, label-filtered caches, admission webhooks, internal certificate management, Server-Side Apply, graceful drain state machines, and envtest-based integration tests.

## The Explainers

| # | Title | What you'll understand after reading |
|---|-------|--------------------------------------|
| 0 | [Overview](00-overview.md) | Why Reconcile only gets a name, the four-stage event pipeline, how the workqueue de-duplicates, SSA as a write pattern, ownership and garbage collection, graceful deletion without finalizers |
| 1 | [The Cache and Client](01-cache-and-client.md) | The split client, label-filtered caches, three cache topologies, field indexers, metadata-only watches, transform functions, and the eventual consistency patterns you need to internalize |
| 2 | [Webhooks](02-webhooks.md) | Admission and conversion webhooks, defaulters and validators, protecting child resources from direct modification, template deletion guards, cert rotation, and why webhooks run on all replicas |
| 3 | [Manager Lifecycle](03-manager-lifecycle.md) | The six-phase startup sequence, why each phase must come before the next, shutdown in reverse, leader election with fast failover, custom runnables, and how to debug startup hangs |
| 4 | [Testing with envtest](04-testing-with-envtest.md) | Running a real etcd and kube-apiserver locally, test helper patterns, ResourceWatcher, fake client error injection, komega assertions, and the traps that produce false-passing tests |

## Reading Order

Start with the [Overview](00-overview.md). It gives you the mental model that everything else builds on. After that, read whichever piece is relevant to what you're working on -- they're designed to stand alone.

If you're building a new operator from scratch, read them in order. If you're debugging a specific problem, jump directly:

- Controller not reconciling? --> [Manager Lifecycle](03-manager-lifecycle.md)
- Stale reads or memory issues? --> [The Cache and Client](01-cache-and-client.md)
- Webhook returning errors or blocking the API server? --> [Webhooks](02-webhooks.md)
- Tests passing locally but failing in CI? --> [Testing with envtest](04-testing-with-envtest.md)
