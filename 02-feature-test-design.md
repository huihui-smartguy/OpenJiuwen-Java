# 02 · 逐特性测试设计（怎么测、怎么测最优）

> 这是本方案最核心的一章。对**核心闭环三层 access(A) / taskcontrol(T) / engine(E)** 给出**逐特性**测试设计；对 session(S)/queue(Q)/跨模块(X) 给**精炼表**。功能点编号与 [../05-diagrams.md](../05-diagrams.md) 图 6c 一致。

## 怎么用这一章

每个功能点用统一模板：

> **[编号] 名称** — 现状(shipped/partial/stub/design_only) · 优先级(P0/P1/P2)
> - **目标**：要证明什么
> - **测试层**：L0–L4（见 [01](01-test-strategy.md)）
> - **夹具/技术**：用哪个 fake/recording/mock/内存件
> - **用例**：正常 / 边界 / 失败 / 并发·幂等（按需）
> - **断言点**：断言哪些字段/顺序
> - **模板**：复用哪个现有测试（可照抄）

优先级口径：**P0 = 核心闭环与状态正确性，必测**；P1 = 重要旁路；P2 = 边角/设计保留。

---

## 一、access 层（A1–A10）

access 的功能点多数要么用 **L3（@SpringBootTest 走 A2A）** 验证端到端，要么用 **L1（适配器单元）** 验证协议映射。E2E 用 `AgentServiceEndToEndIT` 的 `TestRuntime` 配方（详见 [05](05-e2e-and-automation.md)）。

### [A1] A2A SendMessage（普通 JSON）— shipped · P0
- **目标**：一次 `SendMessage` 被归一为 `AgentRequest` 并驱动全链路，最终回一个带 `taskId` 的 accepted，且出站有终帧。
- **测试层**：L3（端到端）。
- **夹具/技术**：`@SpringBootTest(TestRuntime)` + `EchoAgentHandler`；`A2aAccessService.send(envelope)`，轮询 `A2aOutputRegistry`。
- **用例**：
  - 正常：发 "hello world" → accepted.taskId 非空、tenantId 正确；出站含 `kind="Message"`、最后一帧 `terminal=true`、body 含回显文本。
  - 边界：`metadata.sessionId` 缺失 → 应回退用 `contextId` 或新生成 UUID（见 `AccessGateway.normalizeSessionId`）。
  - 失败：未知 `agentId` → engine 找不到 handler → 出站终帧为 error（见 A 链路 + E5）。
- **断言点**：`accepted.accepted()/taskId()/tenantId()`；`outputs.get(last).terminal()`；`outputs.anyMatch(kind=="Message")`。
- **模板**：`bootstrap/AgentServiceEndToEndIT#a2aRequestRunsThroughTheStackAndRepliesBack`（直接照抄）。

```java
@Test
void sendMessage_runsThroughStackAndReplies() {
    A2aEnvelope env = envelope("session-1", "hello world");      // 见 TestRuntime 的 envelope() 工厂
    A2aAcceptedResponse accepted = a2aAccessService.send(env);
    assertThat(accepted.accepted()).isTrue();
    A2aOutputHandle h = new A2aOutputHandle(TENANT, "session-1", accepted.taskId());
    List<A2aOutput> out = awaitOutputs(h);                        // 轮询直到 terminal
    assertThat(out.get(out.size() - 1).terminal()).isTrue();
    assertThat(out).anyMatch(o -> "Message".equals(o.kind()));
}
```

### [A2] A2A SendStreamingMessage（SSE）— shipped · P0
- **目标**：流式模式下出站多帧、首帧 accepted、中间帧 working、终帧 completed。
- **测试层**：L3（HTTP）。
- **夹具/技术**：REST-assured/MockMvc 走 `POST /a2a/`，`Accept: text/event-stream`；或 `A2aAccessService.send(envelope, streaming=true)` 后读出站序列。
- **用例**：正常（多帧、末帧 terminal）；边界（中途无输出只有完成帧）；失败（流中断/agent 抛异常 → 终帧 error）。
- **断言点**：`Content-Type` 含 `text/event-stream`；帧序列首 accepted、末 terminal。
- **模板**：A1 的 E2E + `AccessGateway.submitA2a(envelope, true)`（streaming）。

