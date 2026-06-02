# 交付件 1 · 整体架构总结（逻辑 + 智能体流程）

> 范围：`spring-ai-ascend` 全代码项目的架构总结。
> 视角：资深架构师 / 软件工程专家。
> 语言：中文叙述，类名 / 契约名 / 术语保留英文。
> 日期基线：2026-06-02。

---

## 1. 一句话定位

`spring-ai-ascend` **不是又一个 Agent 框架，而是「托管异构 Agent 的运行时 + 治理内核」**。它把每个团队都在重复造的 Agent 编排「胶水」——如何 suspend 去调一个工具、如何让确定性工作流交棒给 LLM 推理循环、如何让所有 Run 保持租户隔离且可审计——抽取成一个可复用、可自托管的平台层，就像 Spring Boot 当年把 Web/Service 胶水抽出来一样。

- 技术栈：**Spring AI + Spring Boot 4.0.5 + Java 21**。
- 部署目标：自托管在华为 **Kunpeng（ARM64 CPU，跑 JVM 服务层）+ Ascend（NPU，serve 模型）** 国产硬件栈，OSS-first、无专有云锁定。
- 诚实边界：当前运行时是**硬件无关**的 JVM 内核（任意 JVM、原生 ARM64 都能跑）；Ascend-NPU 优化的模型 serving、Kunpeng 调优部署 profile 目前是**设计契约（roadmap）而非已落地代码**（见 `docs/governance/architecture-status.yaml`）。

来源：[README.md](../../../README.md)、[docs/overview.md](../../overview.md)、[docs/architect.md](../../architect.md)。

---

## 2. 四大设计支柱

| 支柱 | 内涵 |
|---|---|
| Performance | 非阻塞 run spine（虚拟线程 Loom）+ 并行模块构建；部署目标 Ascend NPU + Kunpeng ARM 吞吐。 |
| Cost | OSS-first 集成 + 自托管在 commodity Kunpeng/Ascend 硬件，替代按量计费的专有服务。 |
| Developer onboarding | 通过 `@Bean` SPI override 扩展（与 Spring Boot 一致）；quickstart 直达首个 agent run。 |
| Governance | 审计级证据 + posture-aware fail-closed 默认；**Code-as-Contract** gate 让文档与代码 lockstep、漂移即 fail。 |

---

## 3. 核心架构概念（必须先理解的 5 件事）

### 3.1 双执行模式，一套运行时

真实 Agent 系统既需要**确定性工作流**（步骤按可评审的固定顺序执行），又需要**开放式推理**（循环自行决定下一步）。本平台用一个 `Run.mode` 同时把两者作为一等公民：

- **`GRAPH`** —— 确定性状态机。
- **`AGENT_LOOP`** —— ReAct 风格的 LLM 推理循环。

两者共享同一个中断原语 **`SuspendSignal`**：当一个 Run 需要等待（调子 Run、或把能力交还客户端）时，抛出 `SuspendSignal`；`Orchestrator` 给父 Run 打 checkpoint、执行子 Run、再带结果恢复父 Run。因为原语共享，**graph 节点可以调用 agent loop，agent loop 又可以调用另一个 graph——任意双向嵌套，全程一条 `Run` 血缘线**。

### 3.2 Engine Contract —— 基座规范

每个 Run 都经 `EngineRegistry.resolve(envelope)` 按 `docs/contracts/engine-envelope.v1.yaml` 派发；在 registry 之外对 `ExecutorDefinition` 子类做模式匹配是**被禁止的**（Rule R-M.a）。Envelope 与框架无关——加一个新引擎类型 = 写一个 `ExecutorAdapter` 注册到该类型，而非 patch 核心。

- 严格匹配（R-M.b）：声明 `engine_type=X` 的 Run 只能经 X 的 adapter 执行；不匹配抛 `EngineMatchingException` 并把 Run 置 FAILED。
- 跨切面策略（cost / identity / observability / sandbox routing / model gateway / checkpoint / safety guardrails）都表达为 `RuntimeMiddleware`，监听规范化的 `HookPoint` 事件（`docs/contracts/engine-hooks.v1.yaml`），无论引擎是 LangGraph 风格还是 ReAct 风格策略都一致（R-M.c）。
- S2C 回调（server→client 能力调用，用于客户端侧数据访问）经 `S2cCallbackEnvelope` + `S2cCallbackTransport` SPI（R-M.d）。

来源：[docs/architect.md](../../architect.md)、`docs/contracts/engine-envelope.v1.yaml`、`docs/contracts/engine-hooks.v1.yaml`。

