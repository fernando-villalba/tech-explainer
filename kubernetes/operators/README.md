# Building Kubernetes Operators

A Kubernetes operator is a set of controllers that manage custom resources. You define a new resource type -- say, `MultigresCluster` -- and write a program that watches for instances of it and makes the cluster converge toward the desired state. User creates a `MultigresCluster`. Your operator creates the Pods, Services, ConfigMaps, PVCs, and StatefulSets that make it real. User changes the spec. Your operator detects the drift and reconciles. User deletes it. Kubernetes garbage-collects the children.

That's the whole idea. Everything else -- the frameworks, the scaffolding tools, the code generators, the SDKs -- exists to make that loop reliable and the boilerplate manageable.

## The Toolchain Is Layers, Not a Framework

Most tutorials present operator development as "install kubebuilder, follow the steps." That framing hides the actual architecture. There are four layers, each building on the one below:

**apimachinery** (`k8s.io/apimachinery`) -- The type system. Defines how Kubernetes objects work at the lowest level: `runtime.Scheme` (a registry mapping Go types to API group/version/kind triples), `metav1.ObjectMeta` (the metadata every object carries), `metav1.Condition` (standardized status reporting), label selectors, error types like `IsNotFound` and `IsConflict`. Every Kubernetes library depends on this. You use it constantly without thinking about it.

**client-go** (`k8s.io/client-go`) -- The raw Kubernetes client. Provides informers (long-lived watch connections that stream changes), listers (local indexed caches), the REST client, event recording, leader election primitives, and retry utilities. Powerful but low-level. Building an operator directly on client-go means managing your own informer lifecycle, work queues, error handling, and cache synchronization.

**controller-runtime** (`sigs.k8s.io/controller-runtime`) -- The framework. Wraps client-go into the abstractions you actually use: a Manager that coordinates everything, a split Client that reads from cache and writes to the API server, a Builder that wires event sources to reconcilers, predicates that filter events, and a webhook server. This is what runs in your cluster. When your `Reconcile` function is called with a name and you call `r.Get()` to fetch the object, that's controller-runtime.

**kubebuilder / operator-sdk** -- Scaffolding tools. Generate the project skeleton: Go files, Makefile, Dockerfile, Kustomize configs. kubebuilder writes the starter code and gets out of the way. Operator SDK is kubebuilder's CLI wrapped with extra plugins for OLM distribution and non-Go languages. Neither has runtime presence. Neither runs in your cluster.

Alongside these sits **controller-gen** -- a code generator from the controller-tools project that reads marker comments in your Go source (`// +kubebuilder:validation:Minimum=0`) and deterministically generates CRD YAML, RBAC roles, and DeepCopy methods. Despite the `+kubebuilder:` prefix, it's a separate tool. kubebuilder scaffolds the markers. controller-gen reads them.

## What Matters More Than You Think

**controller-runtime** is the layer where operator development actually happens. Understanding how the cache works, how the event pipeline routes changes to reconcilers, how the Manager sequences startup, and how webhooks integrate -- that's the difference between an operator that works and one that has subtle bugs under load. The deep dives are in the [controller-runtime explainers](controller-runtime/).

**controller-gen** is the tool you'll run most often. Every time you change a type or add an RBAC marker, `make manifests` and `make generate` call controller-gen to regenerate CRD schemas, ClusterRoles, and DeepCopy methods. It's deterministic -- same input, same output -- which is exactly what you want for API schemas that the Kubernetes API server enforces. The [kubebuilder overview](kubebuilder/00-overview.md) covers the six generators and how the Makefile wires them.

## What Matters Less Than You Think

**kubebuilder's incremental scaffolding.** `kubebuilder create api` and `kubebuilder create webhook` generate boilerplate -- an empty types file, a controller stub with `// TODO: your logic here`, a webhook with empty validation methods. For a 1-2 CRD operator, this saves time. For anything complex, developers outgrow the scaffolds during development and never run them again.

