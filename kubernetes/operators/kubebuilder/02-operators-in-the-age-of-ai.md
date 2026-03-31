# Building Operators in the Age of AI

Every tutorial on building Kubernetes operators starts with the same ritual. Install kubebuilder. Run `kubebuilder init`. Run `kubebuilder create api`. Fill in the Reconcile function. Run `kubebuilder create webhook`. The framework guides you step by step, scaffolding files you'd otherwise have to write by hand.

But here's what the tutorials don't show you: what happens during development, when the operator outgrows every scaffold before it even ships. The types were rewritten. The controllers moved to a domain-driven package structure. The main.go was gutted and rebuilt with custom cache configuration, internal certificate rotation, and distributed tracing. The PROJECT file -- kubebuilder's own memory -- was deleted entirely. The operator has 13 CRDs. kubebuilder knew about 4 of them before it was removed. This didn't happen after years in production. It happened during the initial build.

This is the story of the [multigres-operator](https://github.com/multigres/multigres-operator), an operator managing sharded PostgreSQL clusters on Kubernetes. It was scaffolded with kubebuilder. It never ran `kubebuilder create api` again after the initial setup. Every subsequent CRD, controller, webhook, and architectural decision was made by developers working with AI, not by a scaffolding tool.

That experience forces an uncomfortable question: in a world where AI can generate boilerplate *and* help architect the hard parts, what's left for kubebuilder to do?

## What Kubebuilder Actually Gave the Multigres Operator

To answer that honestly, look at what survived from the original scaffold and what didn't.

**What survived:**

The Makefile. Not the exact one kubebuilder generated -- it's been modified -- but the *structure* is intact. The `make generate` target that calls controller-gen for DeepCopy methods. The `make manifests` target that generates CRD and RBAC YAML from markers. The `make test` target that wires up envtest. The tool version management that downloads controller-gen and setup-envtest to `bin/`. This wiring is genuinely valuable. Getting it right from scratch means understanding which controller-gen flags produce which outputs, how envtest discovers binaries, and how kustomize overlays compose. kubebuilder nailed this on day one.

The `config/` directory structure. Kustomize bases for CRDs, RBAC, manager deployment, webhook configuration, cert-manager integration. The overlays have been modified, but the compositional structure kubebuilder established is still there.

The `go.mod` with the right dependencies. controller-runtime, client-go, apimachinery -- the correct versions, compatible with each other. Dependency compatibility in the Kubernetes ecosystem is non-trivial.

The Makefile structure and `config/` directory layout. Modified, but the bones are still there.

**What didn't survive:**

The `internal/controller/` directory. Gone entirely. Controllers now live in `pkg/cluster-handler/controller/` and `pkg/resource-handler/controller/` -- a domain-driven split between cluster-level orchestration (MultigresCluster, TableGroup) and resource-level management (Cell, Shard, TopoServer). kubebuilder's flat `internal/controller/` layout couldn't express this.

The scaffolded controller stubs. Every controller was written from scratch with the domain logic baked in from the start. The Shard controller alone manages drain state machines, topology registration, backup health monitoring, rolling restarts, and dynamic PVC ownership. No scaffold could have produced this.

The scaffolded webhook stubs. The operator uses a single unified webhook registration via `multigreswebhook.Setup(mgr, resolver, opts)` that handles defaulting and validation for all types through a shared resolver. kubebuilder's one-webhook-per-type scaffold was the wrong pattern here.

The scaffolded main.go. The current main.go has internal certificate rotation with CA bootstrapping, OpenTelemetry distributed tracing, per-namespace cache filtering with label selectors (secrets in the operator namespace are unwatched, secrets in other namespaces are filtered by label), RPC client injection for controllers that talk to the data plane, and custom flag configuration for webhook settings. None of this is scaffoldable because none of it is generic.

Every `// +kubebuilder:scaffold:` marker. All removed -- `scaffold:imports`, `scaffold:builder`, `scaffold:scheme`. The operator registers controllers manually with custom initialization that includes event recorders and API readers that kubebuilder's scaffold pattern doesn't support. There is no trace of kubebuilder's scaffolding left in the Go source.

The PROJECT file itself. Deleted. It tracked 4 of the 13 CRDs before it was removed. The remaining 9 CRDs were created by hand -- types file, DeepCopy markers, CRD markers, controller, builder registration, scheme registration. All the steps that `kubebuilder create api` automates, done manually, because the developer (working with AI) needed control over the structure. At some point the PROJECT file was just dead weight tracking a fraction of the API surface, so it was removed.

## The Hard Part Was Never the Boilerplate

Look at what the multigres-operator actually needed to get right:

**Ownership hierarchies.** MultigresCluster owns Cells and TableGroups. TableGroups own Shards. Shards own Pods, PVCs, ConfigMaps, Secrets, Services, PodDisruptionBudgets. Delete the root and Kubernetes GC cascades through the entire tree. But PVC ownership is dynamic -- when the retention policy is `Delete`, add the OwnerReference; when it's `Retain`, remove it. No scaffold generates this.

**Graceful deletion without finalizers.** The operator uses a three-step annotation-based protocol: parent marks child with `PendingDeletion` annotation, child controller drains and sets `ReadyForDeletion` condition, parent polls and deletes when ready. This avoids stuck finalizers blocking namespace deletion. The pattern spans multiple controllers coordinating through status conditions. kubebuilder's scaffold generates a controller that doesn't even know other controllers exist.

**Cache configuration.** The operator filters its cache by labels to reduce memory. But secrets in the operator's own namespace must be unfiltered (cert-manager creates secrets without the operator's labels). So the cache uses per-namespace rules: unfiltered for the operator namespace, label-filtered for everything else. This required understanding controller-runtime's `cache.Options` deeply -- something kubebuilder's scaffold never touches.