### [A3] A2A push notification（callback POST）— shipped · P1
- **目标**：带 `taskPushNotificationConfig.url` 时，终帧被 POST 到 callback URL（含鉴权 token 头）。
- **测试层**：L3。
- **夹具/技术**：WireMock 起一个 callback 桩服务，断言收到 POST 且头含 `X-A2A-Notification-Token`。
- **用例**：正常（callback 收到事件）；失败（callback 5xx → 重试/记录，按实现）。
- **断言点**：WireMock `verify(postRequestedFor(urlEqualTo("/a2a/callback")))`。
- **模板**：A1 + WireMock（见 [03](03-interface-and-cases.md) 的 WireMock 片段）。

### [A4] A2A GetTask（聚合为 SDK Task）— shipped · P1
- **目标**：按 `tenantId+sessionId+taskId` 把出站缓存聚合成 SDK `Task`（status/history/artifacts）。
- **测试层**：L3 / L1（`A2aTaskMapper` 单测）。
- **用例**：正常（已有输出→聚合非空）；边界（任务不存在→空/错误）。
- **断言点**：返回 `Task.status/history/artifacts`。
- **模板**：`A2aOutputRegistry` 写入后调 `GetTask`，或单测 `A2aTaskMapper`。

### [A5] A2A CancelTask — shipped · P0
- **目标**：`CancelTask` 转 `AccessOperation.CANCEL` 透传到 task-control，并触发取消链路。
- **测试层**：L3。
- **用例**：正常（运行中任务→CANCELLING/CANCELLED）；边界（terminal 任务→拒绝，见 T3）。
- **断言点**：任务状态进入取消；出站无新增用户输出（取消不产出）。
- **模板**：A1 E2E + `AccessGateway.cancelA2a(envelope)`。

### [A6] Agent Card 发现 — shipped · P2
- **目标**：`GET /.well-known/agent-card.json` 返回合法 `AgentCard`。
- **测试层**：L3（MockMvc）。
- **用例**：正常（200 + JSON schema 合法）。
- **断言点**：状态 200、关键字段存在。
- **模板**：MockMvc `get("/.well-known/agent-card.json")`。

### [A7] async ingress 信封消费 — partial（端口在，未绑 MQ）· P1
- **目标**：`AsyncEnvelope` 被归一为 `AgentRequest` 并驱动链路；`replyTopic` 进入出站绑定。
- **测试层**：L2/L3。
- **用例**：正常（SUBMIT）；边界（operation 缺省按 SUBMIT）；失败（CANCEL）。
- **断言点**：`AccessGateway.submitAsync(envelope)` 后任务创建、出站绑定 `targetRef=replyTopic`。
- **模板**：直接调 `AccessGateway.submitAsync(asyncEnvelope)`（无需 MQ）。

### [A8] 入站归一（→ AgentRequest + ReplyContext）— shipped · P0
- **目标**：A2A/async 各字段正确映射进 `AgentRequest`（tenant/user/agent/session/query/idempotencyKey/metadata）与 `ReplyContext`。
- **测试层**：L1（`AccessGateway` 单元）。
- **用例**：正常映射；边界（sessionId 空→fallback contextId→UUID；text 空→""）。
- **断言点**：`AgentRequest` 各字段值；`metadata` 含 parts/contextId/correlationId。
- **模板**：直接 `new AccessGateway(fakeTaskHandler)`，断言传给 `TaskHandler.run(request, reply)` 的 `request`（用一个录参的 fake `TaskHandler`）。

