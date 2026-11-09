---
title: "The Paved Road: Multi-Cloud GitOps for Platform Engineering"
date: 2026-11-09 13:41:00 +0530
categories: [Platform Engineering, GitOps]
tags: [gitops, argo-cd, kubernetes, terraform, helm, multi-cloud, platform-engineering]
description: How a paved road of golden paths, CI-builds/GitOps-deploys, and a four-layer repo model turns multi-cloud delivery into a file and a pull request.
media_subpath: /assets/img/posts/paved-road-gitops
---

*Platform engineering turns complex infrastructure and deployment into a standardized, self-service experience, so teams build, deploy, and scale consistently across clouds, regions, and environments.*

## Compute isn't a platform

Most internal platforms stop at compute. The platform team provisions the cloud accounts, the networking, and a Kubernetes cluster, hands it over, and says: *you deploy and operate your apps on it.*

That sounds empowering. In practice, **every team reinvents application delivery.** Each one writes its own deployment pipelines, wires its own cluster credentials, invents its own folder layout for environment config, and improvises its own promotion and rollback logic. Multiply that by every team, every environment, every cloud, and every region, and you get:

- **Duplicated delivery logic:** the same deployment plumbing, written N slightly different ways.
- **Credential sprawl:** push-based pipelines hold credentials with write access to the cluster, because they run `kubectl apply` or `helm upgrade` directly.
- **No continuous reconciliation:** once a pipeline finishes, nothing keeps the cluster matching what's in Git. Manual "just this once" changes during an incident quietly become permanent drift.
- **Add-on drift:** shared add-ons installed as Terraform Helm releases are only checked when Terraform runs, so live state diverges between applies.
- **Fragile recovery:** rebuilding a cluster means re-running every team's bespoke process to get back to a known state.
- **Expensive onboarding:** a new app, team, region, or cloud is a project, not a pull request.

The problem is growing. Enterprises are moving away from standardizing on a single cloud provider and spreading workloads across AWS, Azure, and GCP for resilience, pricing leverage, regulatory requirements, or because of M&A and legacy footprints. Every pain point above repeats for each team and again for each cloud.

The cognitive load lands on application teams who just want to ship features. The platform gave them raw material and left them to build the factory. This article describes the alternative, a **paved road**.

## Platform engineering is a product

A platform engineering team's job is to build and maintain an **Internal Developer Platform**: a set of **golden paths**, meaning templates, pipelines, charts, and policies that teams adopt by default, with governance and compliance built into the path itself.

Teams shouldn't read a wiki and hope they got security right. They should consume the artifact the platform team maintains and get the guardrails for free. The same idea appears across the main references on the topic:

