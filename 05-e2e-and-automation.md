# 05 · 端到端（E2E）与自动化

> 回答两个核心问题：① agent-service **能不能脱离整个框架**单独做端到端测试？怎么做？② 怎么把这些测试**自动化**、接进 CI？

---

## 1. 结论先行：能，而且很干净

**agent-service 可以在单进程内、不连任何外部服务，跑完整的 A2A 端到端测试。** 三条理由（均经代码核实）：

1. **兄弟模块是编译期 SPI**：`agent-bus / agent-execution-engine / agent-middleware` 以 jar 形式进 agent-service，请求链路里**没有任何远程调用**（grep 全主源码无 `WebClient/RestClient/Feign/gRPC`）。所以**不需要把兄弟服务跑起来**。
2. **核心件全是内存实现**：内部队列 `InMemoryEngineQueueGateway`、会话默认 `InMemorySessionStore`、任务队列 `QueueManager` 都是进程内的。
3. **Agent 框架可替身**：引擎通过 `AgentHandler` SPI 调 Agent，注册一个 fake `AgentHandler` 即可代替真实 OpenJiuwen/LLM。

**已存在的证据**：`agent-service/src/test/java/.../bootstrap/AgentServiceEndToEndIT.java` 就是这么做的——`@SpringBootTest` 启动五层、发 A2A 请求、fake echo agent 回包、断言出站终帧 + 队列无泄漏，**全程零外部服务**。

---

## 2. 两种端到端 Harness（按需选）

### Harness A · 纯 Java 闭环（无 Spring，最快，毫秒级）
**适用**：验证引擎/状态机的协作闭环（execute→中断→resume→complete、cancel、失败回写）。
**装配配方**（来自 `EngineClosedLoopIntegrationTest`，照抄）：

```java
// 1) 两个录音机（任务侧 + 用户侧）
RecordingTaskControlClient taskControl = new RecordingTaskControlClient();
RecordingAccessLayerClient accessLayer = new RecordingAccessLayerClient();

// 2) 注册一个假 Agent
AgentHandlerRegistry registry = new DefaultAgentHandlerRegistry();
registry.register("echo-agent", new FakeInterruptingAgentHandler("echo-agent"));

// 3) 内存队列 + 派发器 + 订阅者（后台线程消费） + 入队 API
InMemoryEngineQueueGateway gateway = new InMemoryEngineQueueGateway();
EngineDispatcher dispatcher = new EngineDispatcher(registry, taskControl, accessLayer);
new EngineCommandSubscriber(gateway, dispatcher).start();
EngineDispatchApi api = new DefaultEngineDispatchApi(new EngineCommandEventFactory(), gateway);

// 4) 驱动并断言（精确序列）
api.enqueueExecution(new EnqueueEngineExecutionRequest(scope, input));
assertThat(taskControl.transitions).containsExactly("RUNNING:task-1", "WAITING:task-1");
api.enqueueResume(new EnqueueEngineResumeRequest(scope, input));
assertThat(accessLayer.signals).containsExactly("REQUEST_INPUT:task-1", "APPEND:task-1", "COMPLETE:task-1");
```
> 异步：订阅者在后台线程消费，断言前用**轮询等待**（或先用同步 gateway）。

### Harness B · Spring A2A 端到端（@SpringBootTest，真实装配）
**适用**：验证从 A2A 入站到出站回包的整条链路 + 五层胶水装配 + 资源不泄漏。
**装配配方**（来自 `AgentServiceEndToEndIT`，照抄）：

