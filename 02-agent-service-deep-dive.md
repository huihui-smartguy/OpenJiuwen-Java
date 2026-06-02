# 交付件 2 · agent-service 模块详细解读

> 范围：`agent-service`（Compute & Control 面，重构最完整的模块，142 个 Java 文件）。
> 目的：为测试范围划定与接口文档提供精确的模块内部图景。
> 约定：所有路径相对 `<ROOT> = spring-ai-ascend-main`；类名保留英文。

---

## 1. 模块定位与内部分层

`agent-service` 在重构后被组织为**五个内部层**（这是模块自身的内部 L1–L5 编号，**与平台级 L0/L1 是两回事**，见交付件 3）：

```text
agent-service（一个 Spring Boot 进程 / 一个 jar）
  L1  access            协议接入与出口（A2A / async）
  L2  session           会话管理（Session 聚合 + Store）
  L3  queue             internal-event-queue（进程内异步事件队列）
  L4  taskcontrol       task-centric-control（任务编排「大脑」）
  L5  engine            执行调度 + OpenJiuwen Agent 框架适配
  + schema              跨层共享的请求/响应/消息 POJO
  + bootstrap           入口 + 跨层 seam 装配
```

包根：`com.huawei.ascend.service`。

来源：[AgentServiceApplication.java](../../../agent-service/src/main/java/com/huawei/ascend/service/bootstrap/AgentServiceApplication.java)、三份 L1 设计稿 [access](../../architecture/l1/2026-05-30-l1--agent-service-access-layer-design.md) / [engine](../../architecture/l1/2026-05-30-l1--agent-service-engine-model-design.md) / [session-task](../../architecture/l1/2026-05-30-l1--agent-service-session-task-manager-design.md)。

---

## 2. L1 · access（协议接入层）

**职责**：接收标准 A2A JSON-RPC 调用与外部异步入站消息 → 归一为内部请求 → 调下游任务入口；并把下游产出的用户可见消息帧投递回 A2A / async 通道。

### 2.1 对外 HTTP 入口

| HTTP | 路径 | 说明 |
|---|---|---|
| `POST` | `/a2a/` | 标准 A2A JSON-RPC 单入口，按 `method` 区分：`SendMessage` / `SendStreamingMessage`(SSE) / `GetTask` / `CancelTask` |
| `GET` | `/.well-known/agent-card.json` | 标准 Agent Card 发现路径，返回 `org.a2aproject.sdk.spec.AgentCard` |

支持三种回消息模式：普通 JSON、SSE 流式（`Accept: text/event-stream`）、push notification（`params.configuration.taskPushNotificationConfig`）。

### 2.2 核心类

| 类 | 职责 |
|---|---|
| `A2aJsonRpcController` | A2A JSON-RPC 单入口控制器，读 `method` 后用 A2A SDK 类型分发 |
| `A2aWellKnownAgentCardController` | 暴露 Agent Card |
| `AccessGateway` | access 层主编排器：入站归一（`A2aEnvelope`/`AsyncEnvelope` → `AgentRequest`+`ReplyContext`）→ 调 `TaskHandler.run/cancel` |
| `TaskHandler` | access→taskcontrol 的边界接口（`run(AgentRequest, ReplyContext)` / `cancel(AccessCancelCommand)`） |
| `NotificationPort` | 暴露给下游的通知端口 `notify(NotificationFrame)` |
| `EgressDispatcher` / `EgressAdapter` / `EgressQueueRegistry` | 消费回消息队列并分发到 A2A / async 出站适配器 |
| `A2aIngressAdapter` / `A2aEgressAdapter` / `A2aTaskMapper` / `A2aOutputRegistry` | A2A 入/出站协议适配与输出聚合（`GetTask` 时聚合为 SDK `Task`） |
| `AsyncIngressPort` / `AsyncIngressAdapter` / `AsyncEgressAdapter` / `AsyncOutputSink` | 异步通道入/出站（不绑定具体 MQ，仅端口 + 适配） |

> 代码与设计稿的差异：设计稿写的是 `TaskHandler.runTask(AccessIntent)`，实际代码使用 `TaskHandler.run(AgentRequest, ReplyContext)` + `cancel(AccessCancelCommand)`，请求载体是 `schema.AgentRequest`。以代码为准。

### 2.3 出站消息帧映射

