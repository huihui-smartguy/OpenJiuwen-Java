# 03 · 接口文档（易懂版）+ 接口自动化用例

> 目标：让你**不用翻源码**就知道——这个接口干什么、请求长什么样、期望什么响应、错误怎么报、自动化怎么断言。
> 每个接口一张「卡片」，五栏固定：**作用 | 最小请求 | 期望响应 | 常见错误 | 自动化断言点**。

---

## 0. 先记住三件事

1. **对外只有一个 HTTP 入口**：`POST /a2a/`（A2A JSON-RPC，靠 `method` 区分动作）+ 一个发现入口 `GET /.well-known/agent-card.json`。**没有** `/v1/runs`（见 §4 契约漂移）。
2. **入队即返回**：发请求会**立刻**拿到 accepted（带 `taskId`），真正的执行结果**异步**通过出站（SSE/push/轮询 GetTask）拿。
3. **租户与幂等靠头**：可变请求带 `X-Tenant-Id`(UUID)；POST/PUT/PATCH 带 `Idempotency-Key`(UUID)。错误统一是 `ContractError{code,message,traceId}`。

---

## 1. A2A 对外接口（shipped，以此为准）

公共地址：`POST http://<host>:8080/a2a/`，Header `Content-Type: application/json`，`Accept: application/json`（同步）或 `text/event-stream`（流式）。
字段全表（必填/类型/允许值/内部映射）见 access-layer L1 设计稿：`docs/architecture/l1/2026-05-30-l1--agent-service-access-layer-design.md` §5.1，本卡片给"最小可用"。

### 卡片 · SendMessage（普通对话）
| 栏 | 内容 |
|---|---|
| **作用** | 提交一轮用户输入，异步驱动 Agent，返回 accepted+taskId |
| **最小请求** | 见下方 JSON |
| **期望响应** | HTTP 200；`result.role=ROLE_AGENT`；`result.taskId` 非空 |
| **常见错误** | 400（JSON-RPC 外壳/字段非法）、缺 `tenant`/`metadata.userId/agentId/sessionId` 视实现报错 |
| **自动化断言点** | 状态 200；`taskId` 非空；随后轮询出站终帧 `terminal=true` |

```json
{
  "jsonrpc": "2.0", "id": "req-1", "method": "SendMessage",
  "params": {
    "tenant": "tenant-001",
    "message": {
      "role": "ROLE_USER", "messageId": "msg-1", "contextId": "session-1",
      "parts": [{ "kind": "text", "text": "帮我规划三天上海行程" }],
      "metadata": { "tenantId":"tenant-001","userId":"user-1","agentId":"travel-agent","sessionId":"session-1","idempotencyKey":"idem-1" }
    }
  }
}
```

### 卡片 · SendStreamingMessage（SSE 流式）
| 栏 | 内容 |
|---|---|
| **作用** | 同 SendMessage，但响应是 SSE 多帧 |
| **最小请求** | 同上，`method` 改 `SendStreamingMessage`，`Accept: text/event-stream` |
| **期望响应** | 200；`Content-Type` 含 `text/event-stream`；首帧 accepted，后续 working，末帧 completed/terminal |
| **常见错误** | 流中断、agent 异常→末帧应为 error 而非挂起 |
| **自动化断言点** | `Content-Type` 含 SSE；帧序列首 accepted、末 terminal |

### 卡片 · SendMessage + Push Notification（回调）
| 栏 | 内容 |
|---|---|
| **作用** | 立即 accepted，后续事件 POST 到你给的 callback URL |
| **最小请求** | 在 `params.configuration.taskPushNotificationConfig` 放 `{url, token}` |
| **期望响应** | 200 accepted；callback 服务随后收到 POST（头含 `X-A2A-Notification-Token`） |
| **常见错误** | callback 5xx（按实现重试/记录） |
| **自动化断言点** | WireMock 起 callback 桩，`verify(postRequestedFor(...))` |

### 卡片 · GetTask（查任务）
| 栏 | 内容 |
|---|---|
| **作用** | 按 taskId 聚合返回 SDK `Task`（status/history/artifacts） |
| **最小请求** | `method=GetTask`，`params.id`=taskId，`params.metadata.{tenantId,sessionId}` |
| **期望响应** | 200；`result` 为聚合后的 Task |
| **常见错误** | 任务不存在/跨租户→空或错误 |
| **自动化断言点** | `result.status` 与历史/产物存在 |

### 卡片 · CancelTask（取消）
| 栏 | 内容 |
|---|---|
| **作用** | 取消运行中任务（转 `AccessOperation.CANCEL`） |
| **最小请求** | `method=CancelTask`，`params.id`=taskId，`params.metadata.{tenantId,userId,agentId,sessionId}` |
| **期望响应** | 200；任务进入取消链路 |
| **常见错误** | terminal 任务→拒绝（见 [02](02-feature-test-design.md) T3） |
| **自动化断言点** | 任务状态进入 CANCELLING/CANCELLED；无新增用户输出 |

### 卡片 · Agent Card 发现
| 栏 | 内容 |
|---|---|
| **作用** | `GET /.well-known/agent-card.json` 返回标准 AgentCard |
| **期望响应** | 200 + 合法 AgentCard JSON |
| **自动化断言点** | 200；关键字段存在 |

---

## 2. async ingress（异步入站信封，非 A2A）