### 3.3 八个 Maven 模块 × 五个部署面（plane）

不同运行特征的工作负载**不得共享基础设施**（饱和的 ML 任务不能饿死延迟敏感的 HTTP 边）。每个模块被钉死在 5 个部署面之一：

| Module | Plane | 职责 | 重构成熟度（Java 文件数） |
|---|---|---|---|
| `agent-client` | Edge Access | 客户端 SDK 边（skeleton，W3+） | java=3（骨架） |
| `agent-service` | Compute & Control | **HTTP/协议边 + 认知运行时内核**：access / session / queue / taskcontrol / engine 五层 | java=142（最完整） |
| `agent-execution-engine` | Compute & Control | 引擎 adapter SPI + `EngineRegistry`/`EngineEnvelope` + `InProcessEnginePort` | java=30（活跃） |
| `agent-middleware` | Compute & Control | `RuntimeMiddleware` SPI + hook dispatch | java=85（活跃） |
| `agent-bus` | Bus & State Hub | 跨面控制面（C2S ingress / S2C callback）+ 中立 orchestration/engine SPI | java=29（活跃 SPI） |
| `agent-evolve` | Evolution | ML/自进化流水线（skeleton；原 Python，现 Java 适配） | java=5（骨架） |
| `spring-ai-ascend-dependencies` | （build-time） | BoM | 无代码 |
| `spring-ai-ascend-graphmemory-starter` | Bus & State Hub | graph-memory 自动配置 starter | 已发布 |

> **重构状态结论**：6 个核心模块**已全部 Java 化，无残留 .py / .ts / .go**。`agent-service` 是本轮重构最完整、可作为整套架构理解入口的模块；其余多为 SPI 或骨架，按 wave 渐进落地。

总线面（Bus & State Hub）的跨服务流量再切成**三条物理隔离通道**：`control`（PAUSE/KILL 意图，**永不阻塞**）、`data`（run 载荷）、`rhythm`（心跳）——保证过载的 data 路径永远不能延迟一个 kill 信号。

来源：[README.md](../../../README.md) §Architecture at a glance、[architecture/docs/L0/ARCHITECTURE.md](../../../architecture/docs/L0/ARCHITECTURE.md) §2。

### 3.4 SPI 扩展，而非 patch

一切可插拔的都是 SPI：memory（`GraphMemoryRepository`）、run 持久化（`RunRepository`）、resilience（`ResilienceContract`）、引擎 adapter、runtime middleware / hook。你实现接口 + 用 `@Bean` 装配；平台从不要求你改它的内部。本地开发提供 in-memory 参考实现，生产用 `@Bean` override（Postgres、graph store、model gateway、未来 Ascend-served 模型）。这是平台核心原则 **P-A（Business / Platform Decoupling）**。

### 3.5 企业级治理

- **存储引擎级多租户**：每个 Run 带 tenant id，隔离在持久化层强制，而非仅应用层。
- **审计级**：durable idempotency、结构化审计日志、HTTP 边 W3C trace 传播。
- **Posture-aware**：`dev` 宽松快速迭代；`research`/`prod` 在启动时若缺必需配置则 fail-closed。
- **Code-as-Contract**：治理 gate 让文档与代码 lockstep，漂移即 fail。

来源：[docs/overview.md](../../overview.md)、[CLAUDE.md](../../../CLAUDE.md)。

---

## 4. 智能体端到端流程（基于 agent-service 代码实证）

下图是一次用户请求在重构后系统里的真实流转（取自 `agent-service` 实际代码，非旧文档蓝本）：