| `NotificationType` | A2A SDK 输出 |
|---|---|
| `ACK` | `TaskStatusUpdateEvent(SUBMITTED)` |
| `TOOL_RESULT` | `TaskArtifactUpdateEvent(Artifact)` |
| `LLM_RESULT` | Agent `Message`（中间帧 working，terminal 帧 completed） |
| `ERROR` | `TaskStatusUpdateEvent(FAILED)` |

来源：[access-layer 设计稿](../../architecture/l1/2026-05-30-l1--agent-service-access-layer-design.md)、[AccessGateway.java](../../../agent-service/src/main/java/com/huawei/ascend/service/access/core/AccessGateway.java)。

---

## 3. L2 · session（会话管理层）

**职责**：定义并维护 `Session` 聚合（按 `tenantId + sessionId`），维护消息历史、会话状态、Task 占位列表；通过 `SessionStore` 抽象屏蔽存储实现，预留版本号/乐观锁。

### 3.1 公共接口 `SessionManager`

`loadOrCreate / get / exists / appendMessage / putState / removeState / putMetadata / removeMetadata / appendTask / delete`。其它层**只通过 `SessionManager` 访问**，不直接碰 `SessionStore`。

`state`（业务可恢复状态：槽位、流程参数）与 `metadata`（协议来源、trace/debug、观测标签）边界必须清晰。

### 3.2 存储抽象 `SessionStore`

`find / save / update(UnaryOperator) / saveIfVersion(expectedVersion) / remove`。两套实现：

| 实现 | 场景 |
|---|---|
| `InMemorySessionStore` | 本地开发 / 单机 / 单测（重启丢失） |
| `RedisSessionStore` | 联调 / 测试 / 生产 / 多实例（跨进程共享，TTL，CAS 版本保护）|

配置经 `SessionStoreFactory` + `type: memory|redis` 选择；选 `redis` 但无可用 `RedisSessionCommands` 时**必须失败**，不得静默退回内存。`version` 创建时为 1，每次成功写入递增；Redis 用 Lua/事务/RedisJSON 做 `saveIfVersion`，`max-cas-retries`（如 16）防止无限重试。

### 3.3 关键 POJO

`Session(tenantId,userId,agentId,sessionId,version,messages,state,tasks,metadata,createdAt,updatedAt,lastAccessedAt,expiresAt)`、`SessionMessage`、`SessionContentPart`、`SessionMessageRole{USER,ASSISTANT,SYSTEM,TOOL}`、`SessionKey(tenantId,sessionId)`、`Task()`（**占位空对象**，Task 细节归 L4）。

来源：[session-task 设计稿](../../architecture/l1/2026-05-30-l1--agent-service-session-task-manager-design.md)、`agent-service/.../session/**`。

---

## 4. L3 · queue（internal-event-queue）

**职责**：进程内异步事件队列，解耦 task-control 入队与 engine 消费。

核心类：`QueueManager`（按 session 注册/查找队列）、`QueueFactory`（`inMemorySessionQueue(tenantId,sessionId,queueManager)`）、`InternalEventQueue<T>`（`offer / snapshot / find` 等）、`InMemoryInternalEventQueue`、`QueueRegistration`。

当前为**进程内内存实现**；生产可替换为 Kafka / RocketMQ / Redis Streams（engine 设计稿 §15.4），要求 EngineCommandEvent **至少一次投递**。

来源：`agent-service/.../queue/**`、[engine 设计稿 §7/§15](../../architecture/l1/2026-05-30-l1--agent-service-engine-model-design.md)。

---

## 5. L4 · taskcontrol（任务编排「大脑」）★ 测试重点

**职责**：任务生命周期与状态机、幂等去重、session 级并发控制、向 engine 派发、并实现 `TaskControlClient` 接收 engine 回写。`TaskControlService` 是核心实现类。

### 5.1 Task 状态机（DFA）

`TaskState`：`CREATED, RUNNING, WAITING, PAUSED, CANCELLING, COMPLETED, FAILED, CANCELLED`（后三者 terminal）。合法转移（`TaskControlService.allowed()`，**强测试点**）：

```text
CREATED    → RUNNING | CANCELLING | FAILED
RUNNING    → WAITING | COMPLETED | FAILED | CANCELLING | CANCELLED
WAITING    → RUNNING | FAILED | CANCELLING | CANCELLED
PAUSED     → RUNNING | FAILED | CANCELLING
CANCELLING → CANCELLED | FAILED
COMPLETED / FAILED / CANCELLED → （terminal，拒绝任何转移）
同状态自转移 → 允许（幂等）
```

