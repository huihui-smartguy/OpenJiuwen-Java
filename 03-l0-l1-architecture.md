# 交付件 3 · L0 / L1 架构定义总结

> 范围：厘清本项目里「L 编号」的含义。
> 核心结论：**项目里存在三套互不相同的「L 编号」语义**，用户主要问的是其中第一套——**平台架构分层 L0 / L1**。本文先给结论对照，再逐套展开。

---

## 0. 速览：三套 L 编号语义对照

| 体系 | L0 | L1 | L2+ | 权威来源 |
|---|---|---|---|---|
| **① 平台架构分层**（用户主要问的） | 系统级架构层：声明式系统边界 + 65 条编号约束 | 模块级架构层：把 L0 落成 Spring 组合/契约/测试 | L2 = 深度技术设计 | `architecture/docs/L0/ARCHITECTURE.md`、`architecture/docs/L1/`，ADR-0068 |
| **② 治理分层**（governance） | Layer-0 治理原则 `P-A..P-M`（13 条） | Layer-1 工程规则 `D-/R-/G-/M-`（活跃 55 条） | — | `CLAUDE.md`、`docs/governance/principles/`、`docs/governance/rules/` |
| **③ agent-service 内部分层** | （无 L0） | L1=access / L2=session / L3=queue / L4=taskcontrol / L5=engine | — | 三份 `docs/architecture/l1/2026-05-30-*` 设计稿 |

> 三者**不要混用**。例如「agent-service 是 L1 模块」（体系①）与「access 层是 agent-service 内部的 L1」（体系③）是两个不同层级的「L1」。下文 §1 是用户问题的核心答案。

---

## 1. 体系① 平台架构分层 L0 / L1（核心答案）

### 1.1 一句话定义

> **L0 定义平台被允许成为什么；L1 定义每个 Spring 模块被允许如何成为它。**
> （原文：*"L0 defines what `spring-ai-ascend` is allowed to become. L1 defines how each Spring module is allowed to become it."* — `docs/architecture/l1/2026-05-13-l1-architecture-design-guidance.en.md` §17）

### 1.2 L0 = 系统级架构层（declarative）

`architecture/docs/L0/ARCHITECTURE.md`（约 1021 行，`freeze_id: W1-russell-...`，权威 ADR-0068）的自我定位（§0.6 原文）：

> *"This document is the **declarative L0** system boundary + 65 numbered architectural constraints (§4 #1..#65). It states what the platform commits to **STRUCTURALLY**."*

L0 承载（且**仅**承载）以下结构性承诺：

| L0 内容 | 4+1 视图 | 说明 |
|---|---|---|
| §1 系统边界（system boundary） | scenarios | target 架构 vs W0 shipped subset；audience A/B/C 边界 |
| §0.5 跨切面 verticals | logical / process | `TenantContext`（租户）、`APP_POSTURE`（posture）、`TraceContext`（telemetry） |
| §2 模块布局（module layout） | development | 8 模块 × 5 plane 分解 |
| §3 威胁模型（threat model） | physical | 信任边界 + sandbox 拓扑 |
| §4 #1..#65 架构约束 | scenarios | **65 条编号约束语料库**（每条 #N 在图里带自己的视图） |
| §5 分波 rollout（staged rollout） | scenarios | wave 计划（W0→W4） |

L0 **不承载**：enforcement 逻辑（在 `CLAUDE.md` + 规则卡）、runtime 契约（在 `docs/contracts/`）、模块如何实现（在 L1）、能力 shipped/deferred 账本（在 `architecture-status.yaml`）。

### 1.3 L1 = 模块级架构层（design）

`docs/architecture/l1/2026-05-13-l1-architecture-design-guidance.en.md` §1 的定义（原文）：

> *"Module-level architecture that converts L0 decisions into Spring Boot composition, HTTP contracts, persistence contracts, posture behavior, tests, and evidence."*

`architecture/docs/L1/README.md` 给出 L1 的修辞立场：描述每个模块的**设计**（职责、边界契约、SPI 面、关键不变量），回答「这个模块被设计来做什么」。每个活跃 Maven 模块一个 L1 入口（`architecture/docs/L1/<module>/` 目录或 `.md`）。

L1 设计宪章（同一份 guidance §5）要求每份 L1 文档回答 10 问：模块拥有什么 / 显式不拥有什么 / 暴露什么公共契约 / 有哪些 Spring bean / 哪个 `@Configuration` 构造它 / 哪些资源是 durable/request/run/in-memory / dev·research·prod 各变什么 / 什么测试证明行为 / 哪一行 `architecture-status.yaml` 拥有 shipped 声明 / 哪个 wave 拥有 deferred 行为。

### 1.4 ⚠️ 重要纠偏：L1 不是「成熟度」标签

旧的 **L0–L4 成熟度模型**（数字越大越成熟）**已被废弃**，被 `architecture-status.yaml` 的二元 `shipped:` 真值模型取代。guidance §1 明确：