### [A9] 出站分发（队列→A2A/async 适配器）— shipped · P1
- **目标**：`NotificationFrame` 按 `EgressBinding` 投递到正确通道（A2A/async），终帧后清理队列。
- **测试层**：L2/L3。
- **用例**：正常（A2A 投递）；边界（队列不存在→明确异常/记录）；终帧后 `EgressQueueRegistry.find()` 为空（无泄漏）。
- **断言点**：`egressQueueRegistry.find(...)` 终态为空。
- **模板**：`AgentServiceEndToEndIT` 的 `awaitEgressCleanup` + `assertThat(find(...)).isEmpty()`。

### [A10] 通知端口 notify(NotificationFrame) — shipped · P1
- **目标**：`NotificationPort.notify(frame)` 按 `tenant+session+task` 入队；队列不存在时抛明确异常。
- **测试层**：L1/L2。
- **用例**：正常入队；失败（无绑定队列→异常）。
- **断言点**：入队成功/异常类型。
- **模板**：`DefaultNotificationPort` + `DefaultEgressQueueRegistry` 直接装配。

---

## 二、taskcontrol 层（T1–T9）★ 最高优先级

全部用 **L2 白盒**：直接 `new TaskControlService(QueueManager, RecordingEngineDispatchApi, Clock.fixed(...))`。`RecordingEngineDispatchApi` 录下 `executions/resumes/cancels` 并可设 `status=FAILED` 注入失败（见 `TaskControlServiceWhiteboxTest` 内部类）。

**通用装配骨架（照抄）：**
```java
private final RecordingEngineDispatchApi engine = new RecordingEngineDispatchApi();
private final QueueManager queueManager = new QueueManager();
private final TaskControlService service = new TaskControlService(
        queueManager, engine, Clock.fixed(Instant.parse("2026-06-01T00:00:00Z"), ZoneOffset.UTC));
// run/resume/cancel/mark 辅助方法见 TaskControlServiceWhiteboxTest
```

### [T1] 提交执行 run — shipped · P0
- **目标**：`run` 创建 session 队列 + 创建 Task(CREATED) + 派发一次 execution。
- **用例**：正常（accepted、state=CREATED、`engine.executions` 1 条、scope.agentId/sessionId 正确、`service.tasks` 1 条）。
- **断言点**：`result.accepted()/state()`；`engine.executions.get(0).scope()`；`queueManager.findBySession(...)`。
- **模板**：`TaskControlServiceWhiteboxTest#runTaskCreatesSessionQueueAndDispatchesExecution`。

### [T2] 恢复 resume（仅命中 WAITING）— shipped · P0
- **目标**：`resume` 命中处于 WAITING 的同一 Task（taskId 不变）并派发一次 resume；无 WAITING 时新建。
- **用例**：正常（先 run→markRunning→markWaiting→resume，`resumed.taskId == created.taskId`、`engine.resumes` 1 条）；边界（无 WAITING→新建）。
- **断言点**：`resumed.taskId()`、`engine.resumes`。
- **模板**：`...#resumeInputTargetsWaitingTaskAndCancelMakesNextRunCreateNewTask`。

### [T3] 取消 cancel（terminal 拒绝）— shipped · P0
- **目标**：非 terminal 任务→CANCELLING 并派发 cancel；terminal 任务→拒绝。
- **用例**：正常（cancelling.state=CANCELLING、`engine.cancels` 1 条）；边界（对 COMPLETED/FAILED/CANCELLED 任务 cancel→accepted=false、"terminal task cannot be cancelled"）。
- **断言点**：`cancelling.state()`、`engine.cancels`。
- **模板**：同 T2 用例 + 对 terminal 任务补一条。

### [T4] Task 状态机 DFA — shipped · P0 ★
- **目标**：合法转移通过、非法转移拒绝、terminal 不可再转、同态自转移幂等、revision 单调递增。
- **用例**：
  - 正常：CREATED→RUNNING→WAITING→RUNNING→COMPLETED，每步 state 正确、最终 `revision=5`。
  - 边界：对每个 terminal 态再请求任意转移 → `accepted=false`、message="transition rejected"。
  - 失败：用**过期 revision** mark → `accepted=false`、message="stale task revision"、`revision` 为当前值。
