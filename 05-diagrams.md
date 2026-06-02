# 交付件 5 · 图示总结（四份文档的可视化）

> 用 Mermaid 图表代码表达，可在 GitHub / VS Code / 各类 Markdown 预览器里直接渲染为图片。
> 对应文档：[01](01-architecture-summary.md) / [02](02-agent-service-deep-dive.md) / [03](03-l0-l1-architecture.md) / [04](04-java-refactor-test-plan.md)。
> 已导出静态 PNG 至 [`images/`](images/)（用 `@mermaid-js/mermaid-cli` 渲染，scale 2、白底）。下方「静态图片」直接展示，其后保留 Mermaid 源码便于编辑。

---

## 静态图片（PNG 总览）

### 图 1 · 系统全景：8 模块 × 5 部署面
![系统全景：8 模块 × 5 部署面](images/01-system-landscape.png)

### 图 2 · 智能体端到端流程
![智能体端到端流程](images/02-agent-flow.png)

### 图 3 · agent-service 内部五层与数据流
![agent-service 内部五层与数据流](images/03-agent-service-layers.png)

### 图 4 · Task 状态机 DFA
![Task 状态机 DFA](images/04-task-state-machine.png)

### 图 5 · 三套「L 编号」语义
![三套 L 编号语义](images/05-l0-l1-semantics.png)

### 图 6a · 测试方案：范围 / 接口 / 策略
![测试方案：范围/接口/策略](images/06a-test-plan-scope.png)

### 图 6b · 测试方案：七维测试基线
![测试方案：七维测试基线](images/06b-test-plan-baselines.png)

### 图 6c · 功能点编号图例（A/S/Q/T/E/X 各编号含义）
![功能点编号图例](images/06c-feature-id-legend.png)

---

## Mermaid 源码

## 图 1 · 系统全景：8 模块 × 5 部署面（对应文档 01）

```mermaid
flowchart TB
    subgraph EDGE["Edge Access 面"]
        AC["agent-client<br/>SDK 骨架 W3+ (java=3)"]
    end
    subgraph CC["Compute & Control 面"]
        AS["agent-service ★ (java=142)<br/>access / session / queue / taskcontrol / engine"]
        AEE["agent-execution-engine (java=30)<br/>EngineRegistry / EngineEnvelope"]
        AM["agent-middleware (java=85)<br/>RuntimeMiddleware / HookPoint"]
    end
    subgraph BUS["Bus & State Hub 面"]
        AB["agent-bus (java=29)<br/>ingress / s2c / engine SPI + SuspendSignal"]
        GM["graphmemory-starter"]
    end
    subgraph EVO["Evolution 面"]
        AEV["agent-evolve (java=5)<br/>自进化 骨架"]
    end
    subgraph SBX["Sandbox Execution 面"]
        SX["Sandbox 运行时<br/>W2 设计 (sandbox-policies)"]
    end

    AC -->|"IngressGateway (C2S)"| AB
    AS --> AEE
    AS --> AM
    AS --> AB
    AS -. "S2C 回调" .-> AC
    AB -. "control / data / rhythm 三通道物理隔离" .-> AS
```

> 结论：6 模块已全部 Java 化；agent-service 最完整（★），其余为 SPI/骨架。总线面跨服务流量切成 control（永不阻塞）/ data / rhythm 三通道。

---

## 图 2 · 智能体端到端流程（对应文档 01 §4 / 02）