> *"`AGENTS.md` states that the older L0-L4 maturity model has been replaced by the binary `shipped:` truth model … L1 should not be treated as a maturity label."*

所以本项目语境里：**L0/L1/L2 = 架构「层级」（系统级/模块级/深度技术），不是「成熟度等级」。** 一个能力是否「成熟/已发布」由 `architecture-status.yaml#capabilities` 的 `shipped: true/false` 决定，与 L 数字无关。

### 1.5 L2 = 深度技术设计

`architecture/docs/L2/`（W8，由 `docs/L2/` 迁入，ADR-0150）承载比模块设计更细的技术深潜，超出本次范围。

---

## 2. 体系② 治理分层 Layer-0 / Layer-1（governance）

`CLAUDE.md` 与 README「Legacy reading path」描述了另一组「Layer-0 / Layer-1」，指**治理规则的层级**，而非架构文档层级：

- **Layer-0 治理原则 `P-A..P-M`**（13 条 governing principles）：最高层、稳定的协作/设计原则。例：**P-A**（Business / Platform Decoupling，业务只能通过 SPI + 配置扩展平台，禁止改平台内部）、**P-I**（五 plane 拓扑）。存于 `docs/governance/principles/P-*.md`。
- **Layer-1 工程规则 `D-/R-/G-/M-` 命名空间**（活跃 55 条 engineering rules）：可强制、更细。例：`D-1`（根因+最强解读）、`D-4`（三层测试）、`R-M.a`（引擎只经 registry 派发）、`R-J`（存储级租户隔离）、`G-7`（Linux-first）。每条规则回引它强制的 §4 约束（映射在 `docs/governance/enforcers.yaml`）。

> 注意命名巧合：体系② 的「Layer-0/Layer-1」与体系① 的「L0/L1」名字像但所指不同——前者是**规则/原则的层级**，后者是**架构文档的层级**。两者通过「§4 约束 ↔ 规则」交叉引用绑定（`enforcers.yaml`）。

来源：[CLAUDE.md](../../../CLAUDE.md)、`docs/governance/principles/`、`docs/governance/rules/`、`docs/governance/enforcers.yaml`。

---

## 3. 体系③ agent-service 内部分层 L1–L5

这是**模块内部**的层编号，仅在 agent-service 的三份 2026-05-30 L1 设计稿里出现，用于描述模块自身的五层切分：

| 内部层 | 名称 | 代码包 |
|---|---|---|
| L1 | access-layer | `service.access` |
| L2 | session-task-manager | `service.session` |
| L3 | internal-event-queue | `service.queue` |
| L4 | task-centric-control | `service.taskcontrol` |
| L5 | engine | `service.engine` |

> 它与体系① 的关系：整个 `agent-service` 在体系① 里是**一个 L1 模块**；而 access/session/queue/taskcontrol/engine 是该 L1 模块**内部**的 L1–L5 子层。措辞时务必带上下文（「平台 L1 模块 agent-service」vs「agent-service 内部 L1 access 层」）。

详见 [02-agent-service-deep-dive.md](02-agent-service-deep-dive.md)。

---

## 4. 4+1 视图模型（贯穿 L0/L1 的横切组织方式）

ADR-0068 引入 **Layered 4+1** 视图，L0 文档每一节都被分类到一个视图：

| 视图 | 关注 |
|---|---|
| logical | 领域概念（如 `TenantContext`） |
| development | 包/模块分解 |
| process | 启动时/运行时行为、posture fail-closed |
| physical | 信任边界、sandbox 拓扑、部署面 |
| scenarios | 系统边界、约束语料、BA-* 业务活动场景（+1 把其余四视图串起来） |

机器可读的权威根是 `architecture/workspace.dsl`（Structurizr DSL，ADR-0147 + ADR-0150）；L0/L1 markdown 是其人类可读切片。

---

## 5. 一图速记

```text
平台架构（体系①）          治理（体系②）              agent-service 内部（体系③）
─────────────────         ─────────────────         ──────────────────────────
L0 系统边界 + §4×65   ◄──enforced_by──  P-A..P-M (Layer-0 原则)
   │ 被允许成为什么                       D/R/G/M  (Layer-1 规则)
   ▼ converts into
L1 模块设计（每模块一份）                              agent-service = 一个 L1 模块
   · agent-service ───────────────────────────────►   ├ L1 access
   · agent-bus / -engine / -middleware / ...           ├ L2 session
   ▼                                                    ├ L3 queue
L2 深度技术设计                                          ├ L4 taskcontrol
                                                        └ L5 engine
```

来源：[architecture/docs/L0/ARCHITECTURE.md](../../../architecture/docs/L0/ARCHITECTURE.md)、[architecture/docs/L1/README.md](../../../architecture/docs/L1/README.md)、[L1 设计指引](../../architecture/l1/2026-05-13-l1-architecture-design-guidance.en.md)、[CLAUDE.md](../../../CLAUDE.md)、ADR 0064/0068/0069/0138/0155。
