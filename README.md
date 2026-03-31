# Tech Explainers

Deep-dive technical explainers written by reading source code, not docs. Each one aims to give the reader the "aha moment" -- the mental model that makes everything else click.

## Topics

### [Kubernetes Operators](kubernetes/operators/)

Everything involved in building Kubernetes operators in Go -- from the type system at the bottom to the scaffolding tools at the top.

| Area | What it covers |
|------|---------------|
| [Apimachinery](kubernetes/operators/apimachinery/) | The Kubernetes type system: Schemes, GVK/GVR, ObjectMeta, Conditions, error taxonomy, label selectors |
| [Client-go](kubernetes/operators/client-go/) | The raw Kubernetes client: informer pipeline, workqueue, event recording, leader election, retry utilities |
| [Controller-runtime](kubernetes/operators/controller-runtime/) | The operator framework: reconciler interface, cache, webhooks, manager lifecycle, testing with envtest |
| [Kubebuilder](kubernetes/operators/kubebuilder/) | Scaffolding and code generation: kubebuilder, controller-gen, Operator SDK, and operators in the age of AI |
| [Testing](kubernetes/operators/testing/) | Testing strategy, e2e-framework with real Kind clusters, and how AI changes the testing calculus |

Start with the [operators overview](kubernetes/operators/) for the full map and reading order.
