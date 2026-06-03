# 交付件 4 · Java 重构测试方案（总览 / 索引）

> 本文是测试方案的**入口与总览**。详细、面向测试人员、可照抄上手的内容已拆到 **[`04-test-plan/`](04-test-plan/)** 子目录。
> 形态：方案设计文档（含可复制的测试骨架片段，不新增可运行文件）。

---

## 0. 为什么有这一版

第一版测试方案被指出四个问题：策略太简洁、没说"每个特性怎么测/如何快速上手"、接口与基线太难懂、没说明能否脱离框架做 E2E 与自动化。本版据此**重写为面向测试人员的可执行方案**，并把关键事实落到代码实证。

核心结论（详见子目录）：
- **agent-service 能脱离整个框架做单进程端到端测试**：兄弟模块是编译期 SPI、请求链路无远程 RPC、核心件全内存、Agent 可用 fake 替身；`AgentServiceEndToEndIT` 已证明。
- 已有**可复用「测试三件套」**（`FakeInterruptingAgentHandler` / `RecordingTaskControlClient` / `RecordingAccessLayerClient`）与两种装配范式（纯 Java 闭环 / `@SpringBootTest` A2A E2E）。

---

## 1. 测试方案由三块组成（对应背景要求）

| 背景要求 | 落在哪 |
|---|---|
| ① Java 全量功能特性清单（识别测试范围/重点） | 清单见下 §2；**每个特性怎么测**见 [04-test-plan/02-feature-test-design.md](04-test-plan/02-feature-test-design.md) |
| ② 接口文档（接口自动化：功能/性能/可靠/兼容） | [04-test-plan/03-interface-and-cases.md](04-test-plan/03-interface-and-cases.md) |
| ③ 测试基线（性能/可靠/安全/兼容/资料/部署升级迁移/观测） | [04-test-plan/04-test-baselines.md](04-test-plan/04-test-baselines.md) |
| 测试策略 + E2E + 自动化（背景隐含、用户强调） | [01-test-strategy](04-test-plan/01-test-strategy.md) + [05-e2e-and-automation](04-test-plan/05-e2e-and-automation.md) |

子目录导航见 [04-test-plan/README.md](04-test-plan/README.md)（含**新手 30 分钟上手**）。

---

## 2. Java 全量功能特性清单（速览）

按 agent-service 五层 + 跨模块共约 **48 个功能点**；编号与 [05-diagrams.md](05-diagrams.md) 图 6c 一致。每个功能点的「怎么测」逐条展开在 [02-feature-test-design.md](04-test-plan/02-feature-test-design.md)。

| 分组 | 功能点 | 测试重点 | 现状概览 |
|---|---|---|---|
| access (A1–A10) | A2A 四 method + push + 发现 + async + 归一 + 出站 + 通知端口 | A1/A5/A8/A9 | 多 shipped，A7 partial |
| session (S1–S7) | SessionManager + Store(mem/redis) + CAS | S5/S7 | 多 shipped，S5 partial |
| queue (Q1–Q4) | 内存事件队列 | Q1–Q3 | shipped，Q4 design_only |
| **taskcontrol (T1–T9) ★** | Task 状态机 / 幂等 / 乐观锁 / 派发失败回写 | **T4/T6/T8/T9** | shipped（P0 重点） |
| **engine (E1–E11) ★** | 入队 API / 派发 / 双链路映射 / 中断恢复 / OpenJiuwen 适配 | **E4/E5/E8** | 多 shipped，E2/E7 partial/stub，E11 design_only |
| 跨模块 (X1–X7) | envelope 派发 / hook / SuspendSignal / S2C / Ingress / 运维 | X1/X4/X7 | 多 shipped/SPI |

**测试重点结论**：核心闭环 **taskcontrol(T) 与 engine(E)** 为 P0；高优先级回归风险区是「A2A 边（代码）vs `/v1/runs`（openapi-v1.yaml）契约漂移」（详见 [03](04-test-plan/03-interface-and-cases.md) §4）。

---

## 3. 四层测试策略（速览）

L0 架构/静态(ArchUnit) → L1 单元/白盒 → L2 组件/闭环集成(纯 Java 三件套) → L3 Spring E2E(A2A) → L4 契约/基线。完整说明 + 决策表 + 测试诚实规则见 [01-test-strategy.md](04-test-plan/01-test-strategy.md)。复用平台四层策略 `docs/architecture/l0/09-verification/test-strategy.md`。

---

## 4. 七维测试基线（速览）

性能 / 可靠 / 安全(权限+沙箱) / 兼容 / 资料 / 部署升级迁移 / 观测。每维「测什么/怎么测/通过标准/现状」详见 [04-test-baselines.md](04-test-plan/04-test-baselines.md)。
**关键纪律**：区分 `shipped`（可强制）与 `design_only/schema_shipped`（仅目标值），不把设计目标当已实现。

---

## 5. 复用资产（引用，不重造）

- 测试夹具：`agent-service/src/test/.../engine/support/{FakeInterruptingAgentHandler,RecordingAccessLayerClient,RecordingTaskControlClient}.java`
- 装配范式：`.../engine/EngineClosedLoopIntegrationTest.java`、`.../bootstrap/AgentServiceEndToEndIT.java`、`.../taskcontrol/test/TaskControlServiceWhiteboxTest.java`
- 既有基线/契约：`docs/architecture/l0/09-verification/test-strategy.md`、`.../cross-cutting/v1.0-perf-baselines.yaml`、`docs/governance/sandbox-policies.yaml`、`docs/dfx/agent-service.yaml`、`docs/contracts/http-api-contracts.md`、access-layer L1 设计稿。

---

## 阅读延伸

- 测试人员入口（强烈建议从这里开始）→ [04-test-plan/README.md](04-test-plan/README.md)
- 整体架构 → [01-architecture-summary.md](01-architecture-summary.md)；agent-service 详解 → [02-agent-service-deep-dive.md](02-agent-service-deep-dive.md)