```mermaid
sequenceDiagram
    actor U as 客户端/外部系统
    participant C as A2aJsonRpcController
    participant G as AccessGateway
    participant TC as TaskControlService
    participant ED as EngineDispatchApi
    participant Q as InternalEventQueue
    participant D as EngineDispatcher
    participant H as OpenJiuwenAgentHandler
    participant OJ as openjiuwen/agent-core-java
    participant N as NotificationPort / Egress

    U->>C: POST /a2a/ (SendMessage / Streaming / Cancel)
    C->>G: A2aEnvelope
    G->>TC: TaskHandler.run(AgentRequest, ReplyContext)
    Note over TC: 幂等去重 + 建/定位 Task(CREATED) + session 锁
    TC->>ED: enqueueExecution(scope{agentId}, input)
    ED->>Q: EngineCommandEvent (publish)
    TC-->>G: AccessAcceptedResponse(taskId)
    G-->>U: 立即 accepted (taskId)，不 hold 连接

    Q->>D: onCommand (subscribe)
    D->>H: AgentHandler.execute(ctx) : Stream<事件>
    H->>OJ: 执行 Agent (LlmAgent/WorkflowAgent/ReActAgent)
    OJ-->>H: 输出 / 中断 / 完成 事件

    par 状态回写链路
        H-->>TC: markRunning/markWaiting/markSucceeded/markFailed/markCancelled
    and 用户输出链路
        H-->>N: appendOutput/completeOutput/failOutput/requestUserInput
        N-->>U: A2A SSE / push notification / async 回推
    end
```

> 两点核心：①入队即返回（异步执行）；②引擎事件分裂成「状态回写」+「用户输出」两条独立链路。

---

## 图 3 · agent-service 内部五层与数据流（对应文档 02）

```mermaid
flowchart TB
    IN["A2A JSON-RPC / async ingress"] --> L1
    subgraph AGS["agent-service（单 Spring Boot 进程 / 一个 jar）"]
        direction TB
        L1["L1 access<br/>AccessGateway · A2aJsonRpcController · EgressDispatcher · NotificationPort"]
        L2["L2 session<br/>SessionManager · SessionStore(InMemory/Redis, CAS)"]
        L3["L3 queue<br/>QueueManager · InternalEventQueue"]
        L4["L4 taskcontrol ★<br/>TaskControlService（Task 状态机 + 幂等 + session 锁 + 派发）"]
        L5["L5 engine<br/>EngineDispatchApi · EngineDispatcher · AgentHandlerRegistry · OpenJiuwen 适配"]
    end
    L1 -->|"TaskHandler.run/cancel"| L4
    L4 -->|"EngineDispatchApi.enqueueExecution/Resume/Cancel"| L5
    L4 <-->|"建/查 session 队列"| L3
    L5 -->|"publish / subscribe"| L3
    L5 -->|"TaskControlClient.mark*"| L4
    L5 -->|"AccessLayerClient.append/complete/fail/requestUserInput"| L1
    L4 -.->|"appendTask 占位"| L2
    L5 --> OJ["openjiuwen/agent-core-java"]
    L1 --> OUT["Egress：A2A SSE/push / async"]
```

> 入口装配：`AgentServiceApplication` 只扫 `access`+`bootstrap`，其余经 AutoConfiguration 导入；`AgentServiceBootstrapConfiguration` 提供 2 个跨层 seam（入站 AccessTaskHandler、出站 AccessNotificationClient）。

---

## 图 4 · Task 状态机 DFA（对应文档 02 §5.1，强测试点）

```mermaid
stateDiagram-v2
    [*] --> CREATED
    CREATED --> RUNNING
    CREATED --> CANCELLING
    CREATED --> FAILED
    RUNNING --> WAITING
    RUNNING --> COMPLETED
    RUNNING --> FAILED
    RUNNING --> CANCELLING
    RUNNING --> CANCELLED
    WAITING --> RUNNING
    WAITING --> FAILED
    WAITING --> CANCELLING
    WAITING --> CANCELLED
    PAUSED --> RUNNING
    PAUSED --> FAILED
    PAUSED --> CANCELLING
    CANCELLING --> CANCELLED
    CANCELLING --> FAILED
    COMPLETED --> [*]
    FAILED --> [*]
    CANCELLED --> [*]
    note right of COMPLETED : terminal（拒绝任何转移）
    note right of CREATED : 同状态自转移=允许（幂等）；revision 乐观锁
```

