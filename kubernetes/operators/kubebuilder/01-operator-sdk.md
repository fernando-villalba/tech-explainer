# Operator SDK: Kubebuilder With Batteries for Distribution

You're starting a Kubernetes operator project. You search for how to build one. Half the results say "use kubebuilder." The other half say "use Operator SDK." Both have `init` commands. Both have `create api` commands. Both produce Go projects with controllers, types, and Makefiles. The tutorials look nearly identical. You pick one, vaguely worried you chose wrong, and move on.

Here's the thing nobody tells you upfront: Operator SDK is not an alternative to kubebuilder. It *is* kubebuilder. Literally. The `operator-sdk` binary imports kubebuilder's CLI library, creates an instance of kubebuilder's CLI with `cli.WithCommandName("operator-sdk")`, registers the same Go scaffolding plugin, and adds extra plugins and extra commands on top. When you run `operator-sdk init --plugins go/v4`, it calls the exact same kubebuilder code that `kubebuilder init` calls. Same templates. Same output. Same project.

The difference is everything that happens *after* scaffolding. Kubebuilder gets you from zero to a working Go operator. Operator SDK gets you from a working operator to a distributed, tested, OLM-managed package that can be published on OperatorHub. It also lets you write operators in Ansible or Helm instead of Go.

Two CLIs, one scaffolding engine, different scope.

## Proof: The Code That Proves It

This isn't a marketing claim. Here's what `operator-sdk`'s CLI setup actually does:

```go
import (
    "sigs.k8s.io/kubebuilder/v4/pkg/cli"
    golangv4 "sigs.k8s.io/kubebuilder/v4/pkg/plugins/golang/v4"
    kustomizev2 "sigs.k8s.io/kubebuilder/v4/pkg/plugins/common/kustomize/v2"
    // ...
)

func GetPluginsCLIAndRoot() (*cli.CLI, *cobra.Command) {
    gov4Bundle, _ := plugin.NewBundleWithOptions(
        plugin.WithName(golang.DefaultNameQualifier),
        plugin.WithVersion(plugin.Version{Number: 4}),
        plugin.WithPlugins(
            kustomizev2.Plugin{},       // kubebuilder's kustomize plugin
            golangv4.Plugin{},          // kubebuilder's Go scaffolding plugin
            manifestsv2.Plugin{},       // operator-sdk's OLM manifests plugin (added)
            scorecardv2.Plugin{},       // operator-sdk's scorecard plugin (added)
        ),
    )

    c, _ := cli.New(
        cli.WithCommandName("operator-sdk"),       // rename the binary
        cli.WithPlugins(gov4Bundle, ansibleBundle, helmBundle, ...),
        cli.WithDefaultPlugins(cfgv3.Version, gov4Bundle),
        cli.WithExtraCommands(                     // add OLM commands
            bundle.NewCmd(),
            cleanup.NewCmd(),
            generate.NewCmd(),
            olm.NewCmd(),
            run.NewCmd(),
            scorecard.NewCmd(),
        ),
    )
}
```

Read that carefully. `cli.New()` is kubebuilder's constructor. The Go plugin bundle wraps kubebuilder's `golangv4.Plugin{}` and `kustomizev2.Plugin{}` -- the same plugins kubebuilder uses. Operator SDK adds `manifestsv2.Plugin{}` (OLM manifest generation) and `scorecardv2.Plugin{}` (test scaffolding) into the bundle, then registers extra top-level commands for OLM workflows.

Run `operator-sdk init --plugins go/v4` in one directory and `kubebuilder init` in another. Diff the results. They're identical, except Operator SDK's `PROJECT` file includes the `manifests.sdk.operatorframework.io/v2` and `scorecard.sdk.operatorframework.io/v2` plugins, and the project gets a `config/scorecard/` directory and a `config/manifests/` directory. The Go code, the Makefile, the Dockerfile, the controller, the types -- all the same.

## What Operator SDK Adds

Everything Operator SDK provides on top of kubebuilder falls into four categories: non-Go languages, OLM integration, bundle management, and scorecard testing.

