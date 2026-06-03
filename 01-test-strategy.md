# 01 · 测试策略详解

> 一句话：**底层多写跑得快，顶层少写覆盖关键路径**。本章讲清楚「分几层、每层测什么/不测什么、用什么工具、什么时候写」，并给一张「我该写哪种测试」的决策表。

---

## 1. 为什么分五层

agent-service 的逻辑分布很不均匀：

- 大量**纯逻辑**（Task 状态机、幂等去重、事件→动作映射、消息转换）——适合极快的单元/白盒测试；
- 关键在于**多组件协作的闭环**（入队→派发→执行→回写/出站）——适合纯 Java 组件集成；
- 对外是 **A2A HTTP 协议** + 异步出站——需要 Spring 装配的端到端验证；
- 还有**架构纪律**（依赖方向、SPI 纯净、registry-only 派发）——只能靠静态/架构测试守。

把它们硬塞进一种测试会两头不讨好。所以分五层，各司其职。

```
        ▲ 少而慢、贴近真实
   L4  契约 / 基线        "对外承诺没破、非功能达标吗？"
   L3  Spring E2E         "整条链路在真实 Spring 里跑通吗？"
   L2  组件 / 闭环集成      "几个组件拼起来，协作对吗？"（纯 Java，无 Spring）
   L1  单元 / 白盒         "这一个类的逻辑对吗？"
   L0  架构 / 静态         "依赖方向、SPI 纯净度没被破坏吗？"
        ▼ 多而快、贴近代码
```

---

## 2. 五层逐层说明（每层一张卡）

### L0 · 架构 / 静态测试
- **测什么**：依赖方向（runtime 不得 import platform）、SPI 包只 import `java.*`、引擎只经 `EngineRegistry` 派发、`repository.save()` 只在允许处调用、每个引擎声明 hook surface。
- **不测什么**：任何运行期行为、业务逻辑。
- **工具**：ArchUnit（`ArchRule`、`noClasses().that()...`）、源码正则扫描。
- **何时写**：定架构纪律时写一次，长期守门；改包结构/加新模块时回归。
- **现有代表类**：`runtime/architecture/AudienceBExtensionSeamsArchTest`、`RunContextIdentityAccessorsTest`、`RunRepositorySaveGuardTest`、`SpanTenantAttributeRequiredTest`；`engine/runtime/EnginePayloadDispatchOnlyViaRegistryTest`、`EveryEngineDeclaresHookSurfaceTest`。
- **特点**：很多是「空集也通过」（`allowEmptyShould(true)`）的**前置守卫**——当对应类落地时自动开始约束。

### L1 · 单元 / 白盒测试
- **测什么**：单个类的纯逻辑——状态机转移规则、消息/结果转换、事件工厂字段、envelope 校验、SuspendSignal 字段校验。
- **不测什么**：跨组件协作、线程、HTTP、外部依赖。
- **工具**：JUnit5 + AssertJ（`assertThat`）+ Mockito（`mock/when/verify`，仅当被测类有外部协作者时）。
- **何时写**：写每个有逻辑分支的类时**同步**写（最高性价比，应占测试数量的大头）。
- **现有代表类**：`engine/dispatch/EngineDispatcherTest`（用 Mockito mock 两个 client + fake handler）、`engine/api/DefaultEngineDispatchApiTest`、`engine/queue/EngineCommandEventFactoryTest`、`engine/adapter/openjiuwen/OpenJiuwen{MessageConverter,ResultMapper}Test`、`bus/spi/engine/SuspendSignalTest`。
- **关键技巧**：被测类**直接 `new`**，协作者用 mock 或 fake；时间/ID 用可注入的 `Clock.fixed(...)`/供应器，保证可重复断言（见 `TaskControlServiceWhiteboxTest` 注入 `Clock.fixed`）。

### L2 · 组件 / 闭环集成测试（纯 Java，**无 Spring**）★
- **测什么**：多个真实组件拼起来的**协作闭环**：引擎 `入队→订阅→派发→handler→事件→回写/出站`；task-control 的 `提交→状态机→幂等→派发`。
- **不测什么**：HTTP/A2A 协议解析、Spring 装配、真实外部服务。
- **工具**：纯 JUnit5 + AssertJ + **测试三件套**（`FakeInterruptingAgentHandler` / `RecordingTaskControlClient` / `RecordingAccessLayerClient`）+ 内存件（`InMemoryEngineQueueGateway`、`QueueManager`、`DefaultAgentHandlerRegistry`）。
- **何时写**：验证「设计闭环成立」时；这是 agent-service **最有价值**的一层，因为核心价值就在多组件协作。
- **现有代表类**：`engine/EngineClosedLoopIntegrationTest`（引擎全闭环）、`taskcontrol/test/TaskControlServiceWhiteboxTest`（状态机+幂等+派发，用 `RecordingEngineDispatchApi`）、`taskcontrol/test/TaskflowEngineBridgeWhiteboxTest`（task-control↔engine 桥接）、`taskcontrol/test/QueueManagerWhiteboxTest`。
- **关键技巧**：异步链路（订阅者在后台线程消费）用**轮询等待**断言，不要假设同步完成。

