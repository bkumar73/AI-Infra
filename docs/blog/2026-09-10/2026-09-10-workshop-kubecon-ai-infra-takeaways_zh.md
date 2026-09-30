---
status: Active
maintainer: pacoxu
date: 2026-09-10
tags: kubernetes, dra, scheduling, upgrade, scalability, ai-infrastructure
canonical_path: docs/blog/2026-09-10/2026-09-10-workshop-kubecon-ai-infra-takeaways_zh.md
source_urls:
  - https://github.com/pacoxu/AI-Infra/issues/388
  - https://github.com/pacoxu/AI-Infra/issues/390
  - https://github.com/pacoxu/AI-Infra/issues/396
  - https://github.com/pacoxu/AI-Infra/issues/258
  - https://github.com/pacoxu/AI-Infra/issues/261
  - https://dranet.sigs.k8s.io/docs/contributing/webhook-providers/
  - https://github.com/kubernetes-sigs/dranet/pull/223
  - https://github.com/kubernetes-sigs/dranet/pull/241
---

# 从 DRA 到集群升级：AI Infra 交流中的七个工程问题

本文整理自一次 Workshop 和 KubeCon 期间的技术交流。内容已经去除参会公司、
客户地域和个人信息，保留对 AI 基础设施团队有长期价值的工程问题。

这是一份**脱敏后的现场观察与验证清单**，不是任何厂商或 Kubernetes upstream 的
产品承诺。涉及未来版本、支持周期、跨版本升级和具体实现成熟度的内容，实施前仍需
以最终 release notes、version skew policy 和组件兼容矩阵为准。

## 先说结论

这次交流反复指向同一个变化：AI Infra 的关注点正在从“某项功能是否存在”，转向
“它能否在存量集群中迁移、升级、观测和回滚”。

对平台团队而言，值得优先回答的不是“要不要使用 DRA 或新的调度 API”，而是：

1. 跨节点、节点内和设备准备分别由谁负责；
2. 一个 AI workload 从准入到设备就绪的每段时延能否被解释；
3. 存量 Device Plugin、Kubelet 和节点镜像如何渐进迁移；
4. 大规模突发创建时，调度收益是否会被 API Server 或 etcd 瓶颈抵消；
5. 升级能否在严格维护窗口内分批完成，并保留清晰的回退边界。

## 1. DRA 的难点已经从 API 转向迁移与职责边界

DRA 正逐步成为 GPU、NIC 和其他结构化设备的统一资源表达方式，公开 driver 生态也在
扩大。但 API 进入较高成熟度，不等于 driver、CDI、运行时和平台集成已经同时达到生产
成熟度。

新建集群可以选择较新的 Kubernetes 与 DRA 基线；存量集群通常已经积累了定制
Device Plugin、调度扩展和节点初始化逻辑，迁移成本明显更高。因此更稳妥的路径是：

- 按节点池维护 `Kubernetes × Kubelet × runtime × driver × CDI` 兼容矩阵；
- 盘点现有 extended resource 名称、Device Plugin 定制和业务依赖；
- 先在隔离节点池中验证 DRA，再决定能否按池迁移；
- 分别验证声明创建、调度分配、节点准备、容器注入、健康反馈和释放回收；
- 将 CPU、内存等 native resource 方向视为待验证能力，不提前当作生产基线。

一个容易混淆的问题是“Kubelet 是否完全不参与”。更准确的拆分是：调度器负责全局
放置和资源分配决策；Kubelet 仍参与节点侧 Pod 生命周期与设备准备链路；DRA driver、
CDI 和设备运行时完成具体设备配置。PoC 应记录每一步的 owner、状态和超时，而不是只看
最终 Pod 是否进入 `Running`。

