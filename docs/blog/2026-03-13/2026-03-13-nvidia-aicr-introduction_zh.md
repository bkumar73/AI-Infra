---
status: Active
maintainer: pacoxu
date: 2026-03-13
tags: kubernetes, nvidia, gpu, aicr, gitops, cluster-runtime
canonical_path: docs/blog/2026-03-13/2026-03-13-nvidia-aicr-introduction_zh.md
source_urls:
  - https://github.com/NVIDIA/aicr
  - https://github.com/pacoxu/AI-Infra/pull/252
---

NVIDIA AICR: Codifying GPU Cluster Installation Experience into Reproducible Recipes
On March 13, 2026, the AI-Infra repository merged PR #252, adding NVIDIA AI Cluster Runtime (AICR) to its landscape and documentation index. Although the project is new, it addresses a very practical issue: while setting up a GPU Kubernetes cluster isn't difficult, doing so reliably, verifiably, and reproducibly—time and time again—is a challenge.

If you have worked on AI platforms, training platforms, or inference clusters, you have likely encountered similar pitfalls:

Even with the same Ubuntu OS, minor kernel version differences lead to inconsistent driver behavior;
The GPU Operator installs successfully, but stability suffers when combined with the Network Operator, specific driver versions, or Kubernetes versions;
A "working" configuration exists only in a senior colleague's runbook, requiring a trial-and-error process whenever the environment changes;
Helm values, cluster baselines, and acceptance criteria are scattered across multiple repositories, making version locking and auditing difficult.
AICR aims to solve precisely this problem: the inability to turn platform expertise into a standardized product.

What is AICR?
AICR can be understood as a tool for generating and validating "recipes" for GPU Kubernetes clusters. It is neither a new Kubernetes distribution nor a managed cloud service; instead, it operates at a higher level, codifying a set of verified component combinations into reusable recipes.

The official documentation highlights three key attributes:

Optimized: Tuned for specific hardware, cloud environments, operating systems, and workloads;
Validated: Subjected to automated compatibility and constraint checks prior to release;
Reproducible: Capable of generating identical deployment results from the same inputs.
In other words, AICR does not "invent" best practices for you; rather, it delivers NVIDIA-verified GPU cluster configurations in a machine-consumable format.

What core problem does it solve?
A key difference between AI infrastructure and standard Kubernetes platforms is the sheer complexity of the compatibility matrix. You need to consider several factors simultaneously:

GPU models (e.g., H100, GB200);
Cloud environments (e.g., EKS, GKE, or self-managed clusters);
OS and kernel versions;
NVIDIA drivers;
Platform components (e.g., GPU Operator, Network Operator);
High-level workload intent (training vs. inference);
Deployment methods (e.g., Helm, ArgoCD, Flux, or custom pipelines).
While none of these variables are new in isolation, the real challenge lies in the combinatorial explosion of their possible combinations. AICR’s approach is to avoid having individual teams manually maintain a "best combination" chart; instead, it directly publishes a set of validated "recipes."

AICR’s Core Abstractions: Recipe, Overlay, and Bundle
These are the three most important concepts for understanding AICR.

1. Recipe
A Recipe is a version-locked configuration for a target environment. You specify the target environment parameters to AICR, such as:

Cloud platform: EKS / GKE / Self-managed;
GPU: H100 / GB200;
OS: Ubuntu;
Business intent: Training / Inference;
Platform: Kubeflow / Dynamo.
AICR then generates a corresponding recipe containing a combination of component versions and configurations known to work together.

2. Overlay
An Overlay represents a layered configuration model. Instead of assembling a massive "values" file from scratch, AICR breaks the configuration down into distinct layers:

Base default layer;
Cloud platform layer;
Accelerator card layer;
OS layer;
Workload layer.
This offers significant value: platform teams can finally manage GPU cluster baselines through composition rather than copy-pasting.

3. Bundle
A Bundle is the concrete deployment artifact derived from a recipe. AICR outputs a deployment-ready directory structure containing Helm charts, values ​​files, checksums, and documentation for each component.

This means a recipe is not merely a set of "recommended guidelines" but an artifact ready for direct integration into GitOps workflows.

What does the AICR workflow look like? Based on the official README, the main workflow of AICR is clear:

# 1. Capture the current cluster state
aicr snapshot --output snapshot.yaml

# 2. Generate a recipe for the target environment
aicr recipe --service eks --accelerator h100 --os ubuntu \
--intent training --platform kubeflow -o recipe.yaml

# 3. Validate the recipe against the live environment using the snapshot
aicr validate --recipe recipe.yaml --snapshot snapshot.yaml

# 4. Render into deployable artifacts
aicr bundle --recipe recipe.yaml --output ./bundles
The underlying logic is highly engineering-oriented:

snapshot captures the "current state";
recipe describes the "target state";
validate checks for discrepancies between the current and target states;
bundle converts the target state into actual deployable artifacts.
This model is well-suited for platform standardization and is ideal for delivery, upgrade, and auditing scenarios.

Key AICR Components
The official documentation highlights three main components:

aicr CLI
The primary entry point. It handles recipe generation, snapshot capture, configuration validation, and bundle output.

aicrd
A REST API service with capabilities mirroring the CLI. It is better suited for CI/CD pipelines or in-cluster services, facilitating automated integration.

