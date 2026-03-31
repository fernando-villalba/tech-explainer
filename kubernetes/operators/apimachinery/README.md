# Apimachinery Internals

A deep-dive explainer on [apimachinery](https://github.com/kubernetes/apimachinery), the Kubernetes type system. Written by reading the source, not the docs.

apimachinery is the foundation everything else builds on. client-go uses it. controller-runtime uses it. Your CRD types embed its structs. Every `IsNotFound` check, every label selector, every `NamespacedName`, every `metav1.Condition` in your operator -- that's apimachinery.

Examples throughout reference the [multigres-operator](https://github.com/multigres/multigres-operator) -- a production operator that uses apimachinery's error taxonomy, label selectors with requirements, resource quantities, conditions, and the runtime scheme.

## The Explainers

| # | Title | What you'll understand after reading |
|---|-------|--------------------------------------|
| 0 | [Overview](00-overview.md) | The Scheme and how it maps Go types to GVKs, TypeMeta and ObjectMeta, the error taxonomy every operator needs, status Conditions, label selectors, NamespacedName, and resource quantities |

## Reading Order

This is the lowest layer in the operator stack. You can read it first for foundational knowledge, or come back to it when you hit a concept in the [controller-runtime](../controller-runtime/) or [client-go](../client-go/) explainers that bottoms out in apimachinery.

If you're getting "no kind registered for" panics, confused about Schemes, or wondering what `ObservedGeneration` means on a Condition, start here.
