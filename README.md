# spring-ai-ascend Java 重构分析与测试方案

> 日期：2026-06-02 · 作者视角：资深架构师 / 软件工程专家 · 语言：中英混合
> 性质：评审/分析文档（`docs/reviews/` 属 non-overriding context，不进入治理主干、不触发 gate 漂移校验）。

本目录是对 `spring-ai-ascend`（手机银行智能体平台）**Java 重构**的系统性解读与测试方案设计，对应四个输出件。

## 文档索引

| # | 文档 | 内容 |
|---|---|---|
| 1 | [01-architecture-summary.md](01-architecture-summary.md) | 整个代码项目架构总结：定位、四支柱、五大核心概念、8 模块×5 plane、**智能体端到端流程图**、文档/代码分叉纠偏 |
| 2 | [02-agent-service-deep-dive.md](02-agent-service-deep-dive.md) | agent-service 五层（access/session/queue/taskcontrol/engine）详解、依赖、入口装配、Task 状态机、现有测试、落地缺口 |
| 3 | [03-l0-l1-architecture.md](03-l0-l1-architecture.md) | L0/L1 定义：**三套 L 编号语义**对照（平台架构层 / 治理层 / 模块内部层），重点是平台 L0/L1 |
| 4 | [04-java-refactor-test-plan.md](04-java-refactor-test-plan.md) | Java 重构测试方案：①全量功能特性清单 ②接口文档 ③七维测试基线（性能/可靠/安全/兼容/资料/部署升级迁移/观测）+ 四层测试策略 |
| 5 | [05-diagrams.md](05-diagrams.md) | **图示总结**（Mermaid，渲染为图片）：系统全景、智能体流程、五层数据流、Task 状态机、三套 L 编号、测试方案结构 |

建议阅读顺序：01 → 02 → 03 → 04；想先看图可从 [05-diagrams.md](05-diagrams.md) 入手。

## 关键结论速览

1. **重构范围**：6 个核心模块（bus/client/evolve/execution-engine/middleware/service）**已全部 Java 化，无残留 .py/.ts/.go**。`agent-service`（142 Java 文件）最完整。
2. **平台定位**：托管异构 Agent 的「运行时 + 治理内核」（Spring AI + Spring Boot 4 + Java 21），双执行模式（`GRAPH` / `AGENT_LOOP`）共享 `SuspendSignal` 中断原语，经 Engine Contract envelope 派发，SPI 扩展，五 plane 隔离，posture-aware + Code-as-Contract 治理。目标自托管在 Kunpeng + Ascend 国产栈（NPU serving 为 roadmap）。
3. **agent-service 五层闭环**：A2A/async 入站 → `AccessGateway` → `TaskControlService`（Task 状态机 + 幂等 + 派发）→ `EngineDispatchApi` → 内部队列 → `EngineDispatcher` → `OpenJiuwenAgentHandler` → `openjiuwen/agent-core-java`；状态/输出双链路回流。
4. **L0/L1**（核心答案）：**L0 = 系统级架构层**（声明式系统边界 + §4 65 条编号约束）；**L1 = 模块级架构层**（把 L0 落成 Spring 组合/HTTP·持久化契约/posture/测试/证据）。一句话：**「L0 定义平台被允许成为什么，L1 定义每个 Spring 模块被允许如何成为它。」** L1 不是成熟度标签（成熟度由 `architecture-status.yaml` 的二元 `shipped:` 决定）。另有「治理层 Layer-0/Layer-1」与「agent-service 内部 L1–L5」两套同名不同义的编号，文档 03 已厘清。
5. **⚠️ 最大风险点**：重构后 agent-service 对外暴露 **A2A JSON-RPC**（`POST /a2a/`），而主干契约 `openapi-v1.yaml` 仍描述 `/v1/runs` REST——两者分叉，列为**高优先级回归风险区**，测试以代码实证为准（详见 04 §2.4）。
6. **测试资产可复用**：现有 31 个测试类（JUnit5/Mockito/Testcontainers/WireMock/REST-assured/ArchUnit/Temporal）、`gate/` Code-as-Contract、`perf/`（JMH+2× 回归）、`deploy/middle-office-reference/`（Helm+RLS+sandbox+observability）、`ops/runbooks/`、`sandbox-policies.yaml`、`v1.0-perf-baselines.yaml`、`dfx/agent-service.yaml`。多数基线为 `design_only`/`schema_shipped`，测试需区分「目标值校验」与「runtime 强制」。

## 方法与诚实声明

- 所有结论附 `<ROOT>` 内文件路径，便于交叉核对；关键状态机/入口/依赖直接引用源码。
- 基线值引自仓库现有 YAML 并标注 `status`，不把设计目标当已实现（遵循 `docs/architecture/l0/09-verification/test-strategy.md` 的「测试诚实规则」与 `CLAUDE.md` Rule D-4）。