Snapshot Agent
Runs as a Kubernetes Job to collect the state of GPUs, drivers, the OS, Operators, and more within the cluster, writing the data to a ConfigMap for subsequent validation.

From a platform engineering perspective, this design makes sense: the CLI is suited for manual operations and local pipelines, while the API and Agent are suited for in-cluster automation.

How does it relate to GPU Operator and DRA?
This is the aspect of AICR most prone to misunderstanding.

AICR is not a replacement for GPU Operator.
GPU Operator handles node-level GPU lifecycle management, such as the installation and maintenance of drivers, device plugins, and monitoring components.

AICR operates at a higher level; it organizes the GPU Operator—along with other platform components—into a validated, cluster-level deployment recipe.

You can think of the distinction this way:

GPU Operator: Manages a specific category of GPU components;
AICR: Orchestrates the entire GPU cluster baseline. AICR is not DRA.
DRA addresses the need for more flexible device resource allocation within Kubernetes, operating at the level of resource models and scheduling interfaces.

AICR, on the other hand, leans more towards cluster installation and configuration management. It does not alter scheduling semantics directly; instead, it ensures that underlying drivers, Operators, networking components, and auxiliary tools are deployed in validated combinations.

Why should AI Infra engineers pay attention to this project?
I believe AICR’s value goes beyond merely "saving some installation time"; it represents a more mature approach to platform engineering.

1. Transforming implicit experience into explicit artifacts
Many GPU cluster best practices have long remained locked away in internal scripts, runbooks, and individual expertise. AICR turns this knowledge into "recipes," meaning it can finally be version-controlled, audited, reused, and consumed via automation.

2. Extending GitOps to cover the cluster baseline
While many teams already use GitOps to manage applications, the underlying components of GPU clusters often still rely on manual maintenance. By producing bundles, AICR makes it easier to incorporate these components into a unified delivery pipeline.

3. Providing a "reference point" for upgrades and validation
When upgrading GPU drivers, Operators, or Kubernetes versions, the biggest worry is often: "It runs after the change, but is it an officially validated combination?" AICR’s recipe and validation mechanisms provide a clear baseline for comparison.

4. Particularly well-suited for multi-environment replication
If your platform needs to be replicated across multiple regions, customer environments, or cloud providers, the recipe model offers greater stability than manually maintaining configuration values ​​files.

It is also important to recognize its current limitations
AICR shows great promise, but it is not a "universal platform" at this stage.

Based on the official README and the recent AI-Infra merge PR, there are several clear boundaries:

Not a cluster creator: It is not responsible for provisioning control planes or cloud resources;
Not a full distribution: It does not replace products like EKS, GKE, or OpenShift;
Limited support matrix: Currently, it primarily covers EKS, GKE, and Kind, as well as H100/GB200 GPUs and Ubuntu;
Focused workload scenarios: Training support centers on Kubeflow, while inference focuses on Dynamo;
A relatively new project: Ecosystem maturity, component coverage, and community feedback are still in the early stages. Therefore, a more accurate characterization would be: AICR is not about "building an AI platform from scratch," but rather about reliably delivering a runtime baseline for GPU clusters.

Supply chain security is another aspect worth noting.
The official README highlights several supply chain security capabilities:

SLSA Level 3 provenance;
Signed SBOMs;
Cosign image attestations;
Checksum validation for components.
This indicates that NVIDIA views this not merely as a set of installation scripts, but is designing it to produce auditable and verifiable delivery artifacts. This is crucial for enterprise platform teams, as the component chain for GPU clusters is typically far more complex than that of standard applications.

My assessment of AICR
If I had to summarize AICR in a single sentence:

It attempts to transform the process of "reliably setting up a GPU cluster" from an empirical exercise into a productized, declarative, and verifiable engineering workflow.

While such projects may not immediately expand the functional boundaries of schedulers, inference engines, or training frameworks, they address a frequently underestimated aspect of AI platform implementation: the consistency and reproducibility of the cluster baseline.

For platform engineers, the most compelling aspect of AICR to watch isn't just the components it supports, but whether it can establish a community-accepted distribution model for "validated GPU cluster configurations."

If this approach succeeds, future GPU platform delivery could increasingly resemble "selecting a recipe and validating the diffs," rather than starting from a massive, complex Excel compatibility matrix.

Summary
NVIDIA AICR is a classic example of a new AI infrastructure project: it is unflashy yet directly addresses the pain points faced by platform teams. As GPU clusters grow in complexity and version combinations become harder to manage, the ability to codify validated cluster configurations into "recipes"—and subsequently output them as deployable, verifiable, and auditable artifacts—is inherently valuable.

AICR is particularly worth your attention if you are involved in:

Building training or inference platforms on Kubernetes;
Standardized delivery of GPU clusters;
Multi-environment replication and GitOps-based platform governance;
Managing compatibility across GPU Operators, networking components, and system baselines.
It is currently in the early stages, but the direction is promising.

References
- [NVIDIA/aicr GitHub 仓库](https://github.com/NVIDIA/aicr)
- [AI-Infra PR #252: Add NVIDIA AI Cluster Runtime (AICR) to AI-Infra landscape and docs](https://github.com/pacoxu/AI-Infra/pull/252)
