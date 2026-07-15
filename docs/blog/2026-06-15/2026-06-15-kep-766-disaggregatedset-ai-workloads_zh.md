---
status: Active
maintainer: pacoxu
date: 2026-06-15
tags: kubernetes, lws, disaggregatedset, inference, pd-disaggregation, llm-d
canonical_path: docs/blog/2026-06-15/2026-06-15-kep-766-disaggregatedset-ai-workloads_zh.md
source_urls:
  - https://github.com/kubernetes-sigs/lws/blob/main/keps/766-DisaggregatedSet/README.md
  - https://github.com/kubernetes-sigs/lws/blob/main/keps/766-DisaggregatedSet/kep.yaml
  - https://github.com/kubernetes-sigs/lws/blob/main/api/disaggregatedset/v1/groupversion_info.go
  - https://github.com/kubernetes-sigs/lws/blob/main/api/disaggregatedset/v1/disaggregatedset_types.go
  - https://github.com/kubernetes-sigs/lws/blob/main/config/samples/disaggregatedset_v1_disaggregatedset.yaml
  - https://github.com/kubernetes-sigs/lws/pull/767
  - https://github.com/kubernetes-sigs/lws/pull/773
  - https://github.com/kubernetes-sigs/lws/pull/815
  - https://github.com/kubernetes-sigs/lws/pull/836
  - https://github.com/kubernetes-sigs/lws/pull/840
  - https://github.com/kubernetes-sigs/lws/pull/871
  - https://github.com/kubernetes-sigs/lws/releases/tag/v0.9.0
  - https://github.com/kubernetes-sigs/lws
---


An In-Depth Look at KEP-766 DisaggregatedSet: Why It Matters for AI Workloads
This article is based on the schedule as of June 15, 2026, with facts verified up to June 22, 2026. It is important to clarify at the outset: the proposal PR for KEP-766 was merged on March 26, 2026; subsequent implementations and charts have been integrated into the LWS (LeaderWorkerSet) mainline and are included in LeaderWorkerSet v0.9.0, released on June 17, 2026. Therefore, viewing DisaggregatedSet today, it is no longer merely a draft proposal but an alpha-level, multi-role workload API within the LWS ecosystem that is already being implemented.

The Bottom Line
DisaggregatedSet is neither a complete inference platform nor a control plane intended to replace KServe, llm-d, AIBrix, Kthena, or Dynamo. Its position is more accurately defined as a multi-role workload primitive built upon LeaderWorkerSet.

It addresses a specific problem: when an AI inference service is decomposed into roles such as prefill, decode, router, or others, the platform should not require users to manually maintain multiple interdependent LeaderWorkerSets, nor should it offload concerns like upgrade consistency, revision alignment, and Service lifecycle management to Helm templates or manual scripts. DisaggregatedSet consolidates these roles into a single Kubernetes object, with a controller coordinating the underlying LWS, headless Services, and rolling updates.

From the perspective of AI workloads, its significance lies not merely in the addition of a new CRD, but in the formalization of control-plane semantics—areas most prone to error in disaggregated inference:

Unified management of multiple roles as a single logical service
Coordinated upgrades of multiple LWS instances based on the same revision
Independent replica counts for prefill and decode roles, while ensuring synchronized rollouts
Version-aware backend discovery for upstream routers using per-revision Services
State derivation from existing resources, allowing operations to resume after restarts without relying on fragile local in-memory state
1. Problem Background: Why the Dual-LWS Pattern Becomes Complex
The fundamental concept behind PD (Prefill-Decode) disaggregation is clear: splitting the prefill and decode computational tasks of LLM inference across different workers or nodes. The "prefill" phase leans more towards parallel computation and prompt processing, while the "decode" phase focuses on long-tail token generation and KV cache reuse. By decoupling them, the platform can independently schedule, scale, and optimize these two types of resources.

However, the situation becomes more complex when moving to the Kubernetes workload layer. A typical deployment involves two `LeaderWorkerSet` instances:

A "prefill" LWS: responsible for prompt processing and KV generation.
A "decode" LWS: responsible for continuous token generation.
While these two LWS instances can be declared independently on the surface, they are actually tightly coupled:

They must use compatible models, container images, parallelism parameters, and protocols.
Their upgrades must maintain version consistency; otherwise, the router might route requests to an incompatible combination of roles.
Their Services and `EndpointSlice`s should only begin accepting traffic once the relevant roles are ready.
Their replica counts may differ, but the rollout process must not reduce one role's count to zero prematurely, leaving behind "orphan" workloads.
If the controller restarts mid-process or a user submits a sequence of updates (e.g., A -> B -> C), the system must be capable of correctly deriving the target state.
This is the gap that KEP-766 aims to fill: while LWS can define a leader-worker group, the native LWS does not handle organizing multiple roles into a single disaggregated inference service.


2. API Design: A single object describing multiple roles
Based on the current upstream implementation, `DisaggregatedSet` utilizes a dedicated API group. This value is derived from the `GroupVersion` defined in `api/disaggregatedset/v1/groupversion_info.go`, and the upstream sample also employs this same group/version:

# config/samples/disaggregatedset_v1_disaggregatedset.yaml
apiVersion: disaggregatedset.x-k8s.io/v1 # current upstream GroupVersion
kind: DisaggregatedSet
metadata:
name: disaggregatedset-sample
spec:
roles:
- name: prefill
spec:
replicas: 2
leaderWorkerTemplate:
size: 1
leaderTemplate: {}
workerTemplate: {}
- name: decode
spec:
replicas: 2
leaderWorkerTemplate:
size: 1
leaderTemplate: {}
workerTemplate: {}
The core field is `spec.roles`. Each role has a unique name and embeds a `LeaderWorkerSetTemplateSpec`. This means DisaggregatedSet does not reinvent the "leader-worker group" concept but instead reuses the existing workload semantics of LWS: ultimately, each role maps to one or more LWS instances.

Several API constraints are worth noting:

There must be a minimum of 2 and a maximum of 10 roles.
Role names must be unique and comply with Kubernetes DNS label conventions.
Replica counts for roles must be either all zero or all greater than zero.
Partition settings cannot be configured for LWS rollouts within a role, as cross-role rollout coordination is handled centrally by the DisaggregatedSet controller.
Key labels defined upstream also highlight the design focus. In the current implementation, these constants are defined in `api/disaggregatedset/v1/disaggregatedset_types.go`:

`disaggregatedset.x-k8s.io/name` (SetNameLabelKey)
`disaggregatedset.x-k8s.io/role` (RoleLabelKey)
`disaggregatedset.x-k8s.io/revision` (RevisionLabelKey)
These three labels explicitly map the parent DisaggregatedSet, role, and revision to the sub-resources. For operations and routing, this is far more reliable than trying to infer version status from a collection of LWS names.

3. N-dimensional rolling update: The focus is coordination, not just rolling
With a single Deployment, rolling update semantics are relatively straightforward: new replicas are gradually added while old ones are gradually removed. However, disaggregated inference involves multiple roles collectively forming a single service, rather than just one Deployment.

Consider an example with `prefill=5` and `decode=2`: the two roles have different target replica counts and different scaling step sizes. A naive approach would be to let each LWS perform its own rolling update, but this leads to version inconsistency—for instance, a scenario where the new version of the prefill role has scaled up significantly, while the decode role still consists primarily of the old version. If there are changes to data protocols, KV cache formats, or runtime parameters, this state could cause request failures.

The approach taken in KEP-766 treats the rollout as a synchronized progression within an N-dimensional space. Each role has an "old revision" and a "new revision"; the controller calculates how many old replicas to retain and how many new replicas to add at each step, while adhering to `maxSurge` and `maxUnavailable` constraints. The key here is not the algorithm formula itself, but rather several invariants:

Expand before shrink: Increase capacity for the new revision before reducing the old one to avoid a sudden drop in total capacity.
Synchronized progress by role: Prefill and decode components may have different replica counts, but the rollout must progress based on the same logical revision.
Wait for readiness: The controller checks if replicas are ready before proceeding to the next step, avoiding blind advancement.
Coordinated draining: If the old revision of one role scales down to zero, the old revisions of other roles must also scale down, preventing the creation of incomplete combinations that cannot independently serve requests.
Stateless controller: The controller derives the current state from existing LWS, Services, labels, and annotations, allowing it to resume operations after a restart.
This type of design is crucial for AI workloads because the "available capacity" of an AI service is typically not defined by the number of individual Pods, but by whether a set of roles can collectively fulfill requests.

4. Service orchestration: Paving the way for revision-aware routing
DisaggregatedSet creates a headless Service for each role and each revision. The naming pattern specified in the KEP is:

{disaggregatedset-name}-{revision}-{role}-prv
While this design appears to simply automate Service creation, its implications run deeper. A router for disaggregated inference needs to know:

Which prefill backends currently belong to revision A
Which decode backends currently belong to revision A
Whether revision B possesses a complete set of roles
The ratio of backends across different revisions during a rollout
If a single Service mixes versions, it is difficult for the router to be revision-aware. By creating independent headless Services for each role and revision, the router gains a clearer view of the backend sets via EndpointSlices.

This is also a reason why KEP-766 opted to create independent LWS and Service resources for each revision. Compared to directly manipulating LWS partitions, using independent resources for each revision results in more Kubernetes objects, but it yields superior routing visibility, operational observability, and fault isolation capabilities.

5. Why this matters for AI workloads
5.1 Role decoupling moves beyond architectural diagrams
Many architectural diagrams for disaggregated inference depict prefill and decode as separate boxes; however, actual implementation requires the Kubernetes API to express distinct replica counts, templates, scheduling constraints, and lifecycles for each role. The value of `DisaggregatedSet` lies in encapsulating these roles within a single object, rather than scattering them across multiple LWS instances and scripts.

5.2 Clearer Boundaries for Scaling Strategies
The KEP explicitly designates HPA/VPA and automated scaling as non-goals. This is not a shortcoming but rather a matter of clearly defined boundaries. `DisaggregatedSet` addresses the declaration, upgrading, and service discovery of multiple roles; automated scaling based on latency, queue length, KV cache hit rates, or GPU utilization should still be handled by an upper-layer platform or a dedicated autoscaler.

Such boundaries are crucial for platform teams. Otherwise, if a workload primitive attempts to simultaneously address runtime metrics, routing strategies, capacity forecasting, and multi-cluster scheduling, the API would quickly become unmaintainable.

5.3 Upgrade Risks Shift from "Manual Conventions" to Controller Semantics
Upgrading AI inference services often involves more than just swapping container images. Model formats, KV cache formats, runtime parameters, parallelism configurations, and router compatibility may all change. If prefill and decode components are upgraded separately...