- **断言点**：`result.state()/revision()/accepted()/message()`。
- **模板**：`...#markMethodsEnforceRevisionAndTransitionOrder`：
```java
var created  = run("agent", "hello", null);
var running  = service.markRunning(mark(created.taskId(), 1L, null, null, null)).toCompletableFuture().join();
var stale    = service.markWaiting(mark(created.taskId(), 1L, USER_INPUT, null, "x")).toCompletableFuture().join(); // 用过期 revision
var waiting  = service.markWaiting(mark(created.taskId(), 2L, USER_INPUT, null, "x")).toCompletableFuture().join();
assertThat(running.state()).isEqualTo(RUNNING);
assertThat(stale.accepted()).isFalse();           // 乐观锁拦截
assertThat(waiting.state()).isEqualTo(WAITING);
```
> 建议补全**转移矩阵穷举**：对 8 个状态 × 8 个目标态逐一断言 `allowed()` 的预期（合法集见 [../02-agent-service-deep-dive.md](../02-agent-service-deep-dive.md) §5.1）。

### [T5] 状态回写 mark* — shipped · P0
- **目标**：`markRunning/Waiting/Succeeded/Failed/Cancelled` 写对状态 + 失败码 + 等待原因。
- **用例**：正常逐个；边界（缺 waitingReason 的 markWaiting→应报错，见代码 `Objects.requireNonNull(waitingReason)`）。
- **断言点**：`task.getState()/getWaitingReason()/getFailureCode()`。
- **模板**：T4 同源。

### [T6] 幂等去重（6 元组 key）— shipped · P0 ★
- **目标**：相同 `(tenant,session,task,agent,action,idempotencyKey)` 只执行一次、返回同一个 `TaskResult`；不同 resume 目标不被错误合并。
- **用例**：
  - 正常：同 key 连发两次 run → `second.equals(first)`、`engine.executions` 仅 1 条、`tasks` 仅 1 条。
  - 边界：同一 resume key 但**不同 taskId** → 不合并、各自命中自己的 Task、`engine.resumes` 2 条。
- **断言点**：`second == first`、`engine.executions.size()`。
- **模板**：`...#idempotencyKeyReturnsSameTaskResultWithoutSecondDispatch` 与 `...#idempotencyKeyDoesNotCollapseDifferentResumeTargets`。

### [T7] session 级并发锁 — shipped · P1
- **目标**：同 `(tenant,session)` 并发 mark*/run 不丢更新、revision 不回退。
- **测试层**：L2（并发）。
- **用例**：起 N 个线程并发对同一 Task 合法推进，最终 revision 等于成功次数+1、无 IllegalState。
- **断言点**：最终 `task.getRevision()` 与状态一致；无异常。
- **模板**：在 whitebox 基础上用 `ExecutorService` + `CountDownLatch` 并发驱动（新增）。
- **注意**：用真实并发，不要用 `Clock.fixed` 之外的共享可变状态。

### [T8] revision 乐观锁 + stale 拒绝 — shipped · P0
- **目标**：`expectedRevision` 不等于当前→拒绝且不改状态。
- **用例**：见 T4 的 stale 用例。
- **断言点**：`accepted=false`、状态/revision 不变。
- **模板**：T4。

### [T9] engine 派发失败→Task FAILED — shipped · P0
- **目标**：`EngineDispatchApi` 返回 `FAILED` 时把 Task 置 FAILED、失败码 `ENGINE_DISPATCH_REJECTED`、`result.accepted=false`。
- **用例**：正常失败注入（`engine.status = FAILED`）；边界（agentId 空白→`AGENT_ID_INVALID`）。
- **断言点**：`result.state()==FAILED`、`task.getFailureCode()`。
- **模板**：`...#rejectedEngineDispatchMarksTaskFailed`：
```java
engine.status = EnqueueEngineStatus.FAILED;
var result = run("agent", "hello", null);
var task = service.findTask("tenant", "session", result.taskId()).orElseThrow();
assertThat(result.state()).isEqualTo(FAILED);
assertThat(task.getFailureCode()).isEqualTo(TaskFailureCode.ENGINE_DISPATCH_REJECTED);
```