`Task` 携带 `revision`（乐观锁，每次 `transitionTo` 递增）；`mark*` 命令带 `expectedRevision`，不匹配返回 "stale task revision"；非法转移返回 "transition rejected"。

### 5.2 关键能力

| 能力 | 实现要点 |
|---|---|
| 提交执行 | `run(RunCommand)` → `submit("RUN")` → `prepare`（建/定位 Task）→ `dispatch`（调 `enqueueExecution`） |
| 恢复 | `resume(ResumeCommand)` → `submit("RESUME_INPUT")`，仅命中 `WAITING` 的 Task；否则新建 |
| 取消 | `cancel(CancelCommand)` → Task 置 `CANCELLING` → `enqueueCancel`；terminal 任务拒绝取消 |
| 状态回写 | `markRunning/markWaiting/markSucceeded/markFailed/markCancelled`（实现 `TaskControlClient`） |
| 幂等去重 | `idempotencyResults: ConcurrentMap<IdempotencyKey, TaskResult>`，key=`(tenantId,sessionId,taskId,agentId,action,idempotencyKey)`，`computeIfAbsent` 保证同键只执行一次 |
| 并发控制 | `sessionLocks: ConcurrentMap<SessionKey,Object>`，按 `(tenant,session)` 加锁 |
| 失败码 | `TaskFailureCode`（如 `ENGINE_DISPATCH_REJECTED`、`AGENT_ID_INVALID`、`RUNTIME_ERROR`、`CANCELLED_BY_RUNTIME`）；`WaitingReason` 表达等待原因 |

### 5.3 与 engine 的接口

`TaskControlService` 持有 `Supplier<EngineDispatchApi>`，向其 `enqueueExecution / enqueueResume / enqueueCancel`；`EnqueueEngineStatus.FAILED` 时把 Task 置 FAILED 并回 `result(accepted=false)`。

来源：[TaskControlService.java](../../../agent-service/src/main/java/com/huawei/ascend/service/taskcontrol/TaskControlService.java)、[Task.java](../../../agent-service/src/main/java/com/huawei/ascend/service/taskcontrol/Task.java)、[TaskState.java](../../../agent-service/src/main/java/com/huawei/ascend/service/taskcontrol/TaskState.java)。

---

## 6. L5 · engine（执行调度 + OpenJiuwen 适配）★ 测试重点

**职责**：对外提供异步执行入队 API；从队列消费 command；按 `agentId` 找 handler；执行目标 Agent；把用户输出发给 access、把状态回写 task-control；支持 Agent 调 Agent（INLINE）。

### 6.1 入站 API `EngineDispatchApi`（engine 实现，task-control 调用）

```java
EnqueueEngineStatus enqueueExecution(EnqueueEngineExecutionRequest);  // scope + input
EnqueueEngineStatus enqueueResume(EnqueueEngineResumeRequest);        // scope + input
EnqueueEngineStatus enqueueCancel(EnqueueEngineCancelRequest);        // scope
```

约束：调用方只在 `EngineExecutionScope` 中带 `agentId`，**不传 handler 名 / 框架类型 / openJiuwen 内部模式**；找不到 `agentId` 时 `EngineDispatcher` 生成 `EngineFailedEvent` 并回写 task-control。

### 6.2 派发与执行

```text
EngineCommandEventFactory → EngineCommandEvent(commandType: EXECUTE|RESUME|CANCEL, scope, input)
  → EngineQueueGateway.publish → InternalEventQueue
  → EngineCommandSubscriber.onCommand → EngineDispatcher.dispatch
  → AgentHandlerRegistry.findByAgentId(scope.agentId)
  → AgentHandler.execute(AgentExecutionContext) : Stream<EngineExecutionEvent>
```

`AgentHandler` SPI：`agentId() / isHealthy() / execute(ctx)`。session / memory / checkpoint / sandbox 能力**不在 engine 抽象独立接口**，由 `OpenJiuwenAgentHandler` 内部按 openJiuwen 原生能力接入。

### 6.3 OpenJiuwen 适配器

