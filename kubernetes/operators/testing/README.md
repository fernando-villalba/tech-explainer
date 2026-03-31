# Testing Kubernetes Operators

Testing explainers for Kubernetes operators -- from unit tests through e2e, and how AI changes the calculus.

The testing content is spread across multiple folders because each tool belongs to its own ecosystem. This README ties them together.

## The Explainers

| # | Title | Where | What you'll understand |
|---|-------|-------|----------------------|
| 0 | [Testing Strategy in the Age of AI](00-testing-operators-in-the-age-of-ai.md) | here | Why the testing pyramid inverts with AI, the correlated error problem, what the git log of a real operator reveals about test ROI |
| 1 | [E2E Framework](01-e2e-framework.md) | here | Real Kind clusters, Environment/Feature architecture, shared vs dedicated clusters, the full testing spectrum |
| 2 | [Testing with envtest](../controller-runtime/04-testing-with-envtest.md) | controller-runtime | Running a real etcd + kube-apiserver locally, test helpers, fake client injection, komega, false-passing traps |

## Reading Order

Start with the [testing strategy piece](00-testing-operators-in-the-age-of-ai.md). It reframes how to think about testing when AI writes both code and tests, backed by git log data from a real operator. Read this before deciding how to allocate testing effort.

Then read whichever tool applies to what you're doing: [e2e-framework](01-e2e-framework.md) for real cluster tests, [envtest](../controller-runtime/04-testing-with-envtest.md) for fast integration tests against a local API server.
