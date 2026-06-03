# 04 · 测试基线（易懂版）

> 目标：把晦涩的 YAML 翻译成测试人员能**直接执行的检查清单**。
> 每条基线七栏固定：**测什么 | 为什么重要 | 怎么测(步骤) | 通过标准 | 现状 | 工具/来源 | 示例**。

---

## 0. 先看懂「现状」标签（最重要）

很多基线目前只是**设计目标**，还没在代码里强制。做测试前必须分清，否则会"测一个根本没实现的东西"：

| 标签 | 含义 | 你能做的测试 |
|---|---|---|
| `shipped` | 已实现、可运行强制 | 写真实断言，纳入回归 |
| `schema_shipped` | 只落了 schema/默认值，运行期强制未做 | 测"配置良构 + 默认值 + 越权识别"，**不**测运行期强制 |
| `design_only` | 仅文档目标，未仪表化 | 只做"目标值文档 + 压测/检查脚手架"，**不**声称达标 |
| `stub` | 代码占位（可能返回 null） | 用 fake 隔离，标注待补 |

七维基线里：**可靠 / 安全(权限) 多为 shipped**；**性能 / 沙箱运行期 / RLS / Hook / 审计 多为 design_only 或 schema_shipped**。

---

## 1. 性能（Performance）

来源：`docs/architecture/l0/cross-cutting/v1.0-perf-baselines.yaml`（`status: design_only`）+ `perf/`（JMH、`baseline-2026-05-10.md`、2× 回归策略）。参考硬件：Kunpeng 920 ARM64 4 核/8GiB + Ascend 310。

| 子项 | 测什么 | 为什么重要 | 怎么测 | 通过标准 | 现状 | 工具 |
|---|---|---|---|---|---|---|
| 入口入队延迟 | `POST /a2a/`（或入队 API）的响应延迟 | 入队即返回是核心承诺，不能 hold 连接 | 固定流量 profile 压测，采集时延直方图 | p50≤100ms / p95≤500ms / p99≤1000ms | design_only | JMeter/k6 + Micrometer |
| 工具调用延迟 | 内部/外部 tool 调用 p95 | 决定整体响应 | 按 skill_class 分组打点 | 内部≤100ms / 外部≤1000ms (p95) | design_only | `springai_ascend_tool_call_seconds` |
| 模型调用 | 首 token / 流式每 token | 用户体感 | 分 provider 打点 | ascend_local 首 token p95≤2000ms；网关开销 p95≤100ms | design_only | `springai_ascend_model_invoke_seconds` |
| 吞吐 | 单 pod 并发在飞 Run 数 | 容量规划 | 阶梯加压 | 100 并发/pod；20 持续/50 突发 入队/s | design_only | 压测 |
| 租户隔离开销 | 开/关 RLS 的延迟差 | 证明 RLS 可接受 | 同数据集对比基准 | p95 延迟开销≤5%，CPU≤3% | design_only | Testcontainers PG |
| 单 agent 内存 | 常驻内存 | 防泄漏/容量 | JVM 采样 | p95≤512MB / p99≤768MB | design_only | JFR/采样 |
| 回归红线 | 任一指标是否劣化 | 防性能退化 | 与基线比 | 超 2× baseline 即失败 | partial | `perf/README.md` + JMH |

**现在能做**：先把 `springai_ascend_*` 计时器仪表化（`promotion_trigger` 前置），再上回归 IT；当前阶段做"目标值文档 + 压测脚本脚手架"，不声称达标。

---

## 2. 可靠性（Reliability）★ 大部分 shipped，重点测

来源：代码（状态机/幂等/并发）+ `docs/dfx/agent-service.yaml#resilience/availability`。