| 类 | 职责 |
|---|---|
| `OpenJiuwenAgentHandler` | `AgentHandler` 实现：按 agentId 绑定一个可执行 Agent 定义；驱动 `openjiuwen/agent-core-java` |
| `OpenJiuwenAgentFactory` | 创建/恢复 openJiuwen Agent（LlmAgent / WorkflowAgent / ReActAgent，类型由注册配置决定） |
| `OpenJiuwenMessageConverter` | `EngineInput` ↔ openJiuwen 输入转换（第一版仅文本 + variables） |
| `OpenJiuwenResultMapper` | openJiuwen 输出 → `EngineExecutionEvent` |

openJiuwen 能力映射：session→AgentSessionApi/WorkflowSessionApi；checkpoint→Checkpointer；memory→LongTermMemory/retrieval/context engine；agent call→AbilityManager。

### 6.4 执行事件 → 双链路映射

| Engine event | task-control 动作 | access-layer 动作 |
|---|---|---|
| `EngineStartedEvent` | `markRunning` | — |
| `EngineOutputEvent` | — | `appendOutput` |
| `EngineInterruptedEvent(HUMAN_INPUT/APPROVAL)` | `markWaiting` | `requestUserInput` |
| `EngineInterruptedEvent(WAITING_CHILD_AGENT)` | `markWaiting` | — |
| `EngineCompletedEvent` | `markSucceeded` | `completeOutput` |
| `EngineFailedEvent` | `markFailed` | `failOutput` |
| `EngineCancelledEvent` | `markCancelled` | —/complete |

出站 port：`TaskControlClient`（engine→taskcontrol，`engine/port/`）、`AccessLayerClient`（engine→access，`engine/port/`）。

来源：[engine 设计稿](../../architecture/l1/2026-05-30-l1--agent-service-engine-model-design.md)、[EngineDispatchApi.java](../../../agent-service/src/main/java/com/huawei/ascend/service/engine/api/EngineDispatchApi.java)、`agent-service/.../engine/**`。

---

## 7. schema（跨层共享 POJO）

`AgentRequest`、`AgentResponse`、`Message`、`Content`、`ContentType`、`Role`、`RunStatus`。`Message.user(text)` 是常用构造。`AgentRequest(tenantId,userId,agentId,sessionId,input,idempotencyKey,metadata)`。

---

## 8. 构建、依赖与入口

### 8.1 依赖（`agent-service/pom.xml`）

- 平台侧：spring-boot-starter-**web / actuator / validation**、**data-jdbc + postgresql + flyway**、**security + oauth2-resource-server（JWT）**、**resilience4j**、**caffeine + cache**、**spring-cloud-starter-vault-config**、**springdoc-openapi**、**micrometer-registry-prometheus**、**logstash-logback-encoder**。
- 运行时侧：**spring-ai**（anthropic / openai / vector-store-pgvector）、**MCP Java SDK**、**a2a-java-sdk-server-common**、**Temporal SDK**、**Apache Tika**。
- 兄弟模块：`agent-execution-engine`、`agent-bus`、`agent-middleware`。
- 底层 Agent 框架：**`com.openjiuwen:agent-core-java:0.1.7`**（OpenJiuwen 引擎适配目标；亦可源码依赖 `third_party/openjiuwen/agent-core-java`）。
- 测试：spring-boot-starter-test（JUnit5+Mockito）、spring-security-test、temporal-testing、**Testcontainers(postgresql + junit-jupiter)**、**WireMock**、**REST-assured**、**ArchUnit**、resilience4j-circuitbreaker。
- 构建：`maven-failsafe-plugin`（`*IT.java` 集成测试）；把 `engine-envelope/engine-hooks/s2c-callback` 三个契约 YAML 复制到 classpath 供运行时自校验。

### 8.2 入口与装配

- `AgentServiceApplication`：`@SpringBootApplication(scanBasePackages = {service.access, service.bootstrap})`——**只扫 access + bootstrap**，其余四层（session/queue/taskcontrol/engine）经各自 `AutoConfiguration` 导入，避免重复注册。`main()` 启动「全五层」单 Spring 上下文。
- `AgentServiceBootstrapConfiguration`：提供 2 个跨层 seam bean：
  - `accessTaskHandler`（`AccessTaskHandler`）——入站：access → taskcontrol（`@ConditionalOnMissingBean(TaskHandler.class)`，它的存在才激活整条入站链）。
  - `accessNotificationClient`（`AccessNotificationClient`）——出站：engine → access（满足 engine 的 `@ConditionalOnBean(AccessLayerClient.class)` 守卫）。
- 各层自带 AutoConfiguration：`AccessLayerConfiguration`、`SessionManageConfiguration`、`EngineAutoConfiguration`（+`EngineProperties`）、`TaskControlAutoConfiguration`；`META-INF/spring/...AutoConfiguration.imports` 注册。