---

## 三、engine 层（E1–E11）★ 高优先级

多数用 **L2 闭环**（三件套 + 内存件）或 **L1 单元**。通用闭环装配见 README §5 / `EngineClosedLoopIntegrationTest`。

### [E1] 异步入队 API（execute/resume/cancel）— shipped · P0
- **目标**：三个入口把请求转成 command 入队并返回 `SUCCESS`；gateway 拒绝时返回 `FAILED`。
- **测试层**：L1。
- **用例**：正常（三入口各返回 SUCCESS、gateway 收到对应 command）；失败（gateway 拒绝→FAILED）。
- **断言点**：返回值；录参 gateway 收到的 command 类型/字段。
- **模板**：`engine/api/DefaultEngineDispatchApiTest`（用录参 `EngineQueueGateway`）。

### [E2] command 事件构造 — partial(需核实 stub) · P1
- **目标**：`EngineCommandEventFactory` 把 request 转成带 scope/input/createdAt 的 `EngineCommandEvent`，commandType 正确。
- **测试层**：L1。
- **用例**：execute/resume/cancel 三类各断言字段。
- **断言点**：`event.commandType/scope/input/createdAt`。
- **模板**：`engine/queue/EngineCommandEventFactoryTest`。
- **注意**：设计稿中此类曾为 stub（返回 null）；先用本测试**确认其真实落地**，未落地则闭环测试会断在这里。

### [E3] 队列发布/订阅 — shipped · P1
- **目标**：`EngineQueueGateway.publish` 后 `EngineCommandSubscriber` 能消费并交给 dispatcher。
- **测试层**：L2。
- **用例**：正常（publish→消费→dispatch 被调用）；并发（多 command 顺序/无丢失）。
- **断言点**：dispatcher 收到的 command 数与顺序。
- **模板**：`EngineClosedLoopIntegrationTest` 的装配。

### [E4] 派发 + handler 查找 — shipped · P0 ★
- **目标**：`EngineDispatcher` 按 `scope.agentId` 找到 handler、执行、把事件路由到 task-control 与 access 两侧。
- **测试层**：L1（mock 两 client）+ L2（闭环）。
- **用例**：正常（started→output→completed 路由到正确方法）；失败（见 E5）。
- **断言点**：`taskControl.transitions` 与 `accessLayer.signals` 序列。
- **模板**：`engine/dispatch/EngineDispatcherTest`（Mockito）+ `EngineClosedLoopIntegrationTest`（闭环）。

### [E5] agentId 找不到→FAILED 回写 — shipped · P0
- **目标**：未注册的 agentId → 生成 `EngineFailedEvent` 并 `markFailed`，不执行任何 handler。
- **测试层**：L2。
- **用例**：用未注册 agentId 发 execute → `taskControl.transitions` 含 FAILED、`accessLayer` 收到 fail。
- **断言点**：`taskControl.failed` 非空；无 handler 执行副作用。
- **模板**：在闭环装配里发一个未注册 agentId 的 scope。

### [E6] AgentHandler SPI 执行 — shipped · P0
- **目标**：`AgentHandler.execute(ctx)` 返回的事件流被正确消费。
- **测试层**：L1/L2。
- **用例**：正常（fake handler 产出多事件）；异常（handler 抛异常→须转 `EngineFailedEvent`、不悬挂，见 `AgentServiceEndToEndIT` 的 `ThrowingAgentHandler`）。
- **断言点**：异常路径仍产出 terminal/fail。
- **模板**：`AgentServiceEndToEndIT#aThrowingAgentStillRepliesWithATerminalError`。