| 子项 | 测什么 | 怎么测 | 通过标准 | 现状 |
|---|---|---|---|---|
| Task 状态机 | 非法转移拒绝、terminal 不可转、同态幂等 | L2 白盒穷举转移矩阵（见 [02](02-feature-test-design.md) T4） | 全部非法转移 `accepted=false` | shipped |
| 幂等去重 | 同 key 只执行一次 | 并发发同 key（[02](02-feature-test-design.md) T6） | 仅 1 次副作用 | shipped |
| 取消语义 | 非 terminal→取消；terminal→拒绝；engine reject→FAILED | L2（[02](02-feature-test-design.md) T3/T9） | 状态正确、有失败码 | shipped |
| 中断恢复 | WAITING→resume→完成 | `FakeInterruptingAgentHandler`（[02](02-feature-test-design.md) E9） | 两轮序列正确 | shipped |
| Agent 异常不悬挂 | handler 抛异常仍出 terminal | `ThrowingAgentHandler`（E6） | 末帧 terminal+error、无泄漏 | shipped |
| 出站不泄漏 | 终帧后队列清理 | `awaitEgressCleanup`（A9） | `find()` 为空 | shipped |
| posture 启动 | research/prod 缺配置即失败 | 不同 `APP_POSTURE` 启动 | 缺配置→`IllegalStateException` | shipped |
| readiness/Resilience4j | —（W2） | — | — | design_only |

**通过标准（整体）**：上述 shipped 项必须全绿，且失败/边界路径有专门用例（测试诚实规则 #2）。

---

## 3. 安全（权限 + 沙箱）

来源：`docs/dfx/agent-service.yaml#vulnerability` + `docs/governance/sandbox-policies.yaml`（`status: schema_shipped`）+ posture-model + ArchUnit。

### 3.1 权限/认证/隔离（多为 shipped）
| 子项 | 测什么 | 怎么测 | 通过标准 | 现状 |
|---|---|---|---|---|
| JWT 校验 | RS256 + iss/aud/exp/nbf/skew | spring-security-test 注入合法/过期/错 aud token | 非法→401 | shipped(W1) |
| 租户交叉校验 | `X-Tenant-Id` ↔ JWT `tenant_id` | filter chain 集成测试 | 不匹配→403 | shipped(W1) |
| 租户数据隔离 | 跨租户资源表现为 404（无存在 oracle） | 跨租户访问 + ArchUnit（runtime 不 import platform） | 跨租户→404 | shipped |
| secret 卫生 | 源码无 AKIA/sk-/私钥；日志无 raw JWT/body | gate `no_secret_patterns` + 日志审查 | 0 命中 | shipped |
| SPI 纯净 | `*.spi.*` 只 import `java.*` | ArchUnit | 无越界 import | shipped |
| Postgres RLS | tenant_id 谓词在存储层 | Testcontainers + 注入 | 跨租户行不可见 | design_only(W2) |

### 3.2 沙箱（Sandbox Permission Subsumption，schema_shipped）
| 子项 | 测什么 | 怎么测 | 通过标准 | 现状 |
|---|---|---|---|---|
| 默认 deny | 未声明 skill：零出网/零 FS + 资源上限 | 校验 `default_policy` 六键齐备 | schema 良构、默认 deny | schema_shipped |
| `financial_default` | 出网默认 deny+白名单、FS 限 scratch、PII 不出 model | 配置加载 + 越权配置应被拒 | 越权配置识别 | schema_shipped |
| per-skill 白名单 | model-call/memory-write 出网目的地 | 白名单匹配 | 仅白名单可达 | schema_shipped |
| 逻辑≤物理 | SandboxExecutor 拒绝过宽逻辑授权 | （W2 落地后）subsumption_check | 运行期强制 | design_only(W2) |

> **重点**：沙箱**运行期强制是 design_only**；当前只测「策略 schema 良构 + 配置加载 + 越权识别」，**不**声称 runtime 强制。

---

## 4. 兼容性（Compatibility）

| 子项 | 测什么 | 怎么测 | 通过标准 | 现状 |
|---|---|---|---|---|
| OpenAPI 漂移 | live vs `openapi-v1.yaml` | `OpenApiContractIT`/`ApiCompatibilityTest` | 无非预期漂移 | shipped（注意 §[03](03-interface-and-cases.md) 的 /v1/runs 分叉） |
| A2A SDK 版本 | `a2a-java-sdk-server-common:1.0.0.CR1` 类型 | request/response 回归 | 类型校验通过 | shipped |
| 平台版本 | Spring Boot 4 / Java 21 / ARM64 原生 | 在 ARM64 跑全套（Rule G-7） | 全绿 | shipped |
| SPI 兼容 | `service.runtime.*.spi.*` L1 冻结 | SemVer + ArchUnit | 无破坏性变更 | shipped |
| OpenJiuwen 依赖 | `agent-core-java:0.1.7` 集成 | 依赖解析 + 适配器集成 | 可解析、适配通过 | partial |