**Template resolution.** MultigresCluster references CoreTemplate, CellTemplate, and ShardTemplate objects. When a template changes, every cluster referencing it must re-reconcile. This requires custom `Watches()` with `EnqueueRequestsFromMapFunc` handlers that list clusters and check template references. kubebuilder scaffolds `For()` and `Owns()`. The interesting watch patterns are always custom.

**Drain state machines.** Before deleting a Shard's pod, the controller must drain active connections, wait for in-flight queries to complete, unregister from the topology server, and verify backup health. This is a multi-step state machine tracked through annotations and status conditions, with different requeue delays for each phase: 2 seconds while draining, 5 seconds while waiting for readiness, 30 seconds for crash-loop detection.

kubebuilder generated none of this. It couldn't have. These aren't boilerplate problems. They're architecture problems. And they're exactly the kind of problems where working with AI -- describing the intent, reviewing the generated code, iterating on the design -- produced better results faster than any template could.

## What controller-gen Does That AI Shouldn't

Before this turns into "throw out all the tools," there's a critical distinction. controller-gen is not kubebuilder. And controller-gen is irreplaceable.

controller-gen reads markers in your Go source and deterministically generates CRD YAML, RBAC ClusterRoles, DeepCopy methods, and webhook configurations. The key word is *deterministically*. Given the same input, it produces the same output, every time. The generated CRD schema matches your Go types exactly. The RBAC role includes exactly the permissions your markers declare. The DeepCopy methods handle every field, every embedded struct, every pointer.

AI is probabilistic. It generates code that's *probably* correct. For business logic, "probably correct" is fine -- you review it, test it, iterate. For CRD schemas that the API server enforces, "probably correct" means silent data loss when a field is wrong. For RBAC roles, "probably correct" means your operator gets `Forbidden` errors in production at 3am. For DeepCopy, "probably correct" means subtle mutation bugs that only manifest under concurrent access.

You don't replace deterministic generators with probabilistic ones. The multigres-operator runs `make manifests` and `make generate` on every build. controller-gen is load-bearing infrastructure. It will stay load-bearing.

The same logic applies to kustomize. It composes YAML deterministically. The Makefile that wires controller-gen and kustomize into the build pipeline is structural, not boilerplate. This is the part of kubebuilder's output that aged well.

## What AI Does That Kubebuilder Can't

kubebuilder solves a narrow problem: generating files that follow a known pattern. It can stamp out a `_types.go` with Spec and Status. It can stamp out a controller with an empty Reconcile. It can wire them into main.go. These are mechanical transformations from inputs (group, version, kind) to outputs (Go files).

AI solves a different problem: reasoning about intent and producing code that fits a specific context.

When the multigres-operator needed a new CRD, the developer didn't run `kubebuilder create api --kind Shard`. They described what a Shard represents in the domain -- a single PostgreSQL replica set within a TableGroup, with a data directory, a connection pool, a topology registration, backup health tracking, and a drain state machine. AI generated the types with field-level validation markers, the controller skeleton with the right `Owns()` and `Watches()` wiring, the builder functions with OwnerReference setup, and the scheme registration. All at once. All shaped by context that kubebuilder has no access to.

The difference is not speed. kubebuilder is fast too. The difference is *scope*. kubebuilder generates empty shells. AI generates shells pre-shaped for the domain. kubebuilder creates a controller that says `// TODO(user): your logic here`. AI creates a controller that already fetches the object, checks for deletion, walks the ownership tree, and stubs out the reconciliation phases based on what you described.