---

## 图 5 · 三套「L 编号」语义（对应文档 03）

```mermaid
flowchart LR
    subgraph A["① 平台架构分层（用户主要问的）"]
        direction TB
        A0["L0 系统级<br/>系统边界 + §4 65 条约束"]
        A1["L1 模块级<br/>Spring 组合 / HTTP·持久化契约 / posture / 测试 / 证据"]
        A2["L2 深度技术设计"]
        A0 --> A1 --> A2
    end
    subgraph B["② 治理分层 governance"]
        direction TB
        B0["Layer-0 原则 P-A..P-M（13 条）"]
        B1["Layer-1 规则 D / R / G / M（活跃 55 条）"]
        B0 --> B1
    end
    subgraph C["③ agent-service 内部分层"]
        direction TB
        C1["L1 access"]
        C2["L2 session"]
        C3["L3 queue"]
        C4["L4 taskcontrol"]
        C5["L5 engine"]
        C1 --> C2 --> C3 --> C4 --> C5
    end
    B0 -. "enforced_by §4 约束" .-> A0
    A1 == "agent-service 是一个 L1 模块，内含下列子层" ==> C1
```

> 一句话：**L0 定义平台被允许成为什么，L1 定义每个 Spring 模块被允许如何成为它。** L1 不是成熟度标签（成熟度由 `architecture-status.yaml` 的二元 `shipped:` 决定）。

---

## 图 6a · 测试方案：范围 / 接口 / 策略（对应文档 04 §1·§2）

```mermaid
flowchart LR
    ROOT["Java 重构测试方案<br/>范围·接口·策略"]
    ROOT --> F["① 全量功能特性清单<br/>识别范围与重点"]
    ROOT --> I["② 接口文档<br/>支撑接口自动化"]
    ROOT --> ST["四层测试策略"]

    F --> F1["access A1-A10"]
    F --> F2["session S1-S7"]
    F --> F3["queue Q1-Q4"]
    F --> F4["taskcontrol T1-T9 ★"]
    F --> F5["engine E1-E11 ★"]
    F --> F6["跨模块 X1-X7"]

    I --> I1["A2A JSON-RPC（shipped 边）"]
    I --> I2["async ingress"]
    I --> I3["运维 /v1/health · /actuator/*"]
    I --> I4["内部 SPI/API（EngineDispatchApi 等）"]
    I --> I5["⚠️ 契约漂移：/v1/runs vs A2A"]

    ST --> S1["L1 Contract"]
    ST --> S2["L2 Business Activity"]
    ST --> S3["L3 Technical Sub-scenario"]
    ST --> S4["L4 Architecture (ArchUnit)"]
```

## 图 6b · 测试方案：七维测试基线（对应文档 04 §3）

```mermaid
flowchart LR
    BL["③ 测试基线<br/>（7 维）"]
    BL --> B1["性能<br/>SLO p50/p95/p99、吞吐、RLS 开销"]
    BL --> B2["可靠<br/>状态机 / 幂等 / 中断恢复 / 失败回写"]
    BL --> B3["安全<br/>JWT · 租户隔离 · 权限 + 沙箱 subsumption"]
    BL --> B4["兼容<br/>OpenAPI 快照 · A2A SDK · SpringBoot4/Java21/ARM64"]
    BL --> B5["资料<br/>gate Code-as-Contract · DfX · 契约目录"]
    BL --> B6["部署/升级/迁移<br/>Helm · Flyway · runbooks · 回滚开关"]
    BL --> B7["观测<br/>springai_ascend_* · trace · MDC · HookChain"]
```

> 现状标注贯穿全表：`shipped` / `partial` / `stub` / `design_only` / `schema_shipped`，避免把设计目标当已实现。

---

## 图 6c · 功能点编号图例（对应文档 04 §1）

