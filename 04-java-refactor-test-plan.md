# 交付件 4 · Java 重构测试方案设计

> 范围：为重构后的 `spring-ai-ascend`（重点 `agent-service`）设计系统性测试方案。
> 形态：**方案设计文档**（功能清单 + 接口文档 + 测试基线，以策略与矩阵呈现，**不含可执行测试代码**）。
> 原则：**复用仓库已有资产，不重造**；基线值全部引自现有 YAML 并标注其 `status`（`shipped` / `design_only` / `schema_shipped` / `stub`），避免把「设计目标」当「已实现」。

---

## 0. 测试方案总览

三大产出对应背景要求：

1. **§1 功能特性清单** —— 识别测试范围与重点。
2. **§2 接口文档** —— 支撑接口自动化测试（功能 / 性能 / 可靠 / 兼容）。
3. **§3 测试基线** —— 七维度：性能、可靠、安全（权限 + 沙箱）、兼容、资料、部署&升级&迁移、观测指标。

四层测试策略（复用 [docs/architecture/l0/09-verification/test-strategy.md](../../architecture/l0/09-verification/test-strategy.md)）：

| Layer | 目标 | 输入 | 产出 |
|---|---|---|---|
| L1 Contract | 模块交互语义 | ICD + machine-readable YAML | contract tests / mocks / stubs |
| L2 Business Activity | 业务活动能否串起模块/能力/状态/契约/观测 | BA-* 场景 + Capability Map | E2E BA harness + golden trace |
| L3 Technical Sub-scenario | 跨模块机制是否成立 | `technical/S*.md` + State Matrix | scenario / state-machine / failure injection |
| L4 Architecture | 架构不变量 + 依赖方向 | Invariants + Module Cards | ArchUnit static checks |

测试诚实规则（同源）：不用 mock 掩盖 contract 缺失；不用 happy-path 证明 failure semantics；不用字段 snapshot 替代 ICD 语义测试；`design_only` 契约可有 harness draft，但不得声明 runtime enforced。

---

## 1. Java 版本全量功能特性清单

> 列法：按 agent-service 五内部层 + 跨模块依赖枚举功能点；每条标注 **可见性**（外部协议 / 运维 / 内部 SPI）、**现状**（shipped / partial / stub / design_only）、**关键类**、**建议测试层**。
> 现状判定依据：代码实证 + `docs/governance/architecture-status.yaml`（shipped 真值）+ 交付件 2 §10 缺口表。

### 1.1 access 层（L1）

| # | 功能点 | 可见性 | 现状 | 关键类 | 测试层 |
|---|---|---|---|---|---|
| A1 | A2A `SendMessage`（普通 JSON） | 外部协议 | shipped | `A2aJsonRpcController`/`A2aIngressAdapter` | L1/L2 |
| A2 | A2A `SendStreamingMessage`（SSE） | 外部协议 | shipped | 同上 + `A2aEgressAdapter` | L2/L3 |
| A3 | A2A push notification（callback POST） | 外部协议 | shipped | `DefaultA2aOutputSink` | L2/L3 |
| A4 | A2A `GetTask`（聚合为 SDK Task） | 外部协议 | shipped | `A2aTaskMapper`/`A2aOutputRegistry` | L1/L2 |
| A5 | A2A `CancelTask` | 外部协议 | shipped | `AccessGateway.cancelA2a` | L2/L3 |
| A6 | Agent Card 发现 | 外部协议 | shipped | `A2aWellKnownAgentCardController` | L1 |
| A7 | async ingress 信封消费 | 外部协议 | partial（端口在，未绑定 MQ） | `AsyncIngressPort`/`AsyncIngressAdapter` | L1/L3 |
| A8 | 入站归一（→ `AgentRequest`+`ReplyContext`） | 内部 | shipped | `AccessGateway` | L1 |
| A9 | 出站分发（队列→A2A/async 适配器） | 内部 | shipped | `EgressDispatcher`/`EgressQueueRegistry` | L1/L3 |
| A10 | 通知端口 `notify(NotificationFrame)` | 内部 SPI | shipped | `NotificationPort`/`DefaultNotificationPort` | L1 |

