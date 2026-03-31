# Testing Operators in the Age of AI

The [multigres-operator](https://github.com/multigres/multigres-operator) has 1,303 commits at the time of this writing. The git log tells a testing story that contradicts the conventional testing wisdom.

65% of all Go code additions were test code (91,303 out of 140,022 lines). 66% of all Go code deletions were test code (35,703 out of 53,947 lines). The most-modified file in the entire repository -- not the most complex controller, not the drain state machine, not the topology client -- is `multigrescluster_controller_test.go`, touched 75 times. More engineering time went into writing and rewriting tests than writing the operator itself.

On February 28, 2026, a single commit added 7,917 lines of test code: "Increasing code coverage in all modules to as close to 100 percent as possible." After that push, over 20 operator bugs were found and fixed that the test suite didn't catch. Drain state machine bugs. Missing environment variables. Stale pod roles. Orphaned resources. Webhook propagation failures. All discovered through behavioral testing and manual audits, not through the test suite that covered every branch.

This isn't a story about bad tests. The tests were well-written. The coverage was real. The problem is structural, and it's specific to AI-assisted development.

## The Correlated Error Problem

When the same AI agent writes both the code and the tests, the tests inherit the agent's misunderstanding of the requirements. The agent doesn't make typos. It writes syntactically flawless code that does the wrong thing, and then writes syntactically flawless tests that confirm the wrong thing passes.

A concrete example from the multigres-operator. The drain state machine needed to match a Kubernetes pod to a topology record from etcd. The code did this:

```go
fmt.Sprintf("%v", p.Id)
```

`p.Id` is a protobuf message containing the pod's name. `%v` on a protobuf message gives the text format representation, not the name field. So instead of comparing `"pool-shard-0-0"` to `"pool-shard-0-0"`, the code compared `"name:\"pool-shard-0-0\""` to `"pool-shard-0-0"`. It never matched. Drain never worked correctly.

The AI wrote tests for this code. The tests passed. The tests used the same `fmt.Sprintf` call to generate expected values, so the assertions matched perfectly. The code was wrong and the tests proved it was right.

The fix was trivial:

```go
func PodMatchesPooler(podName string, p *topoclient.MultiPoolerInfo) bool {
    if p.Id != nil && p.Id.Name == podName {
        return true
    }
    h := p.GetHostname()
    return h == podName || strings.HasPrefix(h, podName+".")
}
```

Access the field directly instead of stringifying a protobuf. But no amount of unit testing would have caught the original bug, because the writer and the tester shared the same blind spot. This is the correlated error problem: when code and tests come from the same source, the errors are correlated. More tests from the same source don't reduce risk. They increase confidence without increasing correctness.

## The Greenfield Tax

The multigres-operator went through three major architectural changes during development. Each one destroyed a significant portion of the test suite.

**StatefulSet to direct Pod management.** The operator originally managed database pools using StatefulSets. This was a reasonable first design -- StatefulSets handle ordered creation, stable network identities, and persistent storage. But StatefulSets don't support the fine-grained lifecycle control the operator needed: individual pod drain, per-pod PVC retention policies, and rolling updates that respect topology awareness.

The migration to direct Pod and PVC management touched the core of the shard controller. The commit `refactor(shard): remove legacy statefulset management` deleted `pool_statefulset.go` and its test file -- 1,185 lines of test code gone in one commit. The `shard_controller_internal_test.go` had to be rewritten to assert on Pod lists instead of StatefulSet resources. The e2e tests needed adaptation too, but that was a 43-line change (`fix(e2e): adapt tests to pod-based pool management`) compared to the 1,429 lines of unit/integration test churn.

**Finalizer removal.** The operator initially used finalizers on every controller to run cleanup logic before deletion. This caused stuck resources when finalizers weren't removed properly -- a namespace-deletion-blocking problem common enough to have its own Kubernetes issue tracker pattern. The redesign replaced finalizers with an annotation-based graceful deletion protocol using OwnerReferences and status conditions.

The commit `refactor(controller): remove finalizers across all controllers` changed 15 files: 116 insertions, 1,150 deletions. The toposerver controller's internal test file was deleted entirely (220 lines). The shard controller's internal test file lost 280 lines. The multigrescluster controller test lost 70 lines. All those tests meticulously verified finalizer addition, removal, and ordering -- logic that no longer existed.

**Data-handler merge into resource-handler.** The shard's data-plane operations (drain state machine, topology registration, backup health) originally ran in a separate controller. This created cross-controller race conditions and required SSA field exclusions to prevent the two controllers from fighting over status fields. The merge combined both controllers into one.

The commit `refactor(shard): merge data-handler into resource-handler` deleted the entire data-handler shard controller and its tests: `shard_controller_internal_test.go` (2,766 lines) and `shard_controller_test.go` (693 lines). 3,459 lines of test code gone in a single commit. These tests had been written, reviewed, maintained, and rewritten through multiple iterations -- all for a controller that no longer exists.

The pattern repeats across the codebase. Nine commits deleted test files outright during architectural refactors. The shard `integration_test.go` was modified 56 times. The shard `containers_test.go` was modified 53 times. The shard `shard_controller_internal_test.go` was modified 47 times. These files churned because the code they tested churned. Every refactor that improved the operator's architecture required a corresponding rewrite of the test suite.

During these transitions, the tests weren't catching regressions. There was nothing stable to regress against. The tests documented the current shape of the code, and that shape changed with each architectural improvement. The time spent rewriting test suites could have been spent building diagnostic tooling, running the operator in Kind, or writing e2e tests that survive refactors because they test behavior, not implementation. The e2e test adaptation for the StatefulSet-to-Pod migration was 43 lines. The unit test destruction was 1,429 lines. That ratio tells the story.

## 100% Coverage and 20+ Bugs

The timeline makes the point clearly. The table below is a small sample -- across 1,303 commits and a few months of development, there were likely 100+ bugs that unit and integration tests did not catch.

**February 28, 2026:** "Increasing code coverage in all modules to as close to 100 percent as possible." 7,917 lines of test code added. Subsequent commits pushed coverage higher. The test suite was comprehensive by any reasonable measure.

**March through late March, 2026:** Bug fixes that the comprehensive test suite didn't prevent:

| Date | Fix | What went wrong |
|------|-----|-----------------|
| Mar 5 | fix(drain): reload pg config after sync standby removal | PostgreSQL config not reloaded after removing a sync standby |
| Mar 5 | fix(drain): cancel stale drains on spec revert | Pods stuck with drain annotations after scale-down was reversed |
| Mar 7 | fix(drain): check primary pod readiness before RPCs | RPCs sent to primary before it was ready, causing failures |
| Mar 8 | fix(shard): shorten multipooler service ID | Service ID exceeded PostgreSQL's `app_name` character limit |
| Mar 8 | fix(shard): use POSTGRES_PASSWORD env var for pgctld | Wrong env var name for the password |
| Mar 10 | fix(operator): scope multiorch watch-targets to shard | Watch targets too broad, causing unnecessary reconciliations |
| Mar 10 | fix(operator): add graceful deletion for orphan TGs and Cells | Orphaned resources not cleaned up during deletion |
| Mar 11 | fix(shard): requeue on stale podRoles | Controller not re-reconciling when pod roles were stale |
| Mar 13 | fix(webhook,observer): validate resource limits and fix observer bugs | Missing validation for resource limits in webhook |
| Mar 20 | fix(webhook): propagate initdbArgs through defaulter | Webhook defaulter dropping initdb arguments |
| Mar 22 | fix(gateway): validate ExternalIPs and annotations | Missing validation for external gateway config |
| Mar 26 | fix(containers): add PGDATA env var to multipooler | Missing environment variable in container spec |

These aren't edge cases. Missing environment variables. Wrong field names. Incomplete cleanup logic. Stale state detection failures. Each of these bugs lived behind a green test suite with near-100% coverage. The tests verified the implementation. The implementation was wrong.

Coverage measures "did this line execute?" It doesn't measure "did this line do what was actually needed?"

## What Actually Found the Bugs

Two things caught bugs that unit and integration tests couldn't.

**The [observer](https://github.com/multigres/multigres-operator/tree/07acad254c7b77dcbbf3ebdb8a228d867e62ee0a/tools/observer).** Originally designed as an aid for chaos testing -- intended to work alongside tools like [Chaos Mesh](https://chaos-mesh.org/) -- the observer runs alongside the operator in the cluster and continuously validates invariants against live state. It has a map of valid drain state transitions:

```go
var drainStateOrder = map[string]int{
    "requested":          0,
    "draining":           1,
    "acknowledged":       2,
    "ready-for-deletion": 3,
}
```

If a pod ever goes from `acknowledged` back to `requested`, the observer flags it. It checks that topology registrations match running pods. It verifies ownerReferences are set for cascade deletion. It monitors SQL replication health. It detects silent data plane failures -- situations where the operator thinks everything is fine but the database isn't actually serving queries.

The observer became necessary for another reason: the operator's development outpaced Multigres itself in maturity. Multigres had liveness and readiness probes, but they didn't accurately report whether the underlying database was ready for queries in every scenario at the time. The observer filled that gap -- running actual SQL queries against the data plane, inspecting Multigres component logs, and unifying all of this information so failures from any layer were caught in one place. It tested the operator and the system it managed simultaneously.

Later, exerciser skills were built so that an AI agent could run fixtures and scenarios against a live cluster while the observer continuously checked the resulting state. The combination -- agent-driven scenario execution plus observer-driven invariant checking -- caught far more bugs than unit tests, integration tests, fixed e2e tests, or manual QA smoke testing could have in the same time. And it caught bugs in both the operator and in Multigres itself, because the observer validates the full stack -- from operator reconciliation down to whether the database is actually serving queries.

No unit test checks for backward drain state transitions because you'd have to imagine the scenario first. The observer watches what actually happens and catches violations the test author never anticipated.

The observer's git history tells its own story. It started simple (`feat(chaos): add Level 0 observer`) and grew as it found bugs: topology checks, connectivity probes, replication health, silent failure detection, startup grace periods. Each addition was a response to a real bug the observer discovered or could have discovered earlier.

**E2e tests and smoke testing.** Deploying the operator in a Kind cluster and creating actual MultigresCluster resources. Watching pods schedule, services resolve, drains execute. The e2e test suite tests behavior: "create a cluster, wait for all pods to be ready, delete a cell, verify the deletion protocol works." These tests survived every refactor because they don't care about internal package structure or function signatures. They care about outcomes.

The observer has a practical advantage over unit tests during active development: it's stable across refactors. Renaming an internal function, restructuring a package, merging two controllers -- the observer doesn't break. It checks "does the drain state only move forward?" That question is valid regardless of how the internals are organized.

## When Unit Tests Do Work

This isn't an argument against unit tests. It's an argument about timing and trust.

Unit and integration tests work well for:

**Stable interfaces.** "Creating a Shard must produce pods with ownerReferences set." That contract doesn't change when the internal implementation changes. Testing it with envtest is cheap, fast, and genuinely catches regressions. The multigres-operator's integration tests for ownerReference setup survived every refactor because the contract is stable even when the code shape isn't.

**Pure functions.** Template rendering, label computation, name generation, resource quantity parsing. These have clear inputs and outputs, no external dependencies, and don't change shape during refactors. They're the canonical unit test target.

**Library code.** The `pkg/testutil` package, the `pkg/util/metadata` helpers, the resolver's validation logic. Code that other code depends on, with stable APIs, where a regression breaks everything downstream.

**Post-stabilization regression guards.** Once a module stops changing shape, the unit tests that survived the refactor period become genuinely valuable. They lock in behavior. They catch accidental changes. This is the testing pyramid working as designed -- but only after the architecture stabilizes.

The mistake isn't writing unit tests. The mistake is writing comprehensive unit tests too early, when the code is still changing shape, and trusting the coverage number when the same AI wrote both sides.

## Breaking the Correlation

The correlated error problem doesn't go away with more tests. It goes away with independent verification. Two approaches worked in practice for the multigres-operator.

**Independent agent review.** A second AI agent reads the code against the spec, with no knowledge of the first agent's reasoning. It doesn't see the tests. It reads the specification, reads the implementation, and asks: "does this actually do what the spec says?" This breaks the correlation because the reviewer doesn't share the writer's blind spots. The protobuf Sprintf bug would have been caught by any reviewer reading the spec ("match the pod by name") and the code (`fmt.Sprintf("%v", p.Id)`) -- the mismatch is obvious when you're not the one who wrote it.

Multiple review passes for different concerns compound the effect. One agent for correctness against spec. Another for Go style and patterns. Another for security. Each pass is cheap. The combination catches things that testing alone misses.

**Behavioral observation.** The observer checks actual cluster state against invariants. It doesn't know or care about the implementation. It watches outcomes. This is independent verification at the system level rather than the code level.

## A Testing Strategy for AI-Assisted Development

Based on what the multigres-operator's git log reveals, here's what the testing timeline looks like for a complex operator:

**From the start:**
- Write detailed specs and design docs. The AI can only be as correct as the spec -- but even with a good spec, the AI will get things wrong, miss edge cases, and develop biases about its own implementation.
- Use independent agent review on every significant change. This is the single most effective bug-finding tool when AI writes the code. A second agent with a clean context does not share the first agent's biases.
- Deploy early and often. Run in Kind. Watch `kubectl get pods`. Build an observer tool or a skill that makes it easy for an agent to test the operator end to end.
- Write lean integration and unit tests for stable contracts only. Do not chase 100% coverage -- actively discourage it. This sounds counterintuitive, but running e2e tests catches more real bugs per hour than waiting for AI to produce comprehensive unit test suites during active development. Yes, you read that right, you will save time by focusing on e2e testing early in development and catch more bugs.
- Accept that things will change. Even with a clean initial design spec, as was the case with the multigres-operator, new ideas emerge, assumptions break, and occasional refactors are inevitable. Aiming for perfect test coverage at every turn wastes time that could go toward behavioral testing and design iteration.
- Give the AI guidelines for writing testable code early, so backfilling tests later is easier. Run every significant implementation through a fresh agent context with a tailored review skill -- one pass for correctness against spec, another for Go coding standards, another for security.

**Once the architecture stabilizes:**
- Backfill unit tests for modules that stopped changing shape. Now they're regression guards, not documentation of a moving target.
- Increase coverage deliberately, targeting complex logic paths where regressions are likely.
- Keep the observer running. It catches classes of bugs that unit tests can't, regardless of project maturity.

**What not to do:**
- Don't chase 100% coverage during active development. The multigres-operator added 7,917 lines of test code in one push and hundreds of thousands across many commits. It still had 20+ bugs afterward, plus many others in between that unit and integration tests never caught.
- Don't trust a green test suite when the same AI wrote both sides. The tests verify the implementation, not the intent.
- Don't write comprehensive unit tests for code that's going to be refactored next week. Those 35,703 deleted test lines in the multigres-operator represent real engineering time that produced zero lasting value.

## The Inverted Pyramid

The traditional testing pyramid puts unit tests at the base: many, fast, cheap. Integration tests in the middle. E2e tests at the top: few, slow, expensive.

When AI writes both code and tests, the pyramid inverts. Behavioral testing -- observers, e2e tests, smoke tests -- goes at the base because it catches real bugs that unit tests miss. Paradoxically, it's also faster and cheaper to create and run than churning hours and tokens on AI writing unit and integration tests at every turn. Integration tests for stable contracts go in the middle. Unit tests go at the top, added once the code stops changing shape, as regression guards for stable interfaces.

This isn't a permanent inversion. As the project matures and the architecture stabilizes, the pyramid gradually rights itself. Unit tests become more valuable. Test churn decreases. Coverage becomes meaningful. But during active development with AI, the traditional pyramid is a trap: it optimizes for the wrong metric (coverage) using a tool with a structural flaw (correlated errors).

The git log doesn't lie. 91,303 lines of test code added. 35,703 lines deleted. 20+ bugs after 100% coverage. The testing strategy that actually found bugs was deploying the operator, watching it run, and having independent reviewers check the code against the spec. Everything else was noise until the architecture settled.
