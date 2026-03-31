# E2E Framework Internals

A deep-dive explainer on [e2e-framework](https://github.com/kubernetes-sigs/e2e-framework), the Go testing framework for end-to-end tests against real Kubernetes clusters. Written by reading the source, not the docs.

envtest gives you a real API server with no nodes. e2e-framework gives you real Kind clusters with actual kubelets, scheduling, networking, and DNS. It tests what envtest can't: does the operator actually work when deployed as a container in a real cluster?

Examples throughout reference the [multigres-operator](https://github.com/multigres/multigres-operator) -- a production operator that runs e2e tests across shared and dedicated Kind clusters, testing webhook rejections, graceful deletion, scaling, and template resolution against a fully deployed operator.

## The Explainers

| # | Title | What you'll understand after reading |
|---|-------|--------------------------------------|
| 0 | [Overview](00-overview.md) | How e2e-framework relates to envtest, the Environment/Feature/EnvFunc architecture, Kind cluster lifecycle, shared vs dedicated cluster patterns, wait conditions, and the full operator testing spectrum |

## Reading Order

Read the [envtest explainer](../controller-runtime/04-testing-with-envtest.md) first if you haven't. e2e-framework occupies a different point on the testing spectrum, and the contrast makes both tools clearer.

If you're deciding between envtest and e2e tests for a specific scenario, the [overview](00-overview.md) has a decision framework at the end.