### 1.2 session 层（L2）

| # | 功能点 | 可见性 | 现状 | 关键类 | 测试层 |
|---|---|---|---|---|---|
| S1 | `loadOrCreate` / `get` / `exists` / `delete` | 内部 SPI | shipped | `SessionManager`/`SessionManagerImpl` | L1 |
| S2 | `appendMessage` / `putState` / `removeState` / `putMetadata` | 内部 SPI | shipped | 同上 | L1 |
| S3 | `appendTask`（Task 占位） | 内部 SPI | shipped | 同上 | L1 |
| S4 | InMemory 存储 | 内部 | shipped | `InMemorySessionStore` | L1 |
| S5 | Redis 存储 + TTL + CAS 版本保护 | 内部 | partial（依赖 `RedisSessionCommands` 适配） | `RedisSessionStore`/`JacksonSessionCodec` | L2(Testcontainers) |
| S6 | 存储工厂选择（memory/redis），redis 缺客户端必须失败 | 内部 | shipped | `DefaultSessionStoreFactory` | L1/L2 |
| S7 | 乐观锁 version 递增 / `saveIfVersion` | 内部 | shipped | `SessionStore` 实现 | L1/L3(并发) |

### 1.3 queue 层（L3）

| # | 功能点 | 可见性 | 现状 | 关键类 | 测试层 |
|---|---|---|---|---|---|
| Q1 | 按 session 注册/查找队列 | 内部 | shipped | `QueueManager` | L1 |
| Q2 | 创建内存 session 队列 | 内部 | shipped | `QueueFactory`/`InMemoryInternalEventQueue` | L1 |
| Q3 | `offer` / `snapshot` / `find` | 内部 | shipped | `InternalEventQueue` | L1 |
| Q4 | 生产队列（Kafka/RocketMQ/Redis Streams）+ 至少一次投递 | 内部 | design_only | — | （设计保留） |

### 1.4 taskcontrol 层（L4）★ 测试重点

| # | 功能点 | 可见性 | 现状 | 关键类 | 测试层 |
|---|---|---|---|---|---|
| T1 | 提交执行 `run` | 内部 SPI | shipped | `TaskControlService.run` | L1/L2 |
| T2 | 恢复 `resume`（仅命中 WAITING） | 内部 SPI | shipped | `.resume` | L1/L3 |
| T3 | 取消 `cancel`（terminal 拒绝） | 内部 SPI | shipped | `.cancel` | L1/L3 |
| T4 | Task 状态机 DFA（`allowed()`） | 内部 | shipped | `TaskControlService.allowed` / `TaskState` | **L3 强测试** |
| T5 | 状态回写 `markRunning/Waiting/Succeeded/Failed/Cancelled` | 内部 SPI | shipped | 实现 `TaskControlClient` | L1/L2 |
| T6 | 幂等去重（6 元组 key） | 内部 | shipped | `idempotencyResults` map | **L3 强测试** |
| T7 | session 级并发锁 | 内部 | shipped | `sessionLocks` | L3(并发) |
| T8 | revision 乐观锁 + stale 拒绝 | 内部 | shipped | `Task.transitionTo` / `mark` | L1/L3 |
| T9 | engine 派发失败 → Task FAILED 回写 | 内部 | shipped | `failDispatch` | L2/L3 |

### 1.5 engine 层（L5）★ 测试重点