| 栏 | 内容 |
|---|---|
| **作用** | 外部队列生产者投递 `AsyncEnvelope`，L1 消费后归一为 `AgentRequest` |
| **最小请求** | `{headers:{tenantId,userId,agentId,sessionId,operation,idempotencyKey,correlationId,replyTopic}, body:{query,payload}}` |
| **期望** | 创建任务并驱动链路；`replyTopic` 成为出站 `targetRef` |
| **第一版** | 建议只用 `SUBMIT`/`CANCEL` |
| **自动化断言点** | 调 `AccessGateway.submitAsync(envelope)`，断言任务创建 + 出站绑定（无需真 MQ） |

---

## 3. 运维接口

| 接口 | 作用 | 期望 | 断言点 |
|---|---|---|---|
| `GET /v1/health` | 存活探针 | 200；`{status,sha,db_ping_ns,ts}` | `status=UP` |
| `GET /actuator/health` | Spring 健康 | 200；含依赖检查 | `status=UP` |
| `GET /actuator/prometheus` | 指标抓取 | 200；文本含 `springai_ascend_*` | 指标前缀存在 |

> 这三个**免租户/幂等头**；research/prod 应限内网访问。

---

## 4. ⚠️ 契约漂移：openapi-v1.yaml 的 /v1/runs

`docs/contracts/openapi-v1.yaml` + `http-api-contracts.md` 描述了 `POST /v1/runs`、`GET /v1/runs/{id}`、`POST /v1/runs/{id}/cancel`（Run + TaskCursor 形态），**但重构后的代码并未暴露 `/v1/runs`，对外是 A2A 边**。

**测试口径：**
- 功能/接口自动化**以 A2A（本章 §1）为准**。
- 增加一条**契约漂移回归用例**：请求 `POST /v1/runs` → 期望 404/未路由，证明"文档声称 shipped 但代码无对应路由"，推动后续在 `architecture-status.yaml` 校正 shipped 真值。
- OpenAPI 快照测试（`OpenApiContractIT`/`ApiCompatibilityTest`）继续防止**已暴露**接口漂移。

---

## 5. 内部 SPI/API 接口（模块级契约测试用）

这些不是 HTTP，是 Java 接口；做模块集成时按契约测。

| 接口 | 方向 | 关键方法 | 契约要点（断言什么） |
|---|---|---|---|
| `EngineDispatchApi` | task-control→engine | `enqueueExecution/Resume/Cancel` | 只入队，返回 `SUCCESS/FAILED`；不传 handler/框架类型 |
| `TaskControlClient`(engine.port) | engine→task-control | `markRunning/Waiting/Succeeded/Failed/Cancelled` | 状态意图回写 |
| `AccessLayerClient`(engine.port) | engine→access | `appendOutput/completeOutput/failOutput/requestUserInput` | 用户可见输出 |
| `TaskControlClient`(taskcontrol.api) | access→task-control | `run/resume/cancel/mark*` | 返回 `CompletionStage<TaskResult>` |
| `SessionManager` | 任意→session | `loadOrCreate/appendMessage/...` | 只经此入口访问 Session |
| `AgentHandler` | engine SPI | `agentId/isHealthy/execute` | 返回 `Stream<EngineExecutionEvent>` |
| `NotificationPort` | 下游→access | `notify(NotificationFrame)` | 按 tenant+session+task 入队 |

> 这些接口的"怎么测"已在 [02](02-feature-test-design.md) 逐特性给出（用三件套/录参 fake）。

---

## 6. 接口自动化骨架（可照抄）

### 6.1 走真实 HTTP `/a2a/`（REST-assured 风格）
> 需在 `@SpringBootTest(webEnvironment = RANDOM_PORT)` 下，复用 [05](05-e2e-and-automation.md) 的 `TestRuntime` + fake agent。

```java
@SpringBootTest(classes = TestRuntime.class, webEnvironment = WebEnvironment.RANDOM_PORT)
class A2aHttpContractIT {
    @LocalServerPort int port;

    @Test
    void sendMessage_returns200WithTaskId() {
        given().port(port)
            .contentType("application/json")
            .body(SEND_MESSAGE_JSON)              // §1 的最小请求 JSON
        .when()
            .post("/a2a/")
        .then()
            .statusCode(200)
            .body("result.taskId", notNullValue());
    }

    @Test
    void legacyRunsRoute_isNotExposed() {          // §4 契约漂移回归
        given().port(port).contentType("application/json").body("{}")
        .when().post("/v1/runs")
        .then().statusCode(anyOf(is(404), is(405)));
    }
}
```

### 6.2 不起 HTTP，直接调 access service（更快）
```java
@Autowired A2aAccessService a2a;
A2aAcceptedResponse accepted = a2a.send(envelope("session-1", "hello"));
assertThat(accepted.taskId()).isNotBlank();   // 出站轮询见 AgentServiceEndToEndIT
```

### 6.3 WireMock 打桩外部依赖（callback / LLM）
```java
// callback（push notification）
WireMockServer wm = new WireMockServer(9001);
wm.start();
wm.stubFor(post("/a2a/callback").willReturn(ok()));
// ... 发 push 请求 ...
wm.verify(postRequestedFor(urlEqualTo("/a2a/callback"))
        .withHeader("X-A2A-Notification-Token", matching(".+")));

// LLM 网关：把 spring.ai.openai.base-url 指向 WireMock，stub /v1/chat/completions 返回固定 JSON
```

> WireMock、REST-assured 已在 `agent-service/pom.xml` 的 test scope，直接用。