- The **CNCF Platforms white paper** frames platforms as things that "embed governance and controls in templates, patterns, and capabilities," existing to reduce cognitive load and reduce security and regulatory risk by governing capabilities.
- **Golden paths**, or paved roads, are the supported, opinionated, well-lit route that is the easiest way to do the right thing.
- **[platformengineering.org](https://platformengineering.org/blog/what-is-platform-engineering)** treats the platform as a product with users: the platform team gathers their feedback and keeps iterating, because a platform is a long-term effort and not a short-term fix.
- **GitOps** supplies the delivery mechanism: declarative, versioned, pulled automatically, and continuously reconciled.

A simple test shows whether a platform is paved:

> **How hard is it to onboard a new app, a new region, a new cloud, or a new team?** On a paved platform, that takes a file and a PR. If it takes a project, the road isn't paved yet.

## The core shift: CI builds, GitOps deploys

The most important move is to **separate building from deploying**.

In the old push-based model, a CI pipeline builds the image *and* reaches into the cluster to apply it. That's why CI ends up holding cluster credentials, and why nothing reconciles state afterward.

In the GitOps model, responsibilities split cleanly:

- **CI builds, tests, scans, signs, and publishes** an immutable image, then **opens a pull request** that updates the desired state in Git. It never touches the cluster.
- **An in-cluster GitOps controller (Argo CD) pulls** the approved desired state from Git and **continuously reconciles** the cluster to match it.

![CI builds and publishes an image, opens a PR to pin the digest, and Argo CD pulls the approved change and reconciles the cluster, continuously re-checking for drift](diagram-1-ci-gitops-flow.png)
*CI never touches the cluster. It builds, signs, and opens a PR, and Argo CD pulls the approved state and reconciles.*

Why this matters:

- **No cluster write-credentials in CI.** CI needs to push an image and open a PR, and it has no need to administer Kubernetes. The blast radius of a compromised pipeline shrinks.
- **Continuous reconciliation and self-heal.** The controller constantly compares live state to Git and corrects drift, instead of trusting that a one-shot pipeline left things correct.
- **Git as the audit trail.** Every change is a reviewed commit, so who changed what, when, and why is just `git log`.
- **Boring recovery.** Rebuild a cluster, point the controller at Git, and the desired state flows back in.

**CI/CD stays, with a narrower job.** CI still owns producing and verifying trustworthy artifacts, and GitOps takes over deployment and reconciliation. Any CI system works (GitHub Actions, GitLab CI, Buildkite); the reference implementation uses GitHub Actions, but nothing in the model depends on it.

## The four-layer repository model

Clean delivery starts with clean ownership. Split repositories into four layers by responsibility:

![Four repositories: application source and team config are owned by application teams; platform Terraform and platform GitOps are owned by the platform. CI pins a digest from source into team config, which is reconciled by the platform GitOps repo, which deploys onto clusters provisioned by platform Terraform](diagram-2-four-layer-repo-model.png)
*Application teams own source and config; the platform owns infrastructure and the GitOps control plane.*

![Table: the four-layer repository model, listing layer, responsibility, owner, and typical contents for application source, team configuration, platform Terraform, and platform GitOps](table-1-four-layer-model.png)

Two more platform-owned assets round out the paved road. They are separate repos, not folders in the four above:

- **A golden Helm chart repo:** the standard, opinionated service chart every app renders through.
- **A shared CI workflow repo:** reusable build, sign, promote, and validate workflows that teams call instead of copying.

The platform GitOps repo is a **governance control plane**, and teams do not open a PR against it for every release. Each team's own config repo stays the source of truth for that team's workloads, while the platform repo defines the framework they work within.

## Layered configuration: define only what differs

A team's config repo should let them express a difference only at the level where it occurs. A base `values.yaml` holds service defaults, and environment, cloud, and region files override only what changes for that target. **The deepest matching layer wins.**

```
team-alpha-gitops/
└── apps/
    └── orders-api/
        ├── base/
        │   └── values.yaml                 # service defaults
        └── envs/
            ├── dev/
            │   └── values.yaml             # dev overrides
            └── sbx/
                ├── values.yaml             # sandbox-wide overrides
                ├── aws/
                │   ├── values.yaml         # AWS-specific
                │   └── us-east-2/
                │       └── values.yaml     # region-specific
                └── azure/
                    ├── values.yaml         # Azure-specific
                    └── centralus/
                        └── values.yaml     # region-specific
```

Resolution is a simple precedence chain (base, environment, cloud, region), so sandbox-wide settings, AWS and Azure differences, and per-region tweaks each live exactly once. There are no copy-pasted value trees per cloud.

A leaf file stays tiny because it only pins what is unique to that target, most importantly the **immutable image digest**:

```yaml
# apps/orders-api/envs/sbx/azure/centralus/values.yaml
image:
  repository: sbxacrplatform.azurecr.io/orders-api
  digest: "sha256:3f8a...e21c"   # built once by CI, promoted by digest
```

This is **build-once, promote-many**. CI builds and signs an image one time, and promotion to the next environment or region is a reviewed PR that reuses the same digest, with no rebuild. What you tested is bit-for-bit what you ship.

## The paved-road components

Repository separation defines ownership, but a paved road also needs **reusable components and a supported contract**, so that teams never design delivery machinery themselves:

![Table: the paved-road components, listing capability and what the framework provides for application scaffold, reusable CI workflows, golden Helm chart, config template, Argo CD framework, tenant baseline, release promotion, and validation and policy](table-2-paved-road-components.png)

The golden chart is where governance happens by default. A team sets a handful of values and gets a security context, health probes, a pod disruption budget, an HPA, a workload-identity service account, and externalized secrets without knowing the YAML behind any of it.

Teams should be **thin consumers**. If anyone is copying a workflow file or hand-authoring an ApplicationSet to ship a service, the road isn't paved there yet.

## Platform add-ons through GitOps

Shared add-ons (External Secrets, a service mesh, progressive-delivery controllers, observability agents, policy engines) should follow the same desired-state model as tenant apps. Terraform bootstraps only the cluster foundation and the GitOps entry point, and day-2 add-on lifecycle moves to Argo CD after a clear handoff.

- **Add-on catalog:** each approved add-on with its owner, chart or source, target cluster labels, sync order, and a per-cluster version pin.
- **Promotion path:** validate an add-on change in a canary or sandbox cluster, allow soak time, then promote through dev, staging, and production via reviewed Git changes.
- **Safe migration:** move Terraform-managed Helm releases to GitOps one add-on at a time. Terraform and Argo CD must never manage the same release simultaneously.

Add-ons then get the same continuous reconciliation, self-heal, and staged rollout as applications, so they stop drifting silently between infrastructure applies.

## Onboarding: a file and a PR

This is the test the whole framework is built to pass. Once the underlying infrastructure exists, onboarding is **declarative**: add or edit a small set of files, open a reviewed PR, and you are done:

![Table: onboarding scenarios, listing what you change and what automation does for a new app, new team, new environment, new region, and new cloud](table-3-onboarding-scenarios.png)

"A file away" refers to the application and GitOps onboarding steps. A genuinely new region or cloud still needs real infrastructure (network connectivity, identity, registries, DNS, security integration) provisioned and approved first. The framework makes the delivery layer declarative and repeatable once those prerequisites are met. It doesn't conjure a VPC from a YAML file.

## Multi-cloud and multi-region, without copy-paste

The same team config repo and the same golden chart should serve both AKS and EKS. Cloud differences live behind supported chart values or small adapters, so there are no parallel copies of every team's config.

The mechanism is **label-driven targeting**. Every cluster is registered with a consistent metadata contract: at minimum `environment`, `cloud`, and `region`, plus optional labels for product, tenant eligibility, or compliance boundary. ApplicationSets use those labels to decide what goes where:

![An ApplicationSet with a label selector targets registered clusters by environment label: environment=sbx matches both an Azure Central US sandbox cluster and an AWS us-east-2 sandbox cluster, while environment=dev matches a separate Azure Central US dev cluster](diagram-3-label-driven-targeting.png)
*One ApplicationSet and one label selector, regardless of which cloud or region a matching cluster is in.*

Two properties make this safe at scale:

- **Per-cluster version pins.** Each cluster pins its own chart and add-on versions, so a release moves through sandbox, development, staging, and production in a controlled, staggered way.
- **Bounded blast radius.** A single commit never rolls to every cluster in a stage at once. Each hop is a reviewed PR against one target: canary first, soak, then promote.

## Security and the supply chain

GitOps changes where trust lives, so the controls move with it:

- **Least privilege everywhere.** Application CI needs to publish an image and open a PR, nothing more. The GitOps controller gets read-only access to just the repos and paths it reconciles, with scoped, rotated credentials.
- **Protect Git like a control plane, because it is one.** Use branch protection, required reviews, `CODEOWNERS`, required validation checks, and audit retention on the desired-state repos.
- **Protect the artifacts.** Image signing, provenance and attestation, vulnerability scanning, and digest pinning are complementary, and GitOps does not make an untrusted image safe.
- **Externalize secrets.** Keep plaintext secrets out of Git, and use the External Secrets Operator (or equivalent) to pull from the cloud secret manager via workload identity.
- **Define reconciliation boundaries.** Configure ownership and ignore-difference rules so Argo CD doesn't keep overwriting fields owned by an HPA or a cloud load-balancer controller.

## Responsibility boundaries

Paved roads work because everyone knows what they own. Three roles are separated:

![Table: responsibility boundaries across infrastructure/cloud platform, platform engineering, and application team for cloud accounts, Argo CD control plane, app source and CI tests, app deployment config, deployment reconciliation, and runtime and service health](table-4-responsibility-boundaries.png)

## What this is and what it isn't

A framework like this can be **validated end to end as a proof of concept**: Terraform provisioning AKS and EKS clusters across regions, Argo CD self-managing from Git, image build and signing, same-digest promotion across environments and registries, label-driven ApplicationSets, per-cluster version pins, and tenant namespace deployment all working together.

That shows the architecture is viable, but it is **not a turnkey production platform**. Real adoption still needs hardened identities and registries, production secret integration, RBAC and tenancy boundaries, policy enforcement, observability, and operational runbooks. Placeholder names and sandbox resources get replaced with approved production integrations. Treat the POC as evidence, then harden it and pilot with one team before you scale.

## Reference implementation and further reading

A complete, generic reference implementation of everything above is public:

- **Platform Terraform** (clusters, networking, identity, registries, Argo CD bootstrap): [`imkrupakar/platform-terraform`](https://github.com/imkrupakar/platform-terraform)
- **Platform GitOps** (Argo CD control plane, ApplicationSets, AppProjects, add-ons, tenant onboarding): [`imkrupakar/platform-gitops`](https://github.com/imkrupakar/platform-gitops)
- **Golden Helm charts** (the opinionated, governed service chart): [`imkrupakar/golden-helm-charts`](https://github.com/imkrupakar/golden-helm-charts)
- **Shared CI workflows** (reusable build, sign, promote, validate): [`imkrupakar/platform-actions`](https://github.com/imkrupakar/platform-actions)
- **Example team config** (layered values and promotion): [`imkrupakar/team-alpha-gitops`](https://github.com/imkrupakar/team-alpha-gitops)
- **Example application** (source and CI only, no manifests): [`imkrupakar/orders-api`](https://github.com/imkrupakar/orders-api)

Foundational reading:

- [OpenGitOps principles](https://opengitops.dev/): declarative, versioned, pulled, continuously reconciled.
- [CNCF Platforms white paper](https://tag-app-delivery.cncf.io/whitepapers/platforms/): platforms, golden paths, cognitive load.
- [Argo CD best practices: config vs. source repos](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/#separating-config-vs-source-code-repositories).
- [GitOps architecture patterns and anti-patterns](https://platformengineering.org/blog/gitops-architecture-patterns-and-anti-patterns)
- [What is platform engineering?](https://platformengineering.org/blog/what-is-platform-engineering)

## The point of a paved road

A cluster is raw material, and a paved road is a product: the supported, secure path that makes the right way the easy way. When CI builds and GitOps deploys, configuration is layered instead of duplicated, governance lives in a golden chart, and onboarding a new app, region, cloud, or team takes a file and a PR, application teams stop maintaining delivery plumbing and start shipping outcomes. One set of templates and one control plane give consistent governance on whichever cloud or region a team ships to.