| # | 功能点 | 可见性 | 现状 | 关键类 | 测试层 |
|---|---|---|---|---|---|
| E1 | 异步入队 API（execute/resume/cancel） | 内部 API | shipped | `EngineDispatchApi`/`DefaultEngineDispatchApi` | L1/L2 |
| E2 | command 事件构造 | 内部 | partial（设计稿 stub，需核实） | `EngineCommandEventFactory` | L1 |
| E3 | 队列发布/订阅 | 内部 | shipped | `EngineQueueGateway`/`EngineCommandSubscriber` | L1/L3 |
| E4 | 派发 + handler 查找 | 内部 | shipped | `EngineDispatcher`/`AgentHandlerRegistry` | L2/L3 |
| E5 | agentId 找不到 → FAILED 回写 | 内部 | shipped | `EngineDispatcher` | L3(failure) |
| E6 | `AgentHandler` SPI 执行 | 内部 SPI | shipped | `AgentHandler` | L1 |
| E7 | OpenJiuwen 适配（建/转/映射） | 内部 | partial/stub（Factory/Converter 返回 null，需核实） | `OpenJiuwen*` 4 类 | L1/L2 |
| E8 | 9 类执行事件 → 双链路映射 | 内部 | shipped | `EngineExecutionEvent` 子类 | L2/L3 |
| E9 | 中断/恢复（HUMAN_INPUT/APPROVAL/WAITING_CHILD_AGENT） | 内部 | shipped(事件) | `EngineInterruptedEvent` | L3 |
| E10 | Agent 调 Agent INLINE | 内部 | shipped(经 openJiuwen AbilityManager) | `OpenJiuwenAgentHandler` | L2 |
| E11 | Agent 调 Agent CHILD_TASK | 内部 | design_only（仅事件模型） | `EngineAgentCallEvent` | （设计保留） |

### 1.6 跨模块依赖功能点

| # | 功能点 | 模块 | 现状 | 关键类/契约 | 测试层 |
|---|---|---|---|---|---|
| X1 | Engine Contract envelope 派发（registry-only） | agent-execution-engine | shipped | `EngineRegistry`/`engine-envelope.v1.yaml` | L1/L4 |
| X2 | 每引擎声明 hook surface | engine + middleware | shipped | `EveryEngineDeclaresHookSurfaceTest` | L4 |
| X3 | RuntimeMiddleware + HookPoint dispatch | agent-middleware | shipped(SPI) | `engine-hooks.v1.yaml` | L1/L4 |
| X4 | `SuspendSignal` 中断原语 | agent-bus(`bus.spi.engine`) | shipped | `SuspendSignalTest` | L3 |
| X5 | S2C 回调 | agent-bus(`bus.spi.s2c`) | design/partial | `s2c-callback.v1.yaml` | L1 |
| X6 | IngressGateway（C2S） | agent-bus(`bus.spi.ingress`) | shipped(SPI) | `IngressGateway` | L1 |
| X7 | 运维健康/指标 | agent-service | shipped | `/v1/health`、`/actuator/*` | L1/L2 |

### 1.7 测试重点与回归风险区（结论）

1. **高优先级（核心闭环 + 状态正确性）**：T4/T6/T8（Task 状态机、幂等、乐观锁）、E4/E5/E8（派发与事件映射）、A1–A5（A2A 三种回消息模式）、端到端闭环（`AgentServiceEndToEndIT`）。
2. **高优先级回归风险区**：「A2A 边（代码）vs `/v1/runs`（`openapi-v1.yaml`/主干文档）」分叉——契约测试必须明确以谁为准（见 §2.4）。
3. **隔离测试（依赖 stub）**：E2/E7（OpenJiuwen 真实执行未必通）——用 `FakeInterruptingAgentHandler`/`RecordingAccessLayerClient`/`RecordingTaskControlClient` 隔离。
4. **设计保留（暂不纳入功能测试）**：Q4、E11（CHILD_TASK）、S2C 运行时强制。

---

## 2. 接口文档（接口自动化测试基础）

> 测试维度：功能（正确性）/ 性能（延迟吞吐）/ 可靠（失败语义、幂等、并发）/ 兼容（契约快照、SDK/平台版本）。

### 2.1 对外协议接口 —— A2A JSON-RPC（实际 shipped 边）

公共地址：`POST http://<host>:8080/a2a/`。Header：`Content-Type: application/json`；`Accept: application/json`（同步）或 `text/event-stream`（SSE）。