对应跟踪：[DRA lifecycle and driver readiness #388](https://github.com/pacoxu/AI-Infra/issues/388)。

## 2. “两级调度”需要先定义两个层级

AI workload 的调度至少包含两个不同决策面：

- **跨节点决策**：队列准入、gang、配额、公平性、拓扑域和目标节点选择；
- **节点内决策**：GPU/NIC/NUMA 对齐、设备分区、共享容量和设备准备。

Kueue、Volcano、Kubernetes Workload-Aware Scheduling、LWS 和 DRA 并不处于同一层。
对比方案时应明确谁拥有以下决策：

| 决策 | 需要明确的 owner |
| --- | --- |
| 队列与配额准入 | Kueue、Volcano 或其他队列系统 |
| gang / PodGroup admission | 原生 WAS 或第三方调度系统 |
| 节点和拓扑域选择 | kube-scheduler 或替代 scheduler |
| 结构化设备分配 | DRA scheduler integration 与 driver |
| 节点内设备准备 | Kubelet、DRA node plugin、CDI/runtime |
| workload rollout 与恢复 | Job、LWS、JobSet 或上层 controller |

短期内，Volcano 等成熟批调度系统与原生 WAS 很可能继续并存。验证重点不是功能名称
是否重叠，而是两套系统同时部署时，是否会重复控制 admission、gang、preemption 或
placement。

对应跟踪：

- [Workload-Aware Scheduling learning path #390](https://github.com/pacoxu/AI-Infra/issues/390)
- [LWS and WAS integration #396](https://github.com/pacoxu/AI-Infra/issues/396)
- [DRA + Kueue baseline #285](https://github.com/pacoxu/AI-Infra/issues/285)

## 3. GPU Sandbox 不应只验证“能否直通”

Sandbox 访问 GPU 的方案需要同时比较 passthrough、MIG/分区和独占 VM 等路径。
“可以看到 GPU”只是最小功能验证，生产评估还应覆盖：

- driver、CUDA、container runtime 和 guest/kernel 兼容性；
- 吞吐、首请求时延和运行时启动开销；
- 多租户隔离、故障域和设备重置行为；
- 节点升级、驱动升级和问题调试成本；
- DRA claim 与 sandbox 生命周期是否一致释放。

对应跟踪：[GPU sandbox matrix #258](https://github.com/pacoxu/AI-Infra/issues/258)。

## 4. 冷启动必须按阶段测量

现场讨论中的“卡在准备阶段”或“卡在 sleep mode”不能直接转化为优化动作。建议统一
拆成以下阶段：

```text
request / scale trigger
  -> queue admission
  -> scheduling
  -> ResourceClaim allocation
  -> device preparation
  -> sandbox + network + volume
  -> image and model loading
  -> GPU runtime initialization
  -> model warmup / sleep-mode resume
  -> readiness
```

每一段至少记录 P50、P95、P99、失败原因和重试次数。只有这样才能判断 warm pool、
snapshot、模型缓存、设备预分配或 scheduler 优化分别解决了哪一段问题。

对应跟踪：[warm pool and snapshot strategy #261](https://github.com/pacoxu/AI-Infra/issues/261)。

## 5. 大集群升级是控制平面与数据面的两条时间线

大集群经常无法在单个严格维护窗口内完成全部节点升级。升级设计应显式分离：

- 控制平面版本和 API 行为；
- Kubelet version skew；
- 节点 OS、container runtime、driver 和 DRA plugin；
- feature gate 与新 API 的启用时间；
- 业务节点池的 drain、替换和容量恢复。

推荐将 immutable node 作为默认思路：创建新节点池、完成 canary 和兼容性验证、逐步
迁移 workload，最后退役旧节点池。所谓“可回滚”也需要分层描述：二进制、feature gate、
节点池和 workload 可以各自回退，但已经迁移的 API/存储数据不一定能够整体反向恢复。

关于跨 minor 版本升级、长期旧版本支持或未来兼容窗口的讨论，只能作为路线观察；在
upstream 或发行版正式发布支持策略前，不能写入生产 runbook。

## 6. 大规模性能测试要覆盖突发 List/Watch

万级 Pod 突发创建时，瓶颈可能出现在 API Server、admission、etcd、watch fan-out、
scheduler queue 或节点侧准备。Streaming List、Watch List、API Server cache 和 etcd
Range streaming 等方向值得跟踪，但具体可用版本及开关必须基于正式发布材料确认。

建议压测至少包含：

- 数万 Pod 的集中创建和删除；
- controller 或 API Server 重启后的集中 List/Watch；
- 大量 CRD 对象同时更新；
- scheduler/controller 批量重连；
- etcd compaction、defragmentation 或成员异常期间的尾延迟；
- APF 限流下关键控制器是否仍能取得足够并发额度。

指标需要跨组件关联，至少覆盖 API request latency、etcd request/commit latency、watch
初始化、scheduler pending duration、ResourceClaim 状态和 Kubelet/device preparation。

## 7. 网络与 CEL：保持机制简单、边界清晰

DRANET 正在探索通过 DRA 表达和分配网络资源，并通过可扩展 provider 接入不同网络
能力。值得继续跟踪的公开工作包括：

- [Webhook providers](https://dranet.sigs.k8s.io/docs/contributing/webhook-providers/)
- [kubernetes-sigs/dranet#223](https://github.com/kubernetes-sigs/dranet/pull/223)
- [IPVLAN/MACVLAN support #241](https://github.com/kubernetes-sigs/dranet/pull/241)

这条路径与 Cilium/eBPF 的策略、可观测和节点间数据面管理是互补关系，不应把它们
描述成同一层能力。

CEL 适合短小、声明式、可审计的匹配与校验；超长表达式会降低可读性，并可能放大准入
路径的性能和资源消耗风险。平台应限制规则长度与复杂度、增加延迟基准和负向测试，
复杂业务逻辑仍放在可测试、可观测的独立组件中。

## 团队行动清单

| 优先级 | 行动 | 完成信号 |
| --- | --- | --- |
| P0 | 为 DRA PoC 增加端到端分段计时 | 可以区分 scheduler、claim、device preparation、sandbox 和 model warmup |
| P0 | 建立存量 Device Plugin 与版本兼容矩阵 | 每个节点池都有迁移条件和回退边界 |
| P0 | 明确 Kueue/Volcano/WAS/LWS/DRA 的 owner 边界 | 同一决策层不存在两个 active controller |
| P1 | 补齐 GPU sandbox 同口径 benchmark | 至少比较 passthrough、分区和独占隔离路径 |
| P1 | 演练控制平面与节点池分离升级 | 能在维护窗口内暂停、继续或回退一批节点 |
| P1 | 增加 List/Watch 重建与突发创建压测 | 可定位 API Server、etcd、scheduler 或 node-side 瓶颈 |
| P2 | 验证 DRANET provider 与 IPVLAN/MACVLAN 场景 | 有最小配置、失败路径和清理步骤 |
| P2 | 建立 CEL 复杂度与性能门禁 | 有长度限制、基准和拒绝异常表达式的测试 |

最终目标不是一次性采用所有新能力，而是形成一条可以解释、观测、升级和回退的资源
编排链路。
