# Client-go Internals

A deep-dive explainer on [client-go](https://github.com/kubernetes/client-go), the raw Go client library for the Kubernetes API. Written by reading the source, not the docs.

controller-runtime wraps client-go. Most operator developers never import client-go directly. But every `r.Get()`, every cache read, every workqueue requeue, every leader election -- that's client-go running underneath. Understanding it explains why controller-runtime works the way it does, and what to reach for when controller-runtime's abstractions aren't enough.

Examples throughout reference the [multigres-operator](https://github.com/multigres/multigres-operator) -- a production operator that uses client-go directly for event recording, retry logic, port forwarding in e2e tests, and custom cache indexers.

## The Explainers

| # | Title | What you'll understand after reading |
|---|-------|--------------------------------------|
| 0 | [Overview](00-overview.md) | The six subsystems inside client-go, how each maps to a controller-runtime abstraction, the informer pipeline from API server to local cache, workqueue mechanics, and when to reach past controller-runtime into client-go directly |

## Reading Order

Read the [controller-runtime overview](../controller-runtime/00-overview.md) first if you haven't. This explainer assumes you know what a Manager, Client, Cache, and Reconciler are. It shows you the machinery underneath them.

If you're debugging cache behavior, stale reads, or workqueue backpressure, this explainer will give you the mental model to reason about what's happening at the client-go level.