| method | SDK request | 关键 params | 响应 | 内部映射 |
|---|---|---|---|---|
| `SendMessage` | `SendMessageRequest` | `tenant`、`message{role,messageId,contextId,parts[].text,metadata{tenantId,userId,agentId,sessionId,idempotencyKey,correlationId}}` | `SendMessageResponse`（含 `taskId`） | → `AccessOperation.SUBMIT` |
| `SendStreamingMessage` | `SendStreamingMessageRequest` | 同上 | SSE，每帧 `SendStreamingMessageResponse` | → SUBMIT + stream egress |
| `SendMessage` + push | `SendMessageRequest` | `configuration.taskPushNotificationConfig{url,id,token,authentication{scheme,credentials}}` | 立即 accepted；后续事件 POST 到 callback URL（带 `X-A2A-Notification-Token`） | → push egress binding |
| `GetTask` | `GetTaskRequest` | `id`/`taskId` + `metadata{tenantId,sessionId}` | `GetTaskResponse`（聚合 status/history/artifacts） | 查询 output registry |
| `CancelTask` | `CancelTaskRequest` | `id`/`taskId` + `metadata{tenantId,userId,agentId,sessionId}` | `CancelTaskResponse` | → `AccessOperation.CANCEL` |

字段契约完整表（含必填/类型/允许值/内部映射）见 [access-layer L1 设计稿 §5.1](../../architecture/l1/2026-05-30-l1--agent-service-access-layer-design.md)（已含 SendMessage / push / GetTask / CancelTask 的字段表与示例 JSON，可直接作为自动化测试用例数据源）。

**自动化测试要点**：
- 功能：5 个 method × 必填字段缺失/非法值 → 错误语义；`SendMessage` 返回 `result.role=ROLE_AGENT` + `taskId`。
- 性能：SSE 首帧延迟、push callback 投递延迟（对接 §3.1 SLO）。
- 可靠：`idempotencyKey` 重复请求只产生一个 Task；`CancelTask` 对 terminal 任务的幂等行为。
- 兼容：A2A SDK 版本 `org.a2aproject.sdk:a2a-java-sdk-server-common:1.0.0.CR1` 的 request/response 类型校验。

### 2.2 async ingress 信封

`AsyncEnvelope{headers{tenantId,userId,agentId,sessionId,operation,idempotencyKey,correlationId,replyTopic}, body{query,payload}}`。第一版建议外部生产者只用 `SUBMIT` / `CANCEL`。字段表见同设计稿 §8.1。

### 2.3 运维接口

| HTTP | 路径 | 说明 | Header 豁免 |
|---|---|---|---|
| `GET` | `/v1/health` | liveness；`{status,sha,db_ping_ns,ts}` | 是（所有 posture 免认证） |
| `GET` | `/actuator/health` `/health/liveness` `/health/readiness` | Spring Boot Actuator | 是 |
| `GET` | `/actuator/prometheus` | Prometheus 抓取 | 是（research/prod 应限内网） |

Header 约定（`http-api-contracts.md`）：`X-Tenant-Id`（UUID，可变路由必填）、`Idempotency-Key`（UUID，POST/PUT/PATCH 必填）；错误统一 `ContractError{code,message,traceId}`。状态码语义：400 头/体非法、401 缺/坏 JWT、403 租户/JWT 交叉校验失败、404 不存在或跨租户、409 幂等冲突、429 限流、503 依赖不可用。

### 2.4 ⚠️ 契约「以何为准」建议（关键）

`docs/contracts/openapi-v1.yaml` + `http-api-contracts.md` 描述的是 `POST /v1/runs`、`GET /v1/runs/{id}`、`POST /v1/runs/{id}/cancel`（Run + TaskCursor 形态），但**重构后 agent-service 实际未暴露 `/v1/runs`，而是 A2A 边**。

建议：
- **功能/接口自动化测试以 A2A 边（代码实证）为准**，把 §2.1/§2.2 作为权威接口文档。
- 把 `openapi-v1.yaml` 的 `/v1/runs` 契约标为「主干设计契约，当前代码未实现」，作为**契约漂移用例**（验证「文档声称 shipped 但代码无对应路由」），并推动后续在 `architecture-status.yaml` 校正 shipped 真值。
- `/v1/health` + `/actuator/*` 两者一致，正常纳入运维测试。

### 2.5 内部 SPI / API 接口（用于模块级契约测试）