> 命名规则：**字母 = agent-service 内部层**（A=access·S=session·Q=queue·T=taskcontrol·E=engine·X=跨模块），**数字 = 该层第几个功能点**，**★ = 测试重点**。黄色块（taskcontrol / engine）为核心闭环、优先测。

```mermaid
flowchart LR
    ACC["<b>access (L1) · A1-A10</b><br/>A1 A2A SendMessage（JSON）<br/>A2 SendStreamingMessage（SSE）<br/>A3 push notification<br/>A4 GetTask<br/>A5 CancelTask<br/>A6 Agent Card 发现<br/>A7 async 入站消费<br/>A8 入站归一 → AgentRequest<br/>A9 出站分发 Egress<br/>A10 通知端口 notify"]
    SES["<b>session (L2) · S1-S7</b><br/>S1 loadOrCreate/get/exists/delete<br/>S2 appendMessage/putState/putMetadata<br/>S3 appendTask（Task 占位）<br/>S4 内存存储 InMemory<br/>S5 Redis 存储 + TTL + CAS<br/>S6 存储工厂 memory/redis<br/>S7 乐观锁 version / saveIfVersion"]
    QUE["<b>queue (L3) · Q1-Q4</b><br/>Q1 注册/查找 session 队列<br/>Q2 创建内存 session 队列<br/>Q3 offer / snapshot / find<br/>Q4 生产队列 + 至少一次（design_only）"]
    TCC["<b>taskcontrol (L4) · T1-T9 ★</b><br/>T1 提交执行 run<br/>T2 恢复 resume（命中 WAITING）<br/>T3 取消 cancel（terminal 拒绝）<br/>T4 ★ Task 状态机 DFA<br/>T5 状态回写 mark*<br/>T6 ★ 幂等去重（6 元组 key）<br/>T7 session 级并发锁<br/>T8 revision 乐观锁 + stale 拒绝<br/>T9 派发失败 → Task FAILED"]
    ENG["<b>engine (L5) · E1-E11 ★</b><br/>E1 异步入队 API<br/>E2 command 事件构造<br/>E3 队列发布/订阅<br/>E4 ★ 派发 + handler 查找<br/>E5 agentId 缺失 → FAILED<br/>E6 AgentHandler SPI 执行<br/>E7 OpenJiuwen 适配<br/>E8 ★ 事件 → 双链路映射<br/>E9 中断/恢复<br/>E10 Agent 调 Agent INLINE<br/>E11 CHILD_TASK（design_only）"]
    CRS["<b>跨模块 · X1-X7</b><br/>X1 Engine envelope 派发（registry-only）<br/>X2 每引擎声明 hook surface<br/>X3 Middleware + HookPoint 分发<br/>X4 SuspendSignal 中断原语<br/>X5 S2C 回调<br/>X6 IngressGateway（C2S）<br/>X7 运维健康/指标"]
    ACC ~~~ SES ~~~ QUE
    TCC ~~~ ENG ~~~ CRS
    style TCC fill:#fff3cd,stroke:#e0a800
    style ENG fill:#fff3cd,stroke:#e0a800
```

> 合计约 48 个功能点；taskcontrol(T) 与 engine(E) 为核心闭环（★），是功能/可靠性测试的首要对象。明细见 [04-java-refactor-test-plan.md](04-java-refactor-test-plan.md) §1 各表（含可见性 / shipped·stub 现状 / 关键类 / 建议测试层）。

---

## 渲染说明

- 上述图均为 **Mermaid** 代码，GitHub、VS Code（Markdown Preview Mermaid 插件）、Obsidian、Typora 等可直接显示为图片。
- 如需导出为 PNG/SVG 静态图：`npx @mermaid-js/mermaid-cli -i 05-diagrams.md -o diagrams.png`（需 Node 环境）。
- 仓库已有 PlantUML 视图（`docs/architecture/l0/architecture-views/plantuml`）与 Structurizr 权威模型（`architecture/workspace.dsl`），如需与官方视图风格统一可改用 PlantUML 重绘。