### 8.3 运行时配置（`application.yml`）

虚拟线程开启（Loom）；Hikari pool（max 20 / min idle 4）；Flyway 开启 + baseline-on-migrate；`management.endpoints` 暴露 `health,info,prometheus`；`springdoc` `/v3/api-docs` + `/swagger-ui`；`app.posture=${APP_POSTURE:dev}`；`app.idempotency.ttl=PT24H` / `allow-in-memory=false`；`app.runs.dispatch`（core 4 / max 16 / queue 256 / rejection CALLER_RUNS）。注意：Spring AI 的 OpenAI/Anthropic key 用 dummy 默认值占位以让上下文 boot（不真正调用）；Resilience4j SpringBoot3Verifier 与 Vault autoconfig 在 W0 被排除/禁用。

来源：[pom.xml](../../../agent-service/pom.xml)、[application.yml](../../../agent-service/src/main/resources/application.yml)、[AgentServiceBootstrapConfiguration.java](../../../agent-service/src/main/java/com/huawei/ascend/service/bootstrap/AgentServiceBootstrapConfiguration.java)。

---

## 9. 现有测试资产（重构基线）

| 层 | 测试类（`agent-service/src/test`） |
|---|---|
| 单元 / whitebox | `TaskControlServiceWhiteboxTest`、`QueueManagerWhiteboxTest`、`TaskflowEngineBridgeWhiteboxTest`、`EngineCommandEventFactoryTest`、`DefaultEngineDispatchApiTest`、`OpenJiuwenMessageConverterTest`、`OpenJiuwenResultMapperTest`、`EngineDispatcherTest` |
| 集成 | `EngineClosedLoopIntegrationTest`、`AgentServiceEndToEndIT`（failsafe IT，端到端闭环）|
| 引擎契约（runtime） | `EngineEnvelopeValidationTest`、`EnginePayloadDispatchOnlyViaRegistryTest`、`EngineRegistryResolveTest`、`EveryEngineDeclaresHookSurfaceTest`、`SuspendSignalTest` |
| 架构（ArchUnit） | `AudienceBExtensionSeamsArchTest`、`RunContextIdentityAccessorsTest`、`RunRepositorySaveGuardTest`、`SpanTenantAttributeRequiredTest` |
| 测试支撑 | `FakeInterruptingAgentHandler`、`RecordingAccessLayerClient`、`RecordingTaskControlClient` |

测试框架：JUnit5 + Mockito + Spring Boot Test + Spring Security Test + Testcontainers(Postgres) + WireMock + REST-assured + ArchUnit + Temporal testing。

---

## 10. 落地缺口（直接影响测试范围判定）⚠️

| 缺口 | 现状 | 测试含义 |
|---|---|---|
| `EngineCommandEventFactory.execute/resume/cancel` | 设计稿中返回 `null`（stub） | 需核对实际实现是否已补全；未补全则闭环测试会断 |
| `OpenJiuwenAgentFactory.create` / `OpenJiuwenMessageConverter.toOpenJiuwenInput` | 设计稿中返回 `null`（stub） | OpenJiuwen 真实执行路径可能尚未通；用 `FakeInterruptingAgentHandler` 隔离测试 |
| CHILD_TASK（Agent 调 Agent 子任务模式） | 仅保留事件模型，未进入执行闭环 | 不在本轮功能测试范围，标记为「设计保留」 |
| 命名差异 | 代码 `engine/api`（设计稿 `engine/spi`）、`engine/port`（设计稿 `engine/service`） | 接口文档以代码为准 |
| 对外 API 形态 | 代码=A2A 边；主干文档/`openapi-v1.yaml`=`/v1/runs` | 高优先级回归风险区（见交付件 4） |

> 建议在执行阶段跑一次 `./mvnw -q -pl agent-service -am test`（或 `-Pquality verify`）确认上述 stub 的真实落地状态，从而精确界定「已覆盖 vs 待补」。

---

## 11. 阅读延伸

- 整体架构与智能体流程 → [01-architecture-summary.md](01-architecture-summary.md)
- L0 / L1 与三套 L 编号语义 → [03-l0-l1-architecture.md](03-l0-l1-architecture.md)
- 基于本模块的测试方案 → [04-java-refactor-test-plan.md](04-java-refactor-test-plan.md)