### [E7] OpenJiuwen 适配（建/转/映射）— partial/stub · P1
- **目标**：`OpenJiuwenMessageConverter`（EngineInput→OpenJiuwen 输入）、`OpenJiuwenResultMapper`（结果→事件）转换正确。
- **测试层**：L1（纯转换，无需真实 OpenJiuwen）。
- **用例**：converter（文本/variables 映射）；mapper（answer→Completed、error→Failed、interrupt→Interrupted）。
- **断言点**：转换后对象字段。
- **模板**：`OpenJiuwenMessageConverterTest`、`OpenJiuwenResultMapperTest`（mapper 用可注入的 ID/time 供应器）。
- **注意**：`OpenJiuwenAgentFactory.create`/converter 设计稿曾返回 null；**真实 OpenJiuwen 执行路径用 fake `AgentHandler` 隔离**，不纳入功能闭环测试，避免被 stub 阻塞。

### [E8] 9 类执行事件→双链路映射 — shipped · P0 ★
- **目标**：每类事件按设计映射到 task-control 动作与 access 动作（见 [../02-agent-service-deep-dive.md](../02-agent-service-deep-dive.md) §6.4 表）。
- **测试层**：L2。
- **用例**：
  - Started→`markRunning`，无 access。
  - Output→无 task，`appendOutput`。
  - Interrupted(HUMAN_INPUT/APPROVAL)→`markWaiting` + `requestUserInput`；(WAITING_CHILD_AGENT)→`markWaiting`、无 access。
  - Completed→`markSucceeded` + `completeOutput`；Failed→`markFailed` + `failOutput`；Cancelled→`markCancelled`。
- **断言点**：`taskControl.transitions` 与 `accessLayer.signals` 的**精确序列**（`containsExactly`）。
- **模板**：`EngineClosedLoopIntegrationTest`（断言 `containsExactly("RUNNING:task-1","WAITING:task-1",...)`）。

### [E9] 中断/恢复 — shipped · P0
- **目标**：中断→WAITING+requestUserInput；resume→RUNNING→completed。
- **测试层**：L2。
- **用例**：见 `FakeInterruptingAgentHandler` 驱动的 execute→resume。
- **断言点**：两轮 transitions/signals 序列。
- **模板**：`EngineClosedLoopIntegrationTest#executeThenResume_...`。

### [E10] Agent 调 Agent INLINE — shipped(经 OpenJiuwen AbilityManager) · P2
- **目标**：INLINE 子 agent 在 parent handler 内执行，结果归 parent，**不**创建 child task、**不**直接推 access。
- **测试层**：L2。
- **用例**：用一个内部再调一次的 fake handler，断言只有 parent task 的状态/输出。
- **断言点**：无额外 task；最终输出归 parent。
- **模板**：定制 fake handler（基于 `FakeInterruptingAgentHandler` 改造）。

### [E11] Agent 调 Agent CHILD_TASK — design_only · P2
- **现状**：仅保留 `EngineAgentCallEvent`/`AgentCallMode` 模型，未进入执行闭环。
- **测试**：**不纳入功能测试**；可写一个"事件模型字段校验"L1 单测占位，并在用例库标注"设计保留，待契约定义后补"。

---

## 四、session 层（S1–S7）· 精炼表

会话层无外部协作复杂度，主要 **L1 单元** + Redis 用 **L3 Testcontainers**。

| 编号 | 功能点 | 测试层 | 关键用例 | 断言点 | 现状 |
|---|---|---|---|---|---|
| S1 | loadOrCreate/get/exists/delete | L1 | 不存在则创建；exists 轻量判断；delete 后 get 空 | 返回值/Optional | shipped |
| S2 | appendMessage/putState/putMetadata | L1 | 追加消息/写状态/写元信息；version+1、updatedAt 刷新 | `session.version` 递增、字段值 | shipped |
| S3 | appendTask（占位） | L1 | 追加 Task 占位进 tasks 列表 | `tasks.size()` | shipped |
| S4 | InMemory 存储 | L1 | 读写删；重启丢失（进程内） | 值一致 | shipped |
| S5 | Redis 存储+TTL+CAS | L3(Testcontainers) | 跨"进程"共享、TTL 过期、`saveIfVersion` 版本冲突不覆盖 | CAS 失败时不覆盖 | partial |
| S6 | 存储工厂 memory/redis | L1 | type=memory→InMemory；type=redis 但无 `RedisSessionCommands`→**失败**（不静默退内存） | 抛明确异常 | shipped |
| S7 | 乐观锁 version/saveIfVersion | L1/L2(并发) | 并发更新无静默覆盖、version 单调 | version 单调、无丢失 | shipped |

