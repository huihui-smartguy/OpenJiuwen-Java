# agent-service 测试方案（面向测试人员）

> 读者：刚接手 agent-service 测试的工程师。
> 目标：**30 分钟内**理解整体测试思路、跑通现有测试、照抄出你的第一个测试。
> 约定：中文叙述，类名/方法名英文；所有路径相对 `<ROOT> = spring-ai-ascend-main`。

---

## 0. 这套方案怎么读

| 文档 | 你什么时候看它 |
|---|---|
| 本 README | 第一天：建立全局认知 + 跑通环境 + 抄出第一个测试 |
| [01-test-strategy.md](01-test-strategy.md) | 想知道"该写哪种测试、用什么工具、测到什么程度" |
| [02-feature-test-design.md](02-feature-test-design.md) ★ | 拿到一个功能点，想知道"具体怎么测、有哪些用例、怎么断言" |
| [03-interface-and-cases.md](03-interface-and-cases.md) | 做接口测试：请求长什么样、期望什么响应、怎么自动化 |
| [04-test-baselines.md](04-test-baselines.md) | 做性能/可靠/安全/兼容等非功能测试：测什么、通过标准是多少 |
| [05-e2e-and-automation.md](05-e2e-and-automation.md) ★ | 想做端到端、想接 CI 自动化、想知道能不能脱离整个平台单独测 |

★ 是核心，建议精读。

---

## 1. 一句话理解被测系统

agent-service 是一个 **Spring Boot 进程**，内部分 5 层，把一次用户请求从「协议入站」一路驱动到「Agent 执行」再把结果推回去：

```
A2A 入站(/a2a/)  →  access(L1)  →  taskcontrol(L4,Task状态机+幂等)  →  engine(L5)  →  AgentHandler  →  (OpenJiuwen/真LLM 或 fake)
                         ↑                                                        │
                         └──────────── 出站(通知/SSE/push) ◀── 状态回写 + 用户输出 ─┘
```

> 模块解读见 [../02-agent-service-deep-dive.md](../02-agent-service-deep-dive.md)，整体架构见 [../01-architecture-summary.md](../01-architecture-summary.md)。

**对测试最重要的一个事实**：agent-service 依赖的 agent-bus / agent-execution-engine / agent-middleware 都是**编译期 jar（SPI）**，请求链路里**没有任何远程调用**。所以你可以**在一个进程内、不连任何外部服务（数据库/Redis/真实大模型/OpenJiuwen）**就把它从头到尾测一遍——这正是本方案能"照抄上手"的根基。

---

## 2. 测试金字塔（一眼看懂打法）

```
        ▲ 少而慢
   ┌─────────────┐  L4 契约 / 基线        OpenAPI 漂移、A2A 字段契约、性能/安全/可靠基线   → 04
   ├─────────────┤  L3 Spring E2E         @SpringBootTest 走 /a2a/ 全链路（fake agent） → 05
   ├─────────────┤  L2 组件/闭环集成(纯Java) 三件套装配引擎/状态机闭环（无 Spring）         → 02 / 05
   ├─────────────┤  L1 单元 / 白盒         JUnit5+AssertJ+Mockito，单类逻辑              → 02
   └─────────────┘  L0 架构 / 静态         ArchUnit、源码扫描，守护依赖方向/SPI 纯净       → 01
        ▼ 多而快
```

原则：**底层多写、跑得快**（单元/白盒/闭环，毫秒级，PR 必跑）；**顶层少写、覆盖关键路径**（E2E/基线，秒级，按需）。详见 [01-test-strategy.md](01-test-strategy.md)。

---

## 3. 跑通现有测试（5 分钟）

```bash
# 进入仓库根目录
# 只跑 agent-service 的单元/白盒/集成测试（快）
./mvnw -q -pl agent-service -am test

# 含 *IT.java 端到端集成测试（failsafe）
./mvnw -q -pl agent-service -am verify

# 含治理 gate + ArchUnit（最严格，合并前跑）
./mvnw -q -pl agent-service -am -Pquality verify
```