```java
@SpringBootTest(classes = TestRuntime.class)   // 关键：用自定义最小配置，不用 @SpringBootApplication
class MyA2aE2E {

    @Autowired A2aAccessService a2a;
    @Autowired A2aOutputRegistry outputRegistry;
    @Autowired EgressQueueRegistry egressQueueRegistry;

    @Test void a2a_runs_and_replies() {
        A2aEnvelope env = envelope("session-1", "hello world");
        A2aAcceptedResponse accepted = a2a.send(env);
        List<A2aOutput> out = awaitOutputs(new A2aOutputHandle("tenant-e2e","session-1",accepted.taskId()));
        assertThat(out.get(out.size()-1).terminal()).isTrue();
        assertThat(egressQueueRegistry.find("tenant-e2e","session-1",accepted.taskId())).isEmpty(); // 无泄漏
    }

    // ★ 最小运行时：只 @Import 五层配置 + 一个假 agent，绕开 datasource/Flyway/security
    @SpringBootConfiguration
    @Import({ TaskControlAutoConfiguration.class, AgentServiceBootstrapConfiguration.class,
              AccessLayerConfiguration.class, SessionManageConfiguration.class, EngineAutoConfiguration.class })
    static class TestRuntime {
        @Bean AgentHandler echoAgentHandler() { return new EchoAgentHandler(); }   // 假 Agent，回显
        @Bean AgentHandler boomAgentHandler() { return new ThrowingAgentHandler(); } // 假 Agent，抛异常
    }
}
```

**为什么用 `@SpringBootConfiguration` + `@Import` 而不是 `@SpringBootApplication`？**
因为 `AgentServiceApplication` 会触发全量自动配置（datasource/Flyway/security），需要 Postgres 才能 boot。`TestRuntime` 只显式 `@Import` 五层配置，**绕开数据库**，从而零外部依赖启动。

**fake agent 怎么被识别？** `EngineAutoConfiguration` 会把容器里**每一个 `AgentHandler` bean 按 `agentId()` 自动注册**进 registry（`handlers.orderedStream().forEach(...register(handler.agentId(), handler))`）。所以你只要 `@Bean` 一个 fake handler 即可插入一个 agent。

---

## 3. 依赖替身矩阵（什么必需 / 可选 / 怎么替身）

| 依赖 | 启动必需？ | 默认 | 端到端怎么处理 |
|---|---|---|---|
| **Postgres / Flyway** | 用 `@SpringBootApplication` 时**是**；用 `TestRuntime` 时**否** | application.yml 指向 localhost:5432 | 三选一：① `TestRuntime` 直接绕开；② `spring.flyway.enabled=false` + `app.idempotency.allow-in-memory=true`；③ 需验持久化/迁移时上 Testcontainers PG |
| **Redis** | 否 | `session.store.type=MEMORY` → `InMemorySessionStore` | 默认内存即可；验 Redis 时设 `type=redis` + Testcontainers Redis |
| **OpenAI/Anthropic（LLM）** | 否（dummy key 即可 boot） | application.yml 用 `dummy-no-call-expected` 占位 | 不调真模型；要测网关用 WireMock 打桩 `/v1/chat/completions` |
| **OpenJiuwen agent-core** | 否（默认不注册） | `openjiuwen.enabled=false` | 注册 fake `AgentHandler` 替代；真集成走夜间冒烟 |
| **Vault** | 否 | `spring.cloud.vault.enabled=false` | 无需处理 |
| **Resilience4j verifier** | 否 | 已 `autoconfigure.exclude` | 无需处理 |
| **内部事件队列** | 是（但内存） | `InMemoryEngineQueueGateway` | 开箱即用 |

> 一句话：**用 `TestRuntime` + 默认内存 + fake agent + dummy LLM key，零外部服务就能 E2E。**

---

## 4. 渐进式自动化路线（从快到真）

```
① 无外部依赖（PR 必跑，秒级）
   纯 Java 闭环(Harness A) + Spring A2A E2E(Harness B, TestRuntime + fake agent)
        │  覆盖：状态机/幂等/中断恢复/取消/失败/出站不泄漏/A2A 协议
        ▼
② 加 Testcontainers Postgres（PR 或合并门禁）
   验证：幂等持久化、Flyway 迁移(V1/V2)、RLS 跨租户不可见
        ▼
③ 加 WireMock（按需）
   验证：LLM 网关（base-url 指向 WireMock）、push notification callback
        ▼
④ 真 OpenJiuwen 冒烟（夜间/手动）
   注册真实 OpenJiuwenAgentHandler + 真模型，少量 happy-path
```