| 接口 | 方向 | 关键方法 | 契约要点 |
|---|---|---|---|
| `EngineDispatchApi` | task-control → engine（API） | `enqueueExecution/Resume/Cancel` | 只入队；返回 `EnqueueEngineStatus{SUCCESS,FAILED}`；不传 handler/框架类型 |
| `TaskControlClient`(engine.port) | engine → task-control（回写） | `markRunning/Waiting/Succeeded/Failed/Cancelled` | 状态意图回写，真实状态机在 task-control |
| `AccessLayerClient`(engine.port) | engine → access（输出） | `appendOutput/completeOutput/failOutput/requestUserInput` | engine 不管 WS/SSE/A2A 连接 |
| `TaskControlClient`(taskcontrol.api) | access → task-control | `run/resume/cancel/mark*` | `CompletionStage<TaskResult>` |
| `SessionManager` | 任意层 → session | 见 §1.2 | 只经此入口访问 Session |
| `SessionStore` | session 内部 | `find/save/update/saveIfVersion/remove` | 版本保护，不静默覆盖 |
| `NotificationPort` | 下游 → access | `notify(NotificationFrame)` | 按 tenant+session+task 入队 |
| `AgentHandler` | engine SPI | `agentId/isHealthy/execute` | 返回 `Stream<EngineExecutionEvent>` |
| `EgressAdapter` | access 出站 | `channel/deliver` | A2A / ASYNC |

---

## 3. 测试基线（七维度）

> 每维给出：**基线 / 来源文件 / 验证手段 / 现状 / 优先级**。基线值多为 `design_only`/`schema_shipped`——**测试时区分「目标值校验」与「已实现强制」**。

### 3.1 性能基线（Performance）

来源：[docs/architecture/l0/cross-cutting/v1.0-perf-baselines.yaml](../../architecture/l0/cross-cutting/v1.0-perf-baselines.yaml)（`status: design_only`，`instrumentation_status: not_yet_wired`）+ [perf/](../../../perf/)（JMH、`baseline-2026-05-10.md`、2× 回归策略）。参考硬件：Kunpeng 920 ARM64 4 核 / 8GiB + Ascend 310 / openEuler 22.03 / BiSheng JDK 21 / ZGC。

| 指标 | 目标基线 | 验证手段 | 现状 | 优先级 |
|---|---|---|---|---|
| Run/Task admission（入口入队） | p50≤100ms / p95≤500ms / p99≤1000ms | `PerfBaselineRegressionIT`（设计中）抓 `/actuator/prometheus` 比对；`springai_ascend_runs_admit_seconds` | design_only | 高 |
| tool-call internal_api | p95≤100ms | 同上 `springai_ascend_tool_call_seconds`（label skill_class） | design_only | 中 |
| tool-call external_api | p95≤1000ms | 同上 | design_only | 中 |
| model invocation（ascend_local / pangu / openai 透传） | first-token p95 2000/1500ms；gateway overhead p95 100ms | `springai_ascend_model_invoke_seconds`（label model_id） | design_only | 中 |
| 吞吐 | 100 并发 Run/pod；20 持续 / 50 突发 admissions/s | 压测 + 并发 Run 计数（status∈ADMITTED/RUNNING/SUSPENDED_*） | design_only | 高 |
| 租户隔离开销（Postgres RLS） | p95 延迟开销 ≤5%；CPU ≤3% | RLS on/off 对比基准（1M 行分区表） | design_only | 中 |
| 单 agent 常驻内存 | p95≤512MB / p99≤768MB | JVM 内存采样 | design_only | 中 |
| 回归策略 | 任一行 p95 超 2× baseline 即 fail | `perf/README.md` 2× 规则 + JMH | partial | 高 |

> 测试方案要点：先把 `springai_ascend_*` 计时器仪表化（promotion_trigger 所列前置），再让回归 IT 加载本 YAML 在固定流量 profile 后比对 p95。当前阶段先做「目标值 documented + 压测脚手架」，不声称 runtime_enforced。

### 3.2 可靠性基线（Reliability）

来源：代码（状态机/幂等/并发）+ [docs/dfx/agent-service.yaml](../../dfx/agent-service.yaml)（`resilience`/`availability` 段）+ Resilience4j + Temporal SDK。