And AI goes further. It helps with the parts kubebuilder never attempted:

- "How should I structure the ownership hierarchy for these five resource types?"
- "What happens if the topology server is down when a shard tries to register?"
- "Should this controller use SSA or patch-based updates?"
- "How do I implement a drain state machine that's safe under concurrent reconciliation?"

These are architecture questions. kubebuilder is a file generator. It has no opinions about architecture. AI does.

## The Spectrum: When Each Tool Matters

This is not a binary "kubebuilder vs AI" choice. It's a spectrum based on operator complexity.

**Trivial operator (1 CRD, simple CRUD).** kubebuilder's full workflow works well. `init`, `create api`, fill in the Reconcile with some straightforward logic, done. AI doesn't add much here that kubebuilder doesn't already handle.

**Moderate operator (2-5 CRDs, some ownership).** kubebuilder's `init` is still useful for the project skeleton. `create api` might work for the first couple of types. But you'll quickly want custom package structure, shared utilities, and non-trivial watch configurations. AI starts earning its keep here -- helping design the ownership model, the status reporting strategy, the error handling patterns.

**Complex operator (5+ CRDs, deep hierarchies, state machines).** kubebuilder's `init` for the skeleton and Makefile, then never again. Every type, controller, and webhook is designed with AI assistance. The architecture -- domain-driven package structure, builder patterns, shared reconciliation utilities, cache configuration, graceful deletion protocols -- requires judgment that no template can provide.

The multigres-operator sits firmly in the third category. 13 CRDs. Five controllers. A custom webhook system. Domain-driven package layout. Drain state machines. Template resolution. Dynamic ownership. kubebuilder's contribution to the final product: a Makefile and a `config/` directory.

## What Stays, What Goes, What's Coming

**Stays: controller-gen.** Deterministic code generation from markers. CRDs, RBAC, DeepCopy, webhook configs. No replacement needed or wanted.

**Stays: the Makefile and build pipeline.** The wiring that connects controller-gen, kustomize, envtest, and Go into a coherent build. Whether kubebuilder generates this or AI generates this, it needs to exist and it needs to be correct.

**Stays: controller-runtime.** The runtime framework. The Manager, the cache, the workqueue, the reconciler interface, the webhook server. This is the platform you build on. It was never scaffolding. It's infrastructure.

**Diminishes: kubebuilder's incremental scaffolding.** `create api` and `create webhook` generate boilerplate that AI generates better, because AI has context about your specific domain. For simple operators, these commands are a quick shortcut. For complex ones, they're abandoned early.

**Emerges: AI as architecture partner.** The gap kubebuilder couldn't fill -- reconciliation design, ownership modeling, failure handling, state machine design -- is where AI provides the most leverage. Not by generating more boilerplate, but by reasoning about trade-offs and producing code shaped by the specific problem.

A well-designed AI skill for operator development could combine the best of both: generate the project skeleton with correct Makefile wiring (what kubebuilder does well), then help architect the domain model, design the reconciliation strategy, and produce types and controllers pre-fitted to the problem (what kubebuilder never attempted). controller-gen stays in the pipeline. The build system stays. The scaffolding ritual becomes a conversation.

## The Uncomfortable Truth

kubebuilder solves a problem that's getting easier to solve by other means. The boilerplate it generates -- types, controllers, webhooks, main.go wiring -- is the part of operator development that AI handles naturally. The hard part it doesn't touch -- architecture, reconciliation logic, failure handling, ownership design -- is the part where AI adds the most value.

controller-gen solves a problem that is not getting easier. Deterministic schema generation from type definitions is a compiler problem, and compilers should not be probabilistic.

The future of operator development isn't choosing between kubebuilder and AI. It's recognizing that they solve different problems at different altitudes. kubebuilder gives you a runway. controller-gen gives you a bridge. AI gives you wings. Use the runway for takeoff if you want. But the distance you cover in the air dwarfs the distance you covered on the ground.

## Testing Changes Too

AI doesn't just change how operators are built. It changes how they should be tested. When the same AI writes both the code and the tests, the tests inherit the same blind spots as the code. The multigres-operator achieved near-100% coverage and still had 20+ bugs afterward. 65% of all Go code additions were test code. 35,703 lines of test code were deleted during architectural refactors. The traditional testing pyramid -- many unit tests, fewer integration tests, few e2e tests -- inverts when AI is writing both sides.

The [testing strategy explainer](../testing/00-testing-operators-in-the-age-of-ai.md) digs into the data: which bugs the test suite missed and why, what actually caught them (behavioral observation and independent agent review), and how to allocate testing effort during active development versus after the architecture stabilizes.