```text
[入站]
  A2A JSON-RPC (POST /a2a/)  ──┐
  或 async ingress 信封        ├─► A2aEnvelope / AsyncEnvelope
                               │
  AccessGateway.submitA2a/submitAsync
       └─► AgentRequest + ReplyContext（归一）
            └─► TaskHandler.run(request, reply)            ← access→taskcontrol 入站 seam
                 └─► TaskControlService
                      ├─ 幂等去重（tenant+session+task+agent+action+idempotencyKey）
                      ├─ createTask / selectTarget（Task 状态机：CREATED）
                      ├─ session 级锁
                      └─► EngineDispatchApi.enqueueExecution / enqueueResume / enqueueCancel
                           └─► EngineCommandEventFactory → EngineCommandEvent
                                └─► EngineQueueGateway.publish → InternalEventQueue
                                     └─► EngineCommandSubscriber.onCommand
                                          └─► EngineDispatcher.dispatch
                                               ├─ AgentHandlerRegistry.findByAgentId(scope.agentId)
                                               └─► AgentHandler.execute(ctx) : Stream<EngineExecutionEvent>
                                                    └─► OpenJiuwenAgentHandler
                                                         └─► OpenJiuwenAgentFactory / MessageConverter
                                                              └─► openjiuwen/agent-core-java Agent
                                                                   (LlmAgent / WorkflowAgent / ReActAgent)

[回流：两条独立链路]
  状态回写 ─► TaskControlClient.markRunning / markWaiting / markSucceeded / markFailed / markCancelled
  用户输出 ─► AccessLayerClient.appendOutput / completeOutput / failOutput / requestUserInput
                 └─► NotificationPort.notify(NotificationFrame)
                      └─► EgressDispatcher → EgressAdapter
                           ├─ A2A：TaskStatus / Message / Artifact（普通 JSON / SSE 流式 / push notification）
                           └─ async：reply topic / queue
```

关键语义：

- **入队即返回**：`POST /a2a/` 立即返回 accepted（`AccessAcceptedResponse` 带 `taskId`），不 hold 连接等执行完成；执行异步进行，输出通过 egress 回推。
- **状态与输出分离**：engine 产出的事件被路由成两条独立链路——一条回 task-control 改状态，一条回 access 层推用户可见内容（映射见交付件 2 §状态映射表）。
- **Agent 调 Agent**：当前实现仅 INLINE 模式进入闭环（openJiuwen `AbilityManager` 内部解析目标 agent）；CHILD_TASK 仅保留事件模型未实现。
- **中断/恢复**：`EngineInterruptedEvent(HUMAN_INPUT / APPROVAL / WAITING_CHILD_AGENT)` → task 置 WAITING + 向用户 `requestUserInput`；`enqueueResume` 带人工输入/审批结果/子 agent 结果恢复。

---

## 5. 关键事实校正（重构带来的「文档 vs 代码」分叉）⚠️

这是理解本项目当前状态最重要的一点，直接影响测试范围判定：

| 维度 | 平台主干文档蓝本（W0/W1） | 重构后 agent-service 实际代码 |
|---|---|---|
| 对外 API | `POST /v1/runs`、`GET /v1/runs/{id}`、`POST /v1/runs/{id}/cancel`（REST + TaskCursor），见 `openapi-v1.yaml` / `http-api-contracts.md` | **A2A JSON-RPC**：`POST /a2a/`（SendMessage / SendStreamingMessage / GetTask / CancelTask）+ `GET /.well-known/agent-card.json` + async ingress |
| 核心实体 | `Run` / `RunStateMachine` / `RunStatus` DFA | `Task` / `TaskState` DFA + `Session` 聚合 |
| 内部分层 | `service.platform.*`（HTTP/JWT/幂等/租户）+ `service.runtime.*`（orchestration SPI + Run 生命周期 + reference executors） | `access` / `session` / `queue` / `taskcontrol` / `engine` 五层 |
| 引擎 | in-memory 参考 executors（`SyncOrchestrator` 等） | `EngineDispatcher` + `OpenJiuwenAgentHandler` 驱动 `openjiuwen/agent-core-java` |

> 结论：`architecture/docs/L0/ARCHITECTURE.md`、`docs/contracts/openapi-v1.yaml`、`docs/dfx/agent-service.yaml` 仍描述较早的「Run 内核 + `/v1/runs`」形态；而重构后的代码已转向「A2A 边 + Task/Session/Queue/Engine 五层 + OpenJiuwen 适配」。**测试方案必须以代码实证为准，并把这一分叉列为高优先级回归风险区**（详见交付件 4 §4.1 / §4.2）。

---

## 6. 阅读延伸

- agent-service 模块逐层详细解读 → [02-agent-service-deep-dive.md](02-agent-service-deep-dive.md)
- L0 / L1（及三套 L 编号语义）定义 → [03-l0-l1-architecture.md](03-l0-l1-architecture.md)
- Java 重构测试方案（功能清单 + 接口文档 + 测试基线）→ [04-java-refactor-test-plan.md](04-java-refactor-test-plan.md)
- 权威架构入口：[architecture/workspace.dsl](../../../architecture/workspace.dsl) → [architecture/docs/L0/ARCHITECTURE.md](../../../architecture/docs/L0/ARCHITECTURE.md) → [architecture/docs/L1/README.md](../../../architecture/docs/L1/README.md)