### Non-Go Operator Languages

Kubebuilder only scaffolds Go operators. It has a `helm/v2alpha` plugin, but that generates a Helm *chart from* your operator's Kustomize output -- it doesn't use Helm *as* the reconciler. Completely different thing.

Operator SDK adds two plugins that let you write operators without Go at all. These plugins are developed by the operator-framework team, live in operator-framework repos, and are only available through the `operator-sdk` binary. You cannot use them with the `kubebuilder` binary.

**The Helm plugin.** Run `operator-sdk init --plugins helm` and instead of Go types and controllers, you get a `helm-charts/` directory with a Helm chart and a `watches.yaml` that maps Kubernetes resource types (GVKs) to charts:

```yaml
- group: example.com
  version: v1alpha1
  kind: Nginx
  chart: helm-charts/nginx
```

The operator binary watches for CRs matching that GVK. When one appears, it runs `helm install` with the CR's spec as values. When the CR's spec changes, it runs `helm upgrade`. When the CR is deleted, it runs `helm uninstall`. No Go code. No Reconcile function. The Helm release *is* the reconciliation.

This works well for operators that manage a single application and whose desired state maps cleanly to Helm values. It falls apart for operators that need complex multi-step reconciliation, conditional logic, or fine-grained status reporting. The multigres-operator, for example, could never be a Helm operator -- its reconciliation involves drain state machines, topology registration, cross-resource coordination, and dynamic ownership management that has no Helm equivalent.

**The Ansible plugin.** Run `operator-sdk init --plugins ansible` and you get a `roles/` directory with Ansible roles and a `watches.yaml` mapping GVKs to playbooks or roles. The operator binary watches for CRs and runs the corresponding Ansible content when state changes. Variables from the CR's spec are passed as Ansible extra vars.

Same trade-off as Helm: great for operators that configure existing infrastructure with declarative tooling, limiting for operators that need the full expressiveness of Go and controller-runtime.

### OLM Integration

This is the biggest functional gap between kubebuilder and Operator SDK. OLM -- the Operator Lifecycle Manager -- is a Kubernetes extension that manages operators as first-class packages. It handles installation, upgrades, dependency resolution, and RBAC for operators. It's what powers OperatorHub and is pre-installed on OpenShift clusters.

Kubebuilder knows nothing about OLM. Its deployment model is `make deploy`, which uses Kustomize to apply raw manifests. That works for internal operators where you control the cluster.

Operator SDK provides CLI commands for working with OLM:

`operator-sdk olm install` -- installs OLM itself onto a cluster that doesn't have it.

`operator-sdk olm status` -- checks whether OLM is running and healthy.

`operator-sdk olm uninstall` -- removes OLM from the cluster.

`operator-sdk run bundle <image>` -- deploys your operator through OLM using a bundle image. This creates an OLM `Subscription` and `CatalogSource` that reference your operator's bundle, and OLM handles the rest: creating the namespace, applying the RBAC, deploying the operator pod.

`operator-sdk cleanup <package>` -- removes an operator that was deployed via `run bundle`.

If you're distributing your operator to clusters you don't control -- publishing on OperatorHub, shipping to customers, deploying across an enterprise fleet -- you need OLM. And if you need OLM, you need Operator SDK's tooling to generate the OLM-specific artifacts.

### Bundle Management

An OLM bundle is a directory containing everything OLM needs to install your operator:

```
bundle/
  manifests/
    my-operator.clusterserviceversion.yaml   -- the CSV
    my-operator-crd.yaml                     -- the operator's CRDs
  metadata/
    annotations.yaml                         -- bundle metadata
  tests/
    scorecard/
      config.yaml                            -- scorecard test config
```

The CSV -- ClusterServiceVersion -- is the central artifact. It describes your operator: its name, version, description, icon, maintainers, the CRDs it owns, the CRDs it depends on, the RBAC it needs, and the Deployment that runs it. OLM reads this to know how to install and upgrade your operator.