| 项 | 基线 | 验证手段 | 现状 | 优先级 |
|---|---|---|---|---|
| Task 状态机 DFA | 非法转移拒绝；terminal 不可转移；同态幂等 | `TaskControlServiceWhiteboxTest` + 穷举转移矩阵 | shipped | 高 |
| 幂等去重 | 同 6 元组 key 只执行一次；并发重复返回既有结果 | 并发提交同 key 断言单次副作用 | shipped | 高 |
| 取消语义 | 非 terminal→CANCELLING→CANCELLED；terminal 拒绝；engine reject→FAILED | `EngineClosedLoopIntegrationTest` + failure injection | shipped | 高 |
| 中断恢复（SuspendSignal/resume） | WAITING→resume→RUNNING；带人工/审批/子结果 | `FakeInterruptingAgentHandler` 驱动 | shipped | 高 |
| engine 派发失败回写 | `EnqueueEngineStatus.FAILED` → Task FAILED | `failDispatch` 路径断言 | shipped | 高 |
| 启动失败模式（posture） | research/prod 缺必需配置→`IllegalStateException` | Spring Boot Test 不同 `APP_POSTURE` | shipped | 高 |
| readiness 信号 | W2 deferred（依赖 durable IdempotencyStore） | — | design_only | 低 |
| Resilience（信号不被 fallback 掩盖，Rule 7） | W2 deferred | — | design_only | 低 |

### 3.3 安全基线（Security：权限 + 沙箱）