### L3 · Spring 端到端（E2E）测试
- **测什么**：从 **A2A 入站**到**出站回包**的整条链路，在真实 Spring 容器里跑通，验证五层"胶水"装配正确、出站资源不泄漏。
- **不测什么**：真实大模型质量、真实 OpenJiuwen（用 fake agent 替代）、真实 DB 持久化（除非显式加 Testcontainers）。
- **工具**：`@SpringBootTest` + 自定义 `@SpringBootConfiguration`（只 `@Import` 五层配置，不拉 datasource/security）+ `@Bean` fake `AgentHandler` + 轮询等待。可加 REST-assured/MockMvc 走真实 HTTP `/a2a/`。
- **何时写**：验证关键业务流（正常完成、Agent 抛异常仍有 terminal、取消、流式、push）端到端可用；数量要克制。
- **现有代表类**：`bootstrap/AgentServiceEndToEndIT`（含 `TestRuntime` 配方、Echo/Throwing 两个 fake agent）。
- **关键技巧**：命名 `*IT.java` 由 **failsafe** 在 `verify` 阶段跑；用 `@SpringBootConfiguration`+`@Import` 而非 `@SpringBootApplication`，**绕开数据库/Flyway/安全**自动配置（详见 [05](05-e2e-and-automation.md)）。

### L4 · 契约 / 基线测试
- **测什么**：对外契约不漂移（OpenAPI 快照、A2A 字段契约）+ 非功能基线（性能/可靠/安全/兼容/观测）。
- **不测什么**：内部实现细节。
- **工具**：契约快照比对、REST-assured 字段断言、JMH（性能）、Testcontainers（RLS/迁移）、WireMock（LLM 网关）、gate 脚本。
- **何时写**：发布前 + 回归基线；多数非功能基线当前是 `design_only`，先做"目标值校验 + 脚手架"。
- **指向**：接口契约见 [03](03-interface-and-cases.md)，七维基线见 [04](04-test-baselines.md)。

---

## 3. 「我该写哪种测试」决策表

| 你要验证的对象 | 推荐层 | 用什么 | 参考现有类 |
|---|---|---|---|
| 一个类的分支逻辑（转移规则、转换、校验） | L1 | JUnit+AssertJ(+Mockito) | `EngineCommandEventFactoryTest` |
| Task 状态机 / 幂等 / 乐观锁 | L2(白盒) | `new TaskControlService(QueueManager, RecordingEngineDispatchApi, Clock.fixed)` | `TaskControlServiceWhiteboxTest` |
| 引擎"入队→执行→回写/出站"闭环 | L2 | 三件套 + `InMemoryEngineQueueGateway` | `EngineClosedLoopIntegrationTest` |
| 中断→恢复 / 取消 路径 | L2 | `FakeInterruptingAgentHandler` | `EngineClosedLoopIntegrationTest` |
| A2A 请求端到端回包 / 出站不泄漏 | L3 | `@SpringBootTest(TestRuntime)` + fake agent | `AgentServiceEndToEndIT` |
| `/a2a/` HTTP 协议字段/状态码 | L3 | REST-assured/MockMvc | （新增，见 03） |
| Redis 会话存储 / Postgres 迁移 | L3 | Testcontainers | （新增，见 05） |
| LLM 网关行为 | L3 | WireMock 打桩 | （新增，见 05） |
| 依赖方向 / SPI 纯净 / registry-only | L0 | ArchUnit | `EnginePayloadDispatchOnlyViaRegistryTest` |
| 性能 SLO / 安全基线 / OpenAPI 漂移 | L4 | JMH / 注入测试 / 快照比对 | 见 04 |

---

## 4. 测试诚实规则（必须遵守）

复用仓库既定规则（`docs/architecture/l0/09-verification/test-strategy.md` + `CLAUDE.md` Rule D-4「三层测试」/ D-5「自审是发布闸」）：

1. **不用 mock 掩盖契约缺失**——被测的子系统本身不能被 mock 掉再宣称"集成测试"。
2. **不用 happy-path 证明失败语义**——失败/边界路径要有专门用例（如 Agent 抛异常仍须 terminal、engine reject→Task FAILED）。
3. **不用字段 snapshot 替代语义测试**——OpenAPI 快照只防漂移，不替代行为断言。
4. **`design_only` 契约**可以有 harness 草稿，**但不得声明 runtime enforced**——报告里要标 `shipped / design_only / schema_shipped / stub`。
5. **三层齐全才算可发布**：一个功能点要同时有 单元 / 集成 / 公共契约 三层覆盖（见 02 的逐特性模板）。

---

## 5. 与既有四层策略的对应

仓库原有「四层测试策略」（Contract / Business Activity / Technical Sub-scenario / Architecture，见 `docs/architecture/l0/09-verification/test-strategy.md`）是**平台级**说法；本章的 L0–L4 是把它**落到 agent-service 代码**的可执行版：

| 仓库四层 | 本章映射 |
|---|---|
| Architecture | L0 |
| Contract | L1（SPI 契约）+ L4（HTTP/OpenAPI 契约） |
| Technical Sub-scenario | L2（机制闭环：状态机、suspend/resume、失败注入） |
| Business Activity | L3（A2A 端到端业务流 + golden trace） |

下一步：拿一个功能点去 [02-feature-test-design.md](02-feature-test-design.md) 看"具体怎么测"。
