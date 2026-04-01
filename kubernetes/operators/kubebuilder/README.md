# Kubebuilder Internals

A deep-dive explainer on [kubebuilder](https://github.com/kubernetes-sigs/kubebuilder), the scaffolding tool that generates Kubernetes operator projects. Written by reading the source, not the docs.

[Operator SDK](https://github.com/operator-framework/operator-sdk) is covered here too, because it isn't a separate tool -- it's kubebuilder's CLI with extra plugins and commands bolted on for OLM distribution and non-Go languages. If you've been trying to decide between kubebuilder and Operator SDK, the [Operator SDK explainer](01-operator-sdk.md) will clear that up.

Examples throughout reference the [multigres-operator](https://github.com/multigres/multigres-operator) -- a production operator that manages sharded PostgreSQL clusters on Kubernetes -- to show what a kubebuilder-scaffolded project looks like after real development fills in the blanks.

## The Explainers

| # | Title | What you'll understand after reading |
|---|-------|--------------------------------------|
| 0 | [Overview](00-overview.md) | Why kubebuilder's scaffolding CLI is the least important part of the kubebuilder project, the three-tool ecosystem, what each CLI command scaffolds, how markers drive code generation, the plugin architecture, and what most production operators actually use |
| 1 | [Operator SDK](01-operator-sdk.md) | Why Operator SDK is kubebuilder (not an alternative), what it adds (OLM, bundles, scorecard, Ansible/Helm operators), when to use which, and why switching costs nothing |
| 2 | [Operators in the Age of AI](02-operators-in-the-age-of-ai.md) | What survives from kubebuilder's scaffold in a production operator, why controller-gen is irreplaceable but incremental scaffolding is not, and where AI fills the gap kubebuilder never could |

## Reading Order

Start with the [Overview](00-overview.md). It covers everything from `kubebuilder init` through to a deployed operator, and explains how kubebuilder, controller-runtime, and controller-gen divide the work.

The [Operator SDK](01-operator-sdk.md) explainer assumes you've read the overview. It builds on the three-tool model and explains where Operator SDK's additions fit.

If you've already read the [controller-runtime explainer](../controller-runtime/README.md), this will show you how the scaffolding that wires up that runtime gets generated. If you haven't, this explainer will make the controller-runtime explainer click faster when you get to it.
