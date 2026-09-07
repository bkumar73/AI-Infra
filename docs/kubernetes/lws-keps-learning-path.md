---
status: Active
maintainer: pacoxu
last_updated: 2026-08-13
tags: kubernetes, lws, scheduling, gang-scheduling, ai-infrastructure
canonical_path: docs/kubernetes/lws-keps-learning-path.md
source_urls:
  - https://github.com/kubernetes-sigs/lws/tree/main/keps
  - https://github.com/kubernetes-sigs/lws/releases/tag/v0.9.0
  - https://github.com/pacoxu/AI-Infra/issues/361
---

# LWS 高优先级 KEP 学习路径

本文整理 [AI-Infra issue #361](https://github.com/pacoxu/AI-Infra/issues/361)
中的五个高优先级 LeaderWorkerSet（LWS）KEP，面向需要设计多节点训练、
分离式推理和 gang scheduling 集成的平台与调度工程师。事实状态核验于
**2026-08-13**，发布基线为
[LWS v0.9.0](https://github.com/kubernetes-sigs/lws/releases/tag/v0.9.0)。

建议按下面的顺序阅读：

1. **Subgroup**：先理解一个 leader-worker replica 如何切成多个拓扑子组。
2. **StartupPolicy**：再理解 leader 与 workers 的控制器启动依赖。
3. **Gang Scheduling**：把一个完整 replica 交给外部调度器原子放置。
4. **Worker Resizing**：理解组内 worker 数量变更及其破坏性边界。
5. **DisaggregatedSet**：最后把多个 LWS role 组合成一个多角色 workload。

> **状态口径**：`KEP metadata` 表示 `kep.yaml` 当前记录的提案状态和目标
> 里程碑；`发布实现` 表示 v0.9.0 tag 或当前 API 中能够核实的实际能力。
> 二者不一致时分别记录，不用目标里程碑反推实现已经毕业。

## 快速对照

| KEP | 一句话总结 | KEP metadata | v0.9.0 / 当前实现 | Feature gate |
| --- | --- | --- | --- | --- |
| [115 Subgroup](https://github.com/kubernetes-sigs/lws/tree/main/keps/115-Subgroup-support) | 将单个 LWS replica 划分成可独立满足 accelerator island 拓扑的子组。 | `status: provisional`，`stage: alpha`，但 `kep-number: 127` 与目录/标题 115 不一致。 | `subGroupPolicy`、标签和拓扑注解已进入 v0.9.0 API。 | 无独立 gate。 |
| [135 Startup policy](https://github.com/kubernetes-sigs/lws/tree/main/keps/135-startup-policy) | 选择 workers 在 leader 创建后立即创建，还是等 leader Ready 后再创建。 | `status: implementable`，`latest-milestone: v0.4.0`，未声明 stage。 | `spec.startupPolicy` 已进入 v0.9.0 API。 | 无独立 gate。 |
| [407 Gang scheduling](https://github.com/kubernetes-sigs/lws/tree/main/keps/407-gang-scheduling) | 为每个 LWS replica 创建 PodGroup，让外部调度器避免部分调度死锁。 | 仍为 `status: provisional`、`stage: alpha`；metadata 将 v0.9.0 列为 stable 目标。 | v0.9.0 包含 scheduler provider 和 Volcano 集成，但不能据此改写 KEP 状态。 | `PodGroupPerReplica`。 |
| [552 Worker resizing](https://github.com/kubernetes-sigs/lws/tree/main/keps/552-worker-resizing) | 允许通过重建组来改变每个 replica 的 worker 数量。 | `status: implementable`，未声明 stage 或里程碑。 | v0.9.0 和核验时的 `main` API 均未包含提案中的 `resizePolicy`，尚不能按已发布能力使用。 | 无独立 gate。 |
| [766 DisaggregatedSet](https://github.com/kubernetes-sigs/lws/tree/main/keps/766-DisaggregatedSet) | 用一个上层 CRD 协调多个 role 的 LWS、revision 和 Service 生命周期。 | `status: implementable`，`stage: alpha`。 | CRD、API、controller 和 Helm chart 已进入 v0.9.0。 | 无独立 gate；使用独立 CRD/controller。 |

## 1. KEP-115：Subgroup Support

**一句话总结**：`SubGroupPolicy` 把一个 LWS replica 进一步切成固定大小的
子组，使每个子组可以落入独立 accelerator island，同时保留整个 replica 的
leader-worker 身份。

### 核心 API

```yaml
spec:
  leaderWorkerTemplate:
    subGroupPolicy:
      subGroupPolicyType: LeaderWorker
      subGroupSize: 4
```

- `subGroupSize` 是每个 subgroup 的 Pod 数，创建后不可变，且要与组大小满足
  KEP 定义的整除规则。
- `subGroupPolicyType` 支持 `LeaderWorker`（默认，leader 进入首个 subgroup）和
  `LeaderExcluded`（leader 不属于任何 subgroup）。
- controller/webhook 为 Pod 写入
  `leaderworkerset.sigs.k8s.io/subgroup-index`、
  `leaderworkerset.sigs.k8s.io/subgroup-key` 标签及
  `leaderworkerset.sigs.k8s.io/subgroup-size` 注解。
- `leaderworkerset.sigs.k8s.io/subgroup-exclusive-topology` 为 subgroup 增加
  第二层 exclusive topology；它与 replica 级
  `leaderworkerset.sigs.k8s.io/exclusive-topology` 组合成两层放置约束。

### 状态、价值与限制

- **KEP metadata**：目录和标题是 KEP-115，但 `kep.yaml` 的
  `kep-number` 仍为 `127`；状态是 provisional/alpha，里程碑字段仍保留早期
  v0.3.0 信息。应把它视为上游元数据不一致，而不是另一个 KEP。
- **发布实现**：上述 API、标签和注解可在 v0.9.0 API 中核实；没有独立
  feature gate，以字段是否配置为 opt-in 边界。
- **AI Infra 关系**：适用于 TPU slice、NVLink/NVSwitch island 或每个 pipeline
  stage 需要组内紧耦合、组间分散的场景，也是
  [Composite/Sub PodGroup 调研 #359](https://github.com/pacoxu/AI-Infra/issues/359)
  中层级调度模型的 workload-side 表达。
- **限制**：它表达的是 LWS replica 内的固定分组与拓扑，不等同于通用的
  `k-of-n`、层级 quota、分层 preemption 或完整 CompositePodGroup API。

## 2. KEP-135：Startup Policy API

**一句话总结**：`spec.startupPolicy` 控制 worker StatefulSet 是在 leader Pod
创建后立即创建，还是等 leader Pod Ready 后再创建。

### 核心 API

```yaml
spec:
  startupPolicy: LeaderReady
```

- `LeaderCreated` 是默认值：leader Pod 创建后立即创建 workers，不保证
  leader 先 Ready。
- `LeaderReady`：等待 leader Pod Ready 后才创建 worker StatefulSet，适合 Ray
  head、协调器或注册服务必须先可用的应用。
- 该 API 已进入 v0.9.0；KEP 未声明独立 feature gate。

### 状态、价值与限制

- **KEP metadata**：`status: implementable`，只记录
  `latest-milestone: v0.4.0`，没有填写 stage 或毕业序列。
- **AI Infra 关系**：避免 workers 在 leader DNS、监听端口或控制面尚未 Ready
  时启动失败，适合 Ray、MPI bootstrap 和带显式 coordinator 的推理 runtime。
- **限制**：这是控制器的**创建顺序**，不是 scheduler 的资源预留或 dependency
  DAG。`LeaderReady` 也可能放大串行冷启动时间；需要与 KEP-407 的 PodGroup
  资源判断一起评估，不能把它单独视为 gang scheduling。

## 3. KEP-407：Gang Scheduling

**一句话总结**：LWS 为每个 replica 建立独立 PodGroup，并把 leader/workers
关联到该组，让 Volcano、scheduler-plugins 或 YuniKorn 等外部调度器执行
all-or-nothing placement。

### 核心接口与 feature gate

- feature gate：`PodGroupPerReplica`。
- scheduler provider 负责 `CreatePodGroupIfNotExists` 和
  `InjectPodGroupMetadata`，屏蔽不同调度器的 PodGroup CRD、label/annotation
  差异。
- PodGroup 名称按 LWS replica/group index（实现还需处理 revision 生命周期）
  生成，并跟随对应 leader Pod 更新或删除。
- `LeaderCreated` 下 `minMember` 是完整 replica 大小；`LeaderReady` 下 KEP 建议
  `minMember: 1`，但 `minResources` 仍代表整个 replica，前提是所选调度器同时
  正确支持这两个语义。

### 状态、价值与限制

- **KEP metadata**：仍写作 provisional/alpha，同时把 v0.7/v0.8/v0.9 列为
  alpha/beta/stable 目标。这些字段不能作为实际毕业结论。
- **发布实现**：v0.9.0 模块已有 scheduler provider，release note 也包含 Volcano
  PodGroup RBAC 修复；文档仍应以所选 LWS/调度器版本的兼容矩阵为部署依据。
- **AI Infra 关系**：防止资源紧张时只运行多个 leader、所有 workers Pending，
  从而形成“占用 GPU 但没有一个 replica 可服务”的死锁。
- **限制**：LWS 只创建/关联 PodGroup，不实现 queue、quota、fairness 或具体
  调度算法。Kueue 的 admission 与 KEP 学习内容继续由
  [issue #376](https://github.com/pacoxu/AI-Infra/issues/376) 单独跟踪。

## 4. KEP-552：Worker Resizing

**一句话总结**：提案希望允许修改 `spec.leaderWorkerTemplate.size`，并用
`resizePolicy` 明确保持不可变或通过重建现有 Pods 应用新大小。

### 提案 API

```yaml
spec:
  leaderWorkerTemplate:
    resizePolicy: Recreate
    size: 8
```

- `None`：保持当前不可变行为。
- `Recreate`：允许修改 `size`，通过重建既有 Pods 应用组大小变化。
- KEP 明确不在本阶段提供 in-place resize，也不定义以该字段为目标的
  autoscaler。

### 状态、价值与限制

- **KEP metadata**：`status: implementable`，未填写 stage、目标里程碑或
  feature gate。
- **发布实现**：核验 v0.9.0 tag 和 2026-08-13 的 `main` API 后，
  `LeaderWorkerTemplate` 均没有提案中的 `resizePolicy`。因此上面的 YAML 是设计
  草案，不是当前可用配置。
- **AI Infra 关系**：未来可用于调整 tensor/pipeline parallel worker 数量，减少
  GitOps 中先删除再重建整个 LWS 的人工步骤。
- **限制**：`Recreate` 会造成 disruption，且组大小变化可能连带 rank、拓扑、
  checkpoint 和模型并行配置变化。上线前必须等待 API/控制器实现，并设计
  可用性预算与回滚方式。

## 5. KEP-766：DisaggregatedSet

**一句话总结**：`DisaggregatedSet` 用一个上层 CRD 管理 2 至 10 个 LWS role，
为所有 role 形成共同 revision，协调滚动升级并维护 per-role/per-revision
headless Service。

### 核心 API

```yaml
apiVersion: disaggregatedset.x-k8s.io/v1
kind: DisaggregatedSet
spec:
  roles:
    - name: prefill
      spec:
        replicas: 2
        leaderWorkerTemplate: {}
    - name: decode
      spec:
        replicas: 4
        leaderWorkerTemplate: {}
```

- `spec.roles` 至少 2 个、最多 10 个；role 名称唯一。
- 每个 role 内嵌 `LeaderWorkerSetTemplateSpec`，底层仍由 LWS 表达
  leader-worker group。
- controller 以所有 role 模板共同计算 revision，执行 N 维协调滚动升级；
  role 级副本数可以不同，但 revision 进度一起收敛。
- 为 role/revision 创建 headless Service，使上层 router 可以区分不同版本的
  prefill、decode 等后端。

### 状态、价值与限制

- **KEP metadata**：`status: implementable`、`stage: alpha`；没有独立 feature
  gate。
- **发布实现**：CRD、API types、controller 和 Helm chart 已进入 v0.9.0。
- **AI Infra 关系**：把手工维护的双 LWS 或多 LWS 部署提升为一个多角色
  workload primitive，减少 role 配置漂移和 rollout 版本错配，尤其适合 P/D
  disaggregation。
- **限制**：不覆盖 HPA/VPA、多集群、非 LWS backend 或 service mesh/ingress
  流量切分；它不是 KServe、llm-d、AIBrix、Kthena 或 Dynamo 的完整替代。

完整 API、rolling update 和 Service orchestration 分析见
[KEP-766 DisaggregatedSet 深入解读](../blog/2026-06-15/2026-06-15-kep-766-disaggregatedset-ai-workloads_zh.md)。

## 能力边界与关联工作

| 主题 | 本文覆盖 | 继续阅读 |
| --- | --- | --- |
| LWS workload primitive | Subgroup、startup ordering、每 replica PodGroup、worker resize proposal、多 role orchestration | 本文五个 KEP |
| Scheduler placement | 解释 LWS 如何把 replica 交给外部调度器，不比较所有调度算法 | [Scheduling Optimization](./scheduling-optimization.md) |
| Composite / hierarchical PodGroup | 只说明 LWS subgroup 与通用层级 PodGroup 的边界 | [Issue #359](https://github.com/pacoxu/AI-Infra/issues/359) |
| Queue / quota / Kueue | 不在本文重复跟踪 | [Issue #376](https://github.com/pacoxu/AI-Infra/issues/376) |
| 训练作业管理 | JobSet、Kueue、Ray 与 gang 的上层组合 | [Issue #300](https://github.com/pacoxu/AI-Infra/issues/300)（已完成） |
| TAS 实验 | 有/无 topology-aware scheduling 的验证 | [TAS Validation](./tas-validation.md)、[Issue #362](https://github.com/pacoxu/AI-Infra/issues/362)（已完成） |

## 阶段结论

这五个 KEP 形成了一条从“组内结构”到“多角色 workload”的主线：

```text
Subgroup
  -> StartupPolicy
  -> PodGroup per replica
  -> worker size evolution
  -> DisaggregatedSet multi-role orchestration
```

其中 KEP-115/135 的 API 已在 v0.9.0 中可见，KEP-407 已有 scheduler provider
集成但 metadata 尚未同步毕业状态，KEP-552 仍是未进入当前 API 的 proposal，
KEP-766 则以 alpha CRD/controller 进入 v0.9.0。生产选型时应以发布代码和兼容
矩阵为准，而不是只看 KEP 的目标里程碑。

## 参考资料

- [LWS KEP index](https://github.com/kubernetes-sigs/lws/tree/main/keps)
- [LeaderWorkerSet v0.9.0 release](https://github.com/kubernetes-sigs/lws/releases/tag/v0.9.0)
- [LWS v0.9.0 API types](https://github.com/kubernetes-sigs/lws/blob/v0.9.0/api/leaderworkerset/v1/leaderworkerset_types.go)
- [KEP-115 Subgroup Support](https://github.com/kubernetes-sigs/lws/blob/main/keps/115-Subgroup-support/README.md)
- [KEP-135 Startup Policy](https://github.com/kubernetes-sigs/lws/blob/main/keps/135-startup-policy/README.md)
- [KEP-407 Gang Scheduling](https://github.com/kubernetes-sigs/lws/blob/main/keps/407-gang-scheduling/README.md)
- [KEP-552 Worker Resizing](https://github.com/kubernetes-sigs/lws/blob/main/keps/552-worker-resizing/README.md)
- [KEP-766 DisaggregatedSet](https://github.com/kubernetes-sigs/lws/blob/main/keps/766-DisaggregatedSet/README.md)

---

本文在 AI 辅助下整理，并对 KEP README、`kep.yaml`、LWS v0.9.0 release 与
API 类型定义进行了交叉核验。用于生产前，请再次确认所部署版本的 release note、
CRD 和调度器兼容矩阵。