The [multigres-operator](https://github.com/multigres/multigres-operator) -- a production-grade operator for sharded PostgreSQL clusters -- used kubebuilder once for the initial project setup. It has 13 CRDs. kubebuilder knew about 4 of them before the PROJECT file was deleted. The controllers live in a domain-driven package structure that kubebuilder can't scaffold. The webhooks use a custom registration pattern. Every scaffold marker was removed from the source. What survived: the Makefile wiring and the `config/` directory layout.

**The choice between kubebuilder and Operator SDK.** For Go operators, they produce identical code. Operator SDK adds OLM distribution tooling and non-Go language plugins (Ansible, Helm). If you need OLM, use Operator SDK. If you don't, use kubebuilder. If you change your mind later, switching costs nothing. The [Operator SDK explainer](kubebuilder/01-operator-sdk.md) covers this.

## AI Changes the Equation

Scaffolding tools solve a boilerplate problem. AI solves a boilerplate problem *and* an architecture problem. When you describe a CRD to AI -- its domain meaning, its relationships to other types, its validation constraints -- you get types with markers, a controller pre-wired with the right `Owns()` and `Watches()` calls, and builder functions with OwnerReference setup. Not empty stubs. Code shaped by context.

More importantly, AI helps with the parts scaffolding never attempted: ownership hierarchy design, failure handling strategy, cache configuration, graceful deletion protocols, status reporting patterns. These are the hard problems in operator development. controller-gen stays in the pipeline -- you don't replace deterministic generators with probabilistic ones. But the scaffolding ritual is increasingly a conversation with AI rather than a CLI command. The [operators in the age of AI](kubebuilder/02-operators-in-the-age-of-ai.md) explainer digs into this with real data.

## The Explainers

### [Controller-Runtime](controller-runtime/)

The runtime framework. Start here if you're building an operator or debugging one.

| # | Title | What you'll understand |
|---|-------|----------------------|
| 0 | [Overview](controller-runtime/00-overview.md) | The reconciler interface, the four-stage event pipeline, workqueue deduplication, SSA, ownership and garbage collection, graceful deletion |
| 1 | [The Cache and Client](controller-runtime/01-cache-and-client.md) | Split client, label-filtered caches, cache topologies, field indexers, metadata-only watches, eventual consistency |
| 2 | [Webhooks](controller-runtime/02-webhooks.md) | Admission and conversion webhooks, defaulters and validators, cert rotation, why webhooks run on all replicas |
| 3 | [Manager Lifecycle](controller-runtime/03-manager-lifecycle.md) | Six-phase startup, shutdown ordering, leader election, custom runnables, debugging startup hangs |
| 4 | [Testing with envtest](controller-runtime/04-testing-with-envtest.md) | Running a real control plane locally, fake client error injection, komega assertions, false-passing test traps |

### [Kubebuilder](kubebuilder/)

The scaffolding tool, the code generator, and the changing landscape.

| # | Title | What you'll understand |
|---|-------|----------------------|
| 0 | [Overview](kubebuilder/00-overview.md) | The three-tool ecosystem (kubebuilder, controller-gen, controller-runtime), what each CLI command scaffolds, how markers drive code generation |
| 1 | [Operator SDK](kubebuilder/01-operator-sdk.md) | Why Operator SDK is kubebuilder (not an alternative), OLM integration, bundles, scorecard, Ansible/Helm operators |
| 2 | [Operators in the Age of AI](kubebuilder/02-operators-in-the-age-of-ai.md) | What survives from scaffolding in a real operator, why controller-gen is irreplaceable but scaffolding is not, AI as architecture partner |

### [Client-go](client-go/)

The raw Kubernetes client underneath controller-runtime.

| # | Title | What you'll understand |
|---|-------|----------------------|
| 0 | [Overview](client-go/00-overview.md) | The six subsystems inside client-go, how each maps to a controller-runtime abstraction, the informer pipeline, workqueue mechanics, and when to reach past controller-runtime |

### [Apimachinery](apimachinery/)

The Kubernetes type system. The foundation everything else builds on.

| # | Title | What you'll understand |
|---|-------|----------------------|
| 0 | [Overview](apimachinery/00-overview.md) | The Scheme and GVK/GVR coordinate system, TypeMeta and ObjectMeta, the error taxonomy, status Conditions, label selectors, NamespacedName, and resource quantities |

### [Testing](testing/)

Testing strategy for operators -- envtest, e2e, and how AI changes the calculus.

| # | Title | What you'll understand |
|---|-------|----------------------|
| 0 | [Testing Strategy in the Age of AI](testing/00-testing-operators-in-the-age-of-ai.md) | Why the testing pyramid inverts with AI, the correlated error problem, what a real operator's git log reveals about test ROI |
| 1 | [E2E Framework](testing/01-e2e-framework.md) | Real Kind clusters, shared vs dedicated patterns, wait conditions, the testing spectrum |
| 2 | [Testing with envtest](controller-runtime/04-testing-with-envtest.md) | Real etcd + kube-apiserver locally, test helpers, fake client injection, komega, false-passing traps |

## Where to Start

Building a new operator? Read the [controller-runtime overview](controller-runtime/00-overview.md) first. It gives you the mental model for how reconciliation works.

Confused by kubebuilder vs controller-runtime vs controller-gen? Read the [kubebuilder overview](kubebuilder/00-overview.md). It separates the three tools and explains what each one does.

Deciding between kubebuilder and Operator SDK? Read the [Operator SDK explainer](kubebuilder/01-operator-sdk.md).

Debugging a specific problem? The [controller-runtime README](controller-runtime/) has a jump table: stale reads, webhook errors, startup hangs, flaky tests.

Want to understand what's underneath controller-runtime? Read the [client-go overview](client-go/00-overview.md). It maps every controller-runtime abstraction to the client-go primitive it wraps.

Hitting "no kind registered" panics or confused about Schemes and GVKs? Read the [apimachinery overview](apimachinery/00-overview.md). It covers the type system everything else builds on.

Setting up e2e tests with real clusters? Read the [e2e-framework overview](testing/01-e2e-framework.md). It covers Kind cluster lifecycle, shared vs dedicated clusters, and how e2e complements envtest.

Questioning whether scaffolding tools still matter? Read [operators in the age of AI](kubebuilder/02-operators-in-the-age-of-ai.md).

Rethinking your test strategy with AI? Read [testing operators in the age of AI](testing/00-testing-operators-in-the-age-of-ai.md). It's backed by git log data showing what 100% coverage actually caught (and didn't).