> S5/S7 的并发与 CAS 是重点：用 `RedisSessionStore` + Testcontainers Redis，或对 `saveIfVersion` 做并发断言。

---

## 五、queue 层（Q1–Q4）· 精炼表

纯内存件，**L1/L2**。

| 编号 | 功能点 | 测试层 | 关键用例 | 断言点 | 现状 |
|---|---|---|---|---|---|
| Q1 | 注册/查找 session 队列 | L1 | 注册后 `findBySession` 命中；重复 ID 拒绝 | Optional/异常 | shipped |
| Q2 | 创建内存 session 队列 | L1 | `QueueFactory.inMemorySessionQueue` 创建并登记 | 队列存在 | shipped |
| Q3 | offer/snapshot/find | L1 | offer 后 snapshot 含元素；find 按谓词命中 | 元素/谓词 | shipped |
| Q4 | 生产队列+至少一次 | — | **design_only**，不测；标注待生产实现 | — | design_only |

> 模板：`taskcontrol/test/QueueManagerWhiteboxTest`。

---

## 六、跨模块功能点（X1–X7）· 精炼表

agent-service 与兄弟模块的 SPI 协作，多为 **L0 架构** + **L1 契约**。

| 编号 | 功能点 | 测试层 | 关键用例 | 现状 |
|---|---|---|---|---|
| X1 | Engine envelope 派发（registry-only） | L0+L1 | registry 严格匹配 engineType；不匹配→`EngineMatchingException`；禁止 registry 外 instanceof | shipped |
| X2 | 每引擎声明 hook surface | L0 | ArchUnit：ExecutorAdapter 暴露 `hookSurface()` | shipped |
| X3 | Middleware+HookPoint 分发 | L1 | hook 在 LLM/工具/生命周期边界被触发（W2 多为 design_only） | partial |
| X4 | SuspendSignal 中断原语 | L1 | 字段校验、`forClientCallback` 变体、null 校验 | shipped |
| X5 | S2C 回调 | L1 | `S2cCallbackEnvelope` 校验（运行期强制 design_only） | partial |
| X6 | IngressGateway（C2S） | L1 | `IngressEnvelope` 6 必填字段、ACCEPTED 返回 Task Cursor | shipped(SPI) |
| X7 | 运维健康/指标 | L3 | `/v1/health` 200+JSON、`/actuator/prometheus` 含 `springai_ascend_*` | shipped |

> 模板：`engine/runtime/EngineRegistryResolveTest`（X1）、`EveryEngineDeclaresHookSurfaceTest`（X2）、`bus/spi/engine/SuspendSignalTest`（X4）。

---

## 七、一页速查：测什么用哪个模板

| 你要测 | 抄哪个现有测试 |
|---|---|
| A2A 端到端回包/不泄漏 | `AgentServiceEndToEndIT` |
| 引擎闭环/中断恢复/取消 | `EngineClosedLoopIntegrationTest` |
| 状态机/幂等/乐观锁/失败回写 | `TaskControlServiceWhiteboxTest` |
| 派发路由（mock 两 client） | `EngineDispatcherTest` |
| 入队 API/事件工厂 | `DefaultEngineDispatchApiTest` / `EngineCommandEventFactoryTest` |
| OpenJiuwen 转换/映射 | `OpenJiuwen{MessageConverter,ResultMapper}Test` |
| 队列管理 | `QueueManagerWhiteboxTest` |
| 架构纪律 | `runtime/architecture/*` + `engine/runtime/*` |

接口字段与自动化 → [03-interface-and-cases.md](03-interface-and-cases.md)；非功能基线 → [04-test-baselines.md](04-test-baselines.md)；端到端与 CI → [05-e2e-and-automation.md](05-e2e-and-automation.md)。