来源：[docs/dfx/agent-service.yaml#vulnerability](../../dfx/agent-service.yaml) + [docs/governance/sandbox-policies.yaml](../../governance/sandbox-policies.yaml)（`status: schema_shipped`）+ [posture-model.md](../../architecture/l0/cross-cutting/posture-model.md) + ArchUnit。

**权限 / 认证 / 隔离：**

| 项 | 基线 | 验证手段 | 现状 | 优先级 |
|---|---|---|---|---|
| JWT 校验 | RS256 + iss/aud/exp/nbf/clock-skew | spring-security-test 注入合法/过期/错 aud token | shipped(W1) | 高 |
| 租户交叉校验 | `X-Tenant-Id` ↔ JWT `tenant_id` 不匹配→403 | filter chain 集成测试 | shipped(W1) | 高 |
| 租户数据隔离 | 跨租户资源表现为 404（无存在 oracle）；RunContext.tenantId 唯一载体 | 跨租户访问测试 + ArchUnit（runtime 不得 import platform） | shipped | 高 |
| Postgres RLS | tenant_id 谓词在存储层 | `deploy/.../postgres-rls-init.yaml` + 注入测试 | design_only(W2) | 中 |
| secret 卫生 | 源码无 AKIA/sk-/BEGIN PRIVATE KEY；日志无 raw JWT/body | gate `no_secret_patterns`(E20) + 日志审查 | shipped | 高 |
| SPI 纯净度 | `*.spi.*` 只 import java.* | ArchUnit `SpiPurityGeneralizedArchTest` | shipped | 中 |

**沙箱（Sandbox Permission Subsumption，Rule 42/R-L）：**

| 项 | 基线 | 验证手段 | 现状 | 优先级 |
|---|---|---|---|---|
| 默认 deny | 未声明 skill：零出网/零 FS，CPU 100m/Mem 128M/30s/seccomp | 校验 `default_policy` 六键齐备 | schema_shipped | 高 |
| `financial_default` | 出网默认 deny + 白名单；FS 写限 scratch；CPU 2vCPU/Mem 2GiB/60s；`pii_egress_to_model:false` | 配置加载校验 + 越权配置应被拒 | schema_shipped | 高 |
| per-skill（model-call/memory-write） | 出网白名单（api.openai/anthropic、pangu、127.0.0.1:5432/6379）+ 资源上限 | 白名单匹配测试 | schema_shipped | 中 |
| 逻辑授权 ≤ 物理沙箱 | `SandboxExecutor` 拒绝比物理限制更宽的逻辑授权 | （W2 落地后）subsumption_check | design_only(W2) | 中 |

> 注意：沙箱 runtime 强制（`SandboxExecutor`）是 W2 deferred；当前 `sandbox-policies.yaml` 只是 schema + default。测试当前阶段验证「策略 schema 良构 + 配置加载 + 越权配置识别」，不声称 runtime 强制。

### 3.4 兼容性基线（Compatibility）

| 项 | 基线 | 验证手段 | 现状 | 优先级 |
|---|---|---|---|---|
| OpenAPI 快照漂移 | live vs `openapi-v1.yaml` 不漂移（E36） | `OpenApiContractIT` / `ApiCompatibilityTest` | shipped（但见 §2.4 分叉） | 高 |
| A2A SDK 版本 | `a2a-java-sdk-server-common:1.0.0.CR1` | request/response 类型回归 | shipped | 中 |
| 平台版本 | Spring Boot 4.0.5 / Java 21 / Kunpeng ARM64 原生 | 在 ARM64 跑全套（Rule G-7 Linux-first） | shipped | 中 |
| SPI 兼容策略 | `service.runtime.*.spi.*` L1 冻结；两 minor 弃用窗 | SemVer 检查 + ArchUnit | shipped | 中 |
| OpenJiuwen 依赖 | `agent-core-java:0.1.7`（或源码依赖） | 依赖解析 + 适配器集成 | partial | 中 |

### 3.5 资料/文档基线（Documentation）

| 项 | 基线 | 验证手段 | 现状 | 优先级 |
|---|---|---|---|---|
| Code-as-Contract gate | 文档-代码 lockstep，漂移 fail-closed | `gate/check_architecture_sync.sh`（`./mvnw -Pquality verify`） | shipped | 高 |
| DfX 必备性 | 每个 domain 模块有五维 DfX yaml | gate `dfx_yaml_present_and_wellformed` | shipped | 中 |
| 契约目录完整 | 40+ 契约 YAML + catalog | `contract-catalog.md` 校验 | shipped | 中 |
| 接口文档现状校正 | A2A 边文档化、`/v1/runs` 漂移标注 | 本交付件 §2.4 + 后续 ADR | 待补 | 高 |
| baseline_metrics 真值 | 65 §4 约束 / 64 ADR / 35 gate 规则 / 102 self-test 等计数一致 | gate self-tests | shipped | 中 |

### 3.6 部署 & 升级 & 迁移基线（Deploy / Upgrade / Migration）

来源：[deploy/middle-office-reference/](../../../deploy/middle-office-reference/)（Helm）+ Flyway + [ops/runbooks/](../../../ops/runbooks/)。

| 项 | 基线 | 验证手段 | 现状 | 优先级 |
|---|---|---|---|---|
| Helm 部署（中台参考） | `agent-service-deployment`、`agent-bus-statefulset`、`postgres-rls-init`、`sandbox-runtime-deployment`、`model-gateway-deployment`、`observability-stack`、`NETWORK-POLICY` | `helm template`/`helm lint` + kind/minikube 冒烟 | shipped(参考) | 高 |
| 资源 sizing | resources 块由 §3.1 perf 基线反推 | values.yaml 与 perf SLO 一致性核对 | design_only | 中 |
| DB 迁移 | Flyway `V1__init.sql`、`V2__idempotency_dedup.sql`，baseline-on-migrate | Testcontainers Postgres 全量迁移 + 幂等重跑 | shipped | 高 |
| 升级/回滚 | engine 回滚开关 `agent-service.engine.enabled=false`（拒新 command，保留未消费事件）；`ops/runbooks/rollback.md` | 开关切换 + runbook 演练 | shipped | 中 |
| 灾备/事件响应 | `ops/runbooks/{dr,incident-response,total-credential-loss,digest-pin}.md` | runbook 走查 | shipped(文档) | 中 |
| 部署形态 | 五层随 `agent-service.jar` 单体部署（engine 设计稿 §15.1） | 启动健康检查（§3.7） | shipped | 高 |
| sidecar | `ops/compose/sidecar-{ragflow,docling,graphmemory,mem0}.yml` | compose 起停 | shipped(可选) | 低 |

健康检查项（engine 设计稿 §15.5）：`EngineQueueGateway` 可用、`EngineCommandSubscriber` running、`AgentHandlerRegistry` ≥1 注册 agentId、`OpenJiuwenAgentHandler.isHealthy=true`、`TaskControlClient`/`AccessLayerClient` 可用。

### 3.7 观测指标基线（Observability）

来源：[docs/dfx/agent-service.yaml#observability](../../dfx/agent-service.yaml) + `ARCHITECTURE.md §0.5.3 Telemetry vertical` + `ops/helm/.../observability-stack`。

| 项 | 基线 | 验证手段 | 现状 | 优先级 |
|---|---|---|---|---|
| 指标命名空间 | `springai_ascend_*`（E18 gate 强制） | 抓 `/actuator/prometheus` 断言前缀 | shipped | 高 |
| 高基数防护 | `TenantTagMeterFilter` 剥离原始 tenant_id 等高基数标签 | 指标标签审查（E19） | partial(W4 wiring) | 中 |
| trace 传播 | W3C `traceparent` 在 HTTP 边提取/发起；出站 `traceresponse`（E41） | `TraceExtractFilter`(order 10) 集成测试 | shipped | 中 |
| 日志字段 | MDC 带 `tenant_id`/`trace_id`/`span_id`/`run_id`；失败安全决策 WARN+ | 日志结构断言（logstash JSON） | shipped | 中 |
| Hook 发射路径 | 每次 LLM/工具/生命周期边界经 HookChain（Rule 19） | `LlmGatewayHookChainOnlyTest`（design-only 守卫） | design_only(W2) | 低 |
| 审计行 | `run_state_change` 审计行（Rule 24） | — | design_only(W2) | 低 |
| OpenAPI 契约面 | 快照 pinned + 漂移检测（E25/E36） | contract IT | shipped | 中 |
| 关键失败计数 | JWT 失败 / 租户不匹配 / 幂等冲突/replay / posture boot 失败 / repo 失败 计数 | 触发后抓指标断言 | partial | 中 |

---

## 4. 落地执行建议（非破坏性）

1. **标定重构基线绿/缺口**：先跑 `./mvnw -q -pl agent-service -am test`（或 `-Pquality verify` 含 gate + IT），确认现有 31 个测试类（见交付件 2 §9）通过，并核实交付件 2 §10 的 stub（E2/E7）真实落地状态。
2. **补功能测试**：按 §1.7 优先级，先补 T4/T6/T8、E4/E5/E8、A1–A5 的缺口用例（用 `Fake*`/`Recording*` 支撑类隔离 OpenJiuwen 未通路径）。
3. **接口自动化**：以 §2.1/§2.2 A2A 字段表为数据源（REST-assured + WireMock），并新增 §2.4 的「`/v1/runs` 契约漂移」用例。
4. **基线仪表化**：按 §3.1 promotion_trigger 先落 `springai_ascend_*` 计时器，再上 `PerfBaselineRegressionIT`；其余 design_only 维度先做「目标值文档 + 脚手架」，待对应 wave 落地再转 runtime_enforced。
5. **诚实标注**：所有报告区分 `shipped` / `design_only` / `schema_shipped` / `stub`，禁止把设计目标当已实现（遵循 test-strategy「测试诚实规则」与 Rule D-4 三层测试）。

---

## 5. 阅读延伸

- 整体架构与智能体流程 → [01-architecture-summary.md](01-architecture-summary.md)
- agent-service 逐层详解（接口签名/状态机/缺口）→ [02-agent-service-deep-dive.md](02-agent-service-deep-dive.md)
- L0/L1 与三套 L 编号语义 → [03-l0-l1-architecture.md](03-l0-l1-architecture.md)
- 复用资产：[test-strategy.md](../../architecture/l0/09-verification/test-strategy.md)、[verification-matrix.md](../../architecture/l0/09-verification/verification-matrix.md)、[v1.0-perf-baselines.yaml](../../architecture/l0/cross-cutting/v1.0-perf-baselines.yaml)、[sandbox-policies.yaml](../../governance/sandbox-policies.yaml)、[dfx/agent-service.yaml](../../dfx/agent-service.yaml)、[deploy/middle-office-reference/](../../../deploy/middle-office-reference/)、[ops/runbooks/](../../../ops/runbooks/)、[gate/](../../../gate/)。
