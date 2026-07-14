---
status: Active
maintainer: pacoxu
last_updated: 2026-07-09
tags: ai-agents, agent-infra, llmops, observability, sandbox, memory, governance
canonical_path: agent-infra/ai-agent-infra-9-layers-zh.md
---

# AI Infra Landscape: A Layered Breakdown of Agent Frameworks, Scheduling, Orchestration, Sandboxes, Memory Management, and Tracing

This document provides a **layered architectural perspective on production-grade AI Agent infrastructure**.

The repository's primary landscape focuses on the Cloud-Native AI infrastructure ecosystem—mapping the positions of components such as Kubernetes, scheduling, inference, training, observability, and security isolation. The diagram below introduces a different perspective: the path a request traverses when an Agent enters a production environment, the key questions addressed at each layer, and how cross-cutting capabilities—security, deployment, cost management, and developer experience—span the entire lifecycle.

![AI Agent Infra 9 Layers and 4 Cross-Cutting Capabilities](../diagrams/ai-agent-infra-9-layers-crosscut.png)

Image source: [anata_404 on X](https://x.com/anata_404/status/2075046551563727126/photo/1).
This article is based on the "9 Layers + 4 Cross-Cutting Capabilities" perspective from that diagram, incorporating existing repository content on Kubernetes, inference, Agent sandboxes, memory management, and observability.

Key Insight: Most teams prioritize investments in **L4 Agent Frameworks** and **L2 Vector Databases/Knowledge Bases**. However, the stability of production-grade Agents often hinges on other layers, including model gateways, context management, tool execution boundaries, state memory, evaluation gates, tracing, cost attribution, and security governance.

## 9 Layers + 4 Cross-Cutting Capabilities

| Layer | Name | Core Question | Corresponding Repo Area |
| ---- | ---- | -------- | ------------ |
| L0 | Infrastructure Layer | Where do models and applications run? | | Kubernetes, Runtime, Storage, Networking |
| L1 | Model & Inference Layer | Which model to use? How to invoke it? How to reduce costs? | Inference Engine, AI Gateway, Model Lifecycle |
| L2 | Data & Knowledge Layer | How can models securely and accurately utilize private enterprise knowledge? | RAG, Vector Database, Knowledge Base, Permission Inheritance |
| L3 | Prompt & Context Layer | How to structure inputs for reliable model execution? | PromptOps, Context Engineering, Token Budget |
| L4 | Orchestration & Agent Layer | How are complex tasks decomposed, scheduled, and executed? | Agent Framework, Workflow Engine |
| L5 | Tool Execution Layer | What can agents do? What are the execution boundaries? | MCP, API Connector, Sandbox, Browser Automation |
| L6 | State & Memory Layer | How does the system remember everything without exceeding authorization? | Short-term Memory, Long-term Memory, Checkpointing |
| L7 | Evaluation & Quality Layer | Did quality improve or degrade after changes? | Eval, Golden Set, Release Gates |
| L8 | Observability & Operations Layer | Can issues be pinpointed? Can costs be attributed? | Tracing, Metrics, Logs, SRE |

Cross-cutting capabilities spanning all layers:

- **Security Governance**: Identity, permissions, tenant isolation, prompt injection protection, DLP, auditing,
model supply chain security.
- **CI/CD & Release Governance**: Code, prompts, models, RAG indices, tool schemas, and workflows
must support versioning, canary releases, and rollbacks.
- **FinOps Cost Governance**: Tokens, GPUs, vector retrieval, embeddings, reranking, log retention,
storage, and bandwidth must be attributable to specific applications, tenants, users, and tasks.
- **Developer Experience (DevEx)**: Playground, trace replay, prompt debugging, RAG debugging,
evaluation dashboards, SDKs/CLIs, and project templates.

## L0: Infrastructure Layer

L0 serves as the physical and cloud-native foundation for all AI systems, answering the question: "Where do models and AI applications run?" Key Components:

| Category | Capability | Common Choices |
| ---- | ---- | -------- |
| Compute | GPU / TPU / NPU / CPU | NVIDIA GPU, Google TPU, Cloud provider accelerators |
| Orchestration | Container & task scheduling | Kubernetes, Ray, Slurm, Volcano, Kueue |
| Storage | Object / Block / File / Cache | S3, MinIO, JuiceFS, Alluxio, Dragonfly |
| Networking | High-speed interconnect & service governance | RDMA, InfiniBand, VPC, Service Mesh |
| Image & Model Distribution | Container images, model weights, artifact versions | Harbor, Artifact Registry, OCI Artifacts |
| Basic Security | Secrets, isolation, tenant boundaries | KMS, Secret Manager, RuntimeClass, NetworkPolicy |

Production Practices:

- Inference services require elastic scaling, cold-start optimization, model weight pre-warming, and request queuing.
- Training and batch jobs require queuing, fair sharing, Gang Scheduling, and topology-aware scheduling.
- Model weights, images, and datasets should be stored in a unified artifact repository rather than scattered across local disks.
- Multi-tenant platforms must design namespaces, queues, quotas, NetworkPolicies, and runtime isolation in an integrated manner.

Repository Further Reading:

- [Kubernetes Learning Plan](../docs/kubernetes/learning-plan.md)
- [Scheduling Optimization](../docs/kubernetes/scheduling-optimization.md)
- [Workload Isolation](../docs/kubernetes/isolation.md)
- [GPU Pod Cold Start](../docs/kubernetes/gpu-pod-cold-start.md)

## L1: Model and Inference Layer

L1 manages model sourcing, invocation, routing, and inference costs, serving as the model entry point for the Agent system.

Core Capabilities:

- **Model Gateway**: A unified entry point for invoking models from various providers as well as self-hosted models.
- **Model Router**: Selects models based on task type, context length, quality requirements, and cost budgets.
- **Inference Server**: Hosts self-hosted model inference (e.g., vLLM, TGI, TensorRT-LLM, SGLang, etc.). - **Model Registry**: Manages model versions, metadata, evaluation results, canary deployments, and rollbacks.
- **Fallback / Rate Limit / Quota**: Handles timeouts, failures, rate limiting, and tenant budgets.
- **Cache / Batching / Streaming**: Reduces latency and costs.
- **KV Cache / Prefix Cache / Quantization**: Optimizes throughput and GPU memory utilization.

Production Best Practices:

- Route simple tasks to smaller models and complex tasks to high-quality models.
- Set token budgets for each application, tenant, and user; trigger service degradation or manual approval when limits are exceeded.
- Log prompts, model versions, RAG indices, and tool versions together to ensure trace reproducibility.
- When building a custom inference platform, monitor TTFT, TPOT, ITL, throughput, queue latency, and GPU utilization.

Repository Further Reading:

- [Inference Overview](../docs/inference/README.md)
- [Model Switching and Dynamic Scheduling](../docs/inference/model-switching.md)
- [Caching Strategies](../docs/inference/caching.md)
- [Model Lifecycle Management](../docs/inference/model-lifecycle.md)

## L2: Data and Knowledge Layer

L2 transforms enterprise data into context that models can safely utilize, serving as the foundation for RAG and knowledge augmentation. Typical Pipeline:

```text
Data Source -> Parsing/Cleaning -> Chunking -> Embedding -> Indexing -> Retrieval -> Rerank -> Context Injection
```

Key Capabilities:

| Stage | Focus Areas |
| ---- | ------ |
| Data Source Connection | APIs, Database CDC, Object Storage, File Systems, Web Pages, SaaS |
| Document Parsing | PDF, Tables, Images, OCR, Structured Extraction |
| Chunking | Fixed-length, Recursive, Semantic, Structure-aware |
| Embedding | Multilingual, Multimodal, Domain-specific models, Version compatibility |
| Indexing | Vector Index, Full-text Index, Knowledge Graph, Hybrid Retrieval |
| Rerank | Cross-Encoder, LLM Rerank, Rule-based Hybrid |
| Permission Inheritance | Document-level, Field-level, Tenant-level, User-level ACLs |
| Data Governance | Masking, DLP, Data Lineage, Right to be Forgotten, Retention Policies |

From Naive RAG to Agentic RAG:

- **Naive RAG**: Retrieve Top-K, concatenate into the prompt, and generate directly.
- **Advanced RAG**: Query rewriting, hybrid retrieval, reranking, citations, and confidence control.
- **Agentic RAG**: The agent proactively determines when to retrieve, what to retrieve, and whether secondary retrieval or tool invocation is necessary.

Production Best Practices:

- Permissions must be inherited from the data source through to the index and retrieval results; filtering solely at the UI layer is insufficient.
- RAG indices must be versioned; production responses should be traceable back to specific index versions.
- Citations and evidence chains are standard requirements for enterprise Q&A systems, not optional features.
- RAG quality must be managed in tandem with business data update frequencies to prevent stale knowledge from persisting in the context.

## L3: Prompt and Context Layer

L3 manages the structure of the context fed into the model. It is often underestimated but frequently determines system stability.

The context for a single LLM call typically includes:

- System Prompt: Role definition, behavioral constraints, and safety boundaries.
- Developer Prompt: Tool descriptions, output formats, and business rules. - RAG Results: Retrieved knowledge snippets, citations, and evidence.
- Few-shot Examples: Demonstration input-output pairs.
- User Profile: Preferences, permissions, language, and region.
- Conversation Memory: Recent N turns of dialogue and intermediate states.
- User Prompt: The user's current request.

PromptOps requires the following capabilities:

| Capability | Description |
| ---- | ---- |
| Prompt Registry | Centralized management of templates, variables, use cases, and owners. |
| Version Management | Versioning, diffs, reviews, and rollbacks for every change. |
| Experimentation | A/B testing, canary releases, and metric comparisons. |
| Approval | Mandatory reviews for high-risk or system prompt changes. |
| Context Compression | Priority-based compression or truncation when token limits are exceeded. |
| Token Budget | Budget allocation for RAG, memory, tool outputs, and user input. |
| Output Contract | JSON Schema, function-calling schemas, and structural constraints. |

Production Best Practices:

- Manage prompts as code, integrating them into Git, code review, and release workflows.
- Explicitly log the sources of assembled context to facilitate tracing the reasons behind model responses.
- Assign trust levels to tool outputs, RAG snippets, and user inputs to mitigate prompt injection risks.
- Align token budgets with cost budgets to prevent long contexts from silently driving up costs.

## L4: Orchestration and Agents