`operator-sdk generate bundle` creates this directory. It collects CRDs from `config/crd/bases/`, RBAC from `config/rbac/`, the Deployment from `config/manager/`, and composes them into a CSV. It prompts for metadata like display name and description. The `manifests` plugin that Operator SDK bundles into the Go scaffolding plugin generates the CSV base during `create api`, so the information accumulates incrementally.

`operator-sdk bundle validate` checks the bundle for correctness: required fields present, CRDs match what the CSV declares, metadata annotations consistent.

The bundle gets packaged as an OCI image and pushed to a container registry. OLM pulls it from there. This is the distribution mechanism -- your operator becomes an installable package, versioned and publishable like a container image.

### Scorecard

`operator-sdk scorecard` runs a suite of tests against your operator bundle. The tests execute as pods inside the cluster, not locally. Built-in tests check:

- Does the bundle validate against the OLM bundle format?
- Does the CSV have all required fields?
- Does the operator start successfully when deployed via OLM?
- Can it create an instance of its CR from the sample?

You can add custom scorecard tests by building test images and referencing them in the scorecard config. The results come back as a JSON or text report.

Scorecard is optional. Many operators skip it entirely. But if you're publishing on OperatorHub, the review process expects a passing scorecard.

## When to Use Which

The decision is straightforward.

**Use kubebuilder** if you're writing a Go operator and deploying it yourself with `kubectl apply` or Kustomize. Kubebuilder gives you everything you need: project scaffolding, controller-runtime wiring, controller-gen for CRDs and RBAC, and a Makefile that ties it together. The multigres-operator uses kubebuilder for scaffolding and deploys with a Helm chart generated from its Kustomize output -- no Operator SDK needed.

**Use Operator SDK** if any of these apply:
- You want to use Ansible playbooks or Helm charts as your reconciliation logic instead of writing Go.
- You need to distribute your operator through OLM (OperatorHub, OpenShift, enterprise catalog).
- You want the bundle/CSV generation tooling.
- You want scorecard testing.

**Use Operator SDK for Go operators** if you plan to distribute through OLM. For Go scaffolding, you'll get identical code to kubebuilder. The extra value is the OLM commands and bundle generation.

One clarification: you can always add Operator SDK later. Since the Go project layout is identical, you can start with kubebuilder, build your operator, and switch to `operator-sdk` when you need OLM tooling. The `PROJECT` file is compatible. The Makefile is the same. You just install the `operator-sdk` binary and run `operator-sdk generate bundle` when you're ready to package for distribution.

## What Operator SDK Does NOT Do

Operator SDK does not change how your operator runs. The reconciliation loop, the cache, the event pipeline, the webhook server, leader election -- all of that is controller-runtime. Operator SDK doesn't wrap or modify controller-runtime. It doesn't add middleware or change the runtime behavior. A Go operator built with `operator-sdk init` runs the exact same controller-runtime machinery as one built with `kubebuilder init`.

Operator SDK does not replace controller-gen. CRD generation, RBAC generation, DeepCopy generation -- all of that still comes from controller-gen via the Makefile. Operator SDK doesn't touch that pipeline.

Operator SDK's scope is scaffolding (via kubebuilder's plugin system) and distribution (via OLM tooling). At runtime, it disappears just like kubebuilder does.

## The Unifying Idea

Operator SDK is kubebuilder plus a distribution layer. Same scaffolding engine, same project layout, same controller-runtime underneath. The difference is what happens after your operator works: kubebuilder stops at `make deploy`. Operator SDK carries you through to OLM bundles, catalog images, OperatorHub publishing, and scorecard validation.

Next time someone asks "kubebuilder or Operator SDK?", you'll know it's the wrong question. The right question is: "Do I need OLM distribution or non-Go languages?" If yes, Operator SDK. If no, kubebuilder. And if you change your mind later, switching costs nothing -- because one is built on top of the other.

---

Previous: [Overview](00-overview.md) | Next: [Operators in the Age of AI](02-operators-in-the-age-of-ai.md)