---

## 5. 资料/文档（Documentation）

| 子项 | 测什么 | 怎么测 | 通过标准 | 现状 |
|---|---|---|---|---|
| Code-as-Contract gate | 文档↔代码 lockstep | `gate/check_architecture_sync.sh`（`-Pquality verify`） | gate 通过、无漂移 | shipped |
| DfX 必备性 | 每个 domain 模块有五维 DfX | gate `dfx_yaml_present_and_wellformed` | 存在且良构 | shipped |
| 契约目录 | 40+ 契约 YAML + catalog | catalog 校验 | 完整 | shipped |
| baseline 计数 | 65 §4 / 64 ADR / 35 gate 规则… | gate self-tests | 计数一致 | shipped |

---

## 6. 部署 & 升级 & 迁移（Deploy / Upgrade / Migration）

来源：`deploy/middle-office-reference/`（Helm）+ Flyway + `ops/runbooks/`。

| 子项 | 测什么 | 怎么测 | 通过标准 | 现状 |
|---|---|---|---|---|
| Helm 部署 | agent-service/agent-bus/postgres-rls/sandbox/observability 模板 | `helm lint`/`template` + kind 冒烟 | 渲染/部署成功 | shipped(参考) |
| 资源 sizing | resources 与 perf 基线一致 | values.yaml 与 §1 对照 | 一致 | design_only |
| DB 迁移 | `V1__init.sql`/`V2__idempotency_dedup.sql` | Testcontainers PG 全量迁移 + 幂等重跑 | 迁移成功、可重跑 | shipped |
| 升级/回滚 | engine 回滚开关 `agent-service.engine.enabled=false` | 切换开关 + runbook 演练 | 拒新 command、保留未消费事件 | shipped |
| 灾备/事件 | runbooks 可用 | 走查 dr/rollback/incident/credential-loss | 步骤可执行 | shipped(文档) |
| 健康检查 | 五层启动健康项 | 启动后查健康 | 各项 healthy | shipped |

> 健康检查项（engine 设计稿 §15.5）：`EngineQueueGateway` 可用、`EngineCommandSubscriber` running、`AgentHandlerRegistry` ≥1 注册、handler `isHealthy=true`、两个 client 可用。

---

## 7. 观测指标（Observability）

来源：`docs/dfx/agent-service.yaml#observability` + `ARCHITECTURE.md §0.5.3` + `ops/.../observability-stack`。

| 子项 | 测什么 | 怎么测 | 通过标准 | 现状 |
|---|---|---|---|---|
| 指标命名空间 | `springai_ascend_*` 前缀 | 抓 `/actuator/prometheus` 断言前缀 | 前缀存在 | shipped |
| 高基数防护 | tenant_id 等不进原始标签 | 指标标签审查 | 无高基数标签 | partial(W4 wiring) |
| trace 传播 | W3C `traceparent` 提取/发起 | `TraceExtractFilter` 集成测试 | 出站含 traceresponse | shipped |
| 日志字段 | MDC 带 tenant_id/trace_id/span_id/run_id | 日志结构断言 | 字段齐全、无 secret | shipped |
| Hook 发射 | LLM/工具/生命周期经 HookChain | ArchUnit 守卫（design-only） | — | design_only(W2) |
| 审计行 | run_state_change 审计 | — | — | design_only(W2) |
| 关键失败计数 | JWT 失败/租户不匹配/幂等冲突… | 触发后抓指标 | 计数增长 | partial |

---

## 8. 一页执行清单（按现状排优先级）

**先测（shipped，能强制）**：可靠(§2 全部)、安全权限(§3.1 JWT/租户/secret/SPI)、兼容 OpenAPI、资料 gate、部署迁移(Flyway/Helm/回滚)、观测前缀/trace/MDC。

**再做脚手架（design_only/schema_shipped，只做目标值校验）**：性能 SLO(§1)、沙箱运行期(§3.2 逻辑≤物理)、RLS、Hook/审计。

**报告纪律**：每条结论标 `shipped/schema_shipped/design_only/stub`；`design_only` 不写"已达标"，写"目标值=X，仪表化后回归"。