> posture 由环境变量 `APP_POSTURE` 控制：`dev`（默认，宽松，内存后端可用）/ `research` / `prod`（缺配置即启动失败）。本地测试用 `dev`。

现有测试清单（共约 22 个测试类）见 [../02-agent-service-deep-dive.md](../02-agent-service-deep-dive.md) §9——它们就是你最好的模板。

---

## 4. 必懂的「测试三件套」（agent-service 测试的灵魂）

位于 `agent-service/src/test/java/com/huawei/ascend/service/engine/support/`，让你**不需要真实 Agent / 大模型**就能测完整闭环：

| 夹具 | 作用 | 你用它来…… |
|---|---|---|
| `FakeInterruptingAgentHandler` | 假 Agent：第一次执行就「中断等输入」，第二次「完成」 | 模拟 Agent 行为，免 LLM/OpenJiuwen |
| `RecordingTaskControlClient` | 录下每次状态迁移（`RUNNING:task-1`、`WAITING:task-1`…） | 断言**任务状态机**走对了 |
| `RecordingAccessLayerClient` | 录下每个出站信号（`APPEND`、`COMPLETE`、`REQUEST_INPUT`…） | 断言**用户看到的输出**对且有序 |

它们三个组合起来 = 一个完整的「闭环测试台」：**假 Agent 演戏，两个录音机记录任务侧和用户侧分别看到了什么**。

---

## 5. 你的第一个测试：照抄这个（10 分钟）

下面是真实存在的 `EngineClosedLoopIntegrationTest`（纯 Java、无 Spring、无外部依赖）。复制它，改 `scope()`/断言，就是你的新用例：

```java
@BeforeEach
void setUp() {
    taskControl = new RecordingTaskControlClient();
    accessLayer = new RecordingAccessLayerClient();
    AgentHandlerRegistry registry = new DefaultAgentHandlerRegistry();
    registry.register("echo-agent", new FakeInterruptingAgentHandler("echo-agent"));

    gateway = new InMemoryEngineQueueGateway();
    EngineDispatcher dispatcher = new EngineDispatcher(registry, taskControl, accessLayer);
    new EngineCommandSubscriber(gateway, dispatcher).start();
    api = new DefaultEngineDispatchApi(new EngineCommandEventFactory(), gateway);
}

@Test
void executeThenResume_drivesInterruptThenCompletion() {
    // 1) 第一次执行：Agent 中断、等待输入
    api.enqueueExecution(new EnqueueEngineExecutionRequest(scope(), input()));
    assertThat(taskControl.transitions).containsExactly("RUNNING:task-1", "WAITING:task-1");
    assertThat(accessLayer.userInputRequests).hasSize(1);

    // 2) 恢复：Agent 完成，先流式输出再完成
    api.enqueueResume(new EnqueueEngineResumeRequest(scope(), input()));
    assertThat(taskControl.transitions)
        .containsExactly("RUNNING:task-1", "WAITING:task-1", "RUNNING:task-1", "SUCCEEDED:task-1");
    assertThat(accessLayer.signals)
        .containsExactly("REQUEST_INPUT:task-1", "APPEND:task-1", "COMPLETE:task-1");
}
```

> 出处：`agent-service/src/test/java/com/huawei/ascend/service/engine/EngineClosedLoopIntegrationTest.java`。
> 想走真正的 HTTP/A2A 全链路？看 [05-e2e-and-automation.md](05-e2e-and-automation.md) 的 `@SpringBootTest` 配方（同样不需要外部服务）。

---

## 6. 然后做什么

1. 用 [02-feature-test-design.md](02-feature-test-design.md) 找到你负责的功能点（A/S/Q/T/E/X 编号，见 [../05-diagrams.md](../05-diagrams.md) 图 6c），照模板补用例。
2. 接口测试看 [03-interface-and-cases.md](03-interface-and-cases.md)。
3. 非功能（性能/安全/可靠…）看 [04-test-baselines.md](04-test-baselines.md)，注意区分「已实现可强制」与「仅设计目标」。
4. 端到端 + 自动化 + CI 看 [05-e2e-and-automation.md](05-e2e-and-automation.md)。