**重要**：①②③ 全部不依赖真实大模型与真实 OpenJiuwen，可在普通 CI 机器上稳定跑；④ 才需要真实环境，且只做冒烟。

---

## 5. CI 命令分层

| 阶段 | 命令 | 跑什么 | 何时 |
|---|---|---|---|
| 快测 | `./mvnw -q -pl agent-service -am test` | 单元/白盒/纯 Java 闭环（surefire） | 每次 push/PR |
| 集成 | `./mvnw -q -pl agent-service -am verify` | 上面 + `*IT.java`（failsafe，含 A2A E2E + Testcontainers） | PR 合并门禁 |
| 全量门禁 | `./mvnw -q -pl agent-service -am -Pquality verify` | 上面 + gate + ArchUnit + 契约 | 合并到 main |

约定：
- **命名规则**：单元/白盒/闭环 = `*Test.java`（surefire，`test` 阶段）；端到端/容器 = `*IT.java`（failsafe，`verify` 阶段）。
- **异步等待**：用轮询/awaitility 风格（参考 `AgentServiceEndToEndIT.awaitOutputs`：5 秒超时，每 20ms 轮询直到 terminal），**不要**用裸 `Thread.sleep(固定值)` 当唯一同步。
- **测试隔离**：每个用例用独立 `sessionId/taskId`；E2E 后断言 `EgressQueueRegistry.find()` 为空，防资源泄漏污染下个用例。
- **posture**：CI 用 `APP_POSTURE=dev`；另起一个 job 用 `research` 验证"缺配置即 fail-closed"。
- **覆盖率建议**（JaCoCo）：核心层 `taskcontrol`/`engine` 行覆盖 ≥80%、分支 ≥70%；`access` ≥70%；其余 ≥60%。门槛随基线绿后逐步上调。

---

## 6. 可复用测试支撑库蓝图（建议）

当前三件套（`FakeInterruptingAgentHandler`/`RecordingTaskControlClient`/`RecordingAccessLayerClient`）+ `TestRuntime` 的 `EchoAgentHandler`/`ThrowingAgentHandler` 都在 `agent-service/src/test` 内。建议：

1. **抽成 test-fixtures**：用 Maven `test-jar` 或 Gradle `java-test-fixtures`，把这些 fake/recording/Echo/Throwing handler 与 `TestRuntime` 配方导出，供其它模块/集成测试复用。
2. **补两个常用 fake**：`CompletingAgentHandler`（一次性完成，不中断）、`ApprovalAgentHandler`（APPROVAL 中断），覆盖更多事件路径。
3. **沉淀一个 `A2aTestClient`**：封装 envelope 构造 + send + `awaitOutputs` 轮询，进一步降低写 E2E 的门槛。

> 这是"让测试人员快速上手"的长期手段：把装配配方变成一行 `new A2aTestClient(ctx).send(...).awaitTerminal()`。

---

## 7. 端到端用例清单（建议优先实现）

| 用例 | Harness | 现状参考 |
|---|---|---|
| A2A SendMessage 正常完成 | B | `AgentServiceEndToEndIT#a2aRequest...` |
| Agent 抛异常仍 terminal、无泄漏 | B | `...#aThrowingAgent...` |
| 中断→resume→完成 | A | `EngineClosedLoopIntegrationTest#executeThenResume` |
| 取消（运行中→cancelled，不跑 handler） | A | `...#cancel_marksTaskCancelled...` |
| SSE 流式多帧、末帧 terminal | B(HTTP) | 新增（[03](03-interface-and-cases.md) §6.1） |
| push notification 回调收到事件 | B+WireMock | 新增 |
| `/v1/runs` 未路由（契约漂移） | B(HTTP) | 新增（[03](03-interface-and-cases.md) §4） |
| Testcontainers：幂等持久化 + 迁移 | B+PG | 新增（路线 ②） |
| posture：research 缺配置 fail-closed | B | 新增 |

> 先把前 4 个跑绿（已有现成模板），再按路线 ②③ 扩展。
