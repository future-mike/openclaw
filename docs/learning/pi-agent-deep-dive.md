# Pi Agent 深入学习文档

本文档面向想搞清楚 **"Pi 在 OpenClaw 里到底做什么、怎么做的、边界在哪"** 的读者。

---

## 一、Pi 是什么

Pi 是 OpenClaw 内嵌的 **Agent 引擎**，由三个 npm 包提供：

| 包名                            | 职责                                                                                        |
| ------------------------------- | ------------------------------------------------------------------------------------------- |
| `@mariozechner/pi-agent-core`   | Agent 核心协议：AgentTool、AgentMessage、AgentEvent 等类型定义                              |
| `@mariozechner/pi-ai`           | 模型调用层：`streamSimple` 等流式 API，屏蔽不同 provider 差异                               |
| `@mariozechner/pi-coding-agent` | 编码 Agent 运行时：SessionManager（会话/transcript 管理）、`createAgentSession`、资源加载等 |

OpenClaw 把 Pi 以 **embedded（内嵌）** 方式跑在 Gateway 同一进程里，不是独立服务。

---

## 二、Pi 在整体架构中的位置

```
用户消息 → Gateway (chat.send / agent RPC)
                │
                ▼
        ┌─────────────────────────────────┐
        │  dispatchInboundMessage          │  Gateway 层
        │  → getReplyFromConfig            │  （路由、命令、配置）
        │  → agentCommandFromIngress       │
        └──────────────┬──────────────────┘
                       │
                       ▼
        ┌─────────────────────────────────┐
        │  runEmbeddedPiAgent              │  ← Pi 运行时入口
        │  ├─ 入队 (session lane 串行)      │
        │  ├─ 解析 model / auth / hooks     │
        │  └─ runEmbeddedAttempt           │  ← 单次尝试
        │      ├─ 构建 system prompt        │
        │      ├─ 加载 SessionManager       │
        │      ├─ 注册工具 (createOpenClawCodingTools) │
        │      ├─ 调用 streamSimple (→ LLM) │
        │      ├─ 收到 tool_calls → execute │
        │      ├─ 结果写回 transcript        │
        │      └─ 循环直到模型结束           │
        └──────────────┬──────────────────┘
                       │
                       ▼
        Gateway 拿到最终回复 → 投递到通道 / WebChat / UI
```

**一句话边界**：Gateway 决定「何时、哪个会话、用哪个 Agent 跑」；Pi 决定「这一轮怎么和模型/工具交互」。

---

## 三、文件地图

### 3.1 Pi 运行时核心（`src/agents/pi-embedded-runner/`）

| 文件                                | 作用                                                                                              |
| ----------------------------------- | ------------------------------------------------------------------------------------------------- |
| **`run.ts`**                        | **入口**：`runEmbeddedPiAgent` — 入队、解析 model/auth/hooks、failover 循环                       |
| **`run/params.ts`**                 | `RunEmbeddedPiAgentParams` 类型定义 — 一次 run 需要的所有参数                                     |
| **`run/attempt.ts`**                | **单次尝试**：`runEmbeddedAttempt` — 构建 prompt、加载 session、注册工具、调 LLM、处理 tool_calls |
| **`run/payloads.ts`**               | 构建发给 LLM 的 payload（消息格式适配各 provider）                                                |
| **`run/types.ts`**                  | `EmbeddedRunAttemptParams` / `EmbeddedRunAttemptResult` 类型                                      |
| `lanes.ts`                          | session lane / global lane 解析（控制同一会话串行）                                               |
| `runs.ts`                           | 活跃 run 注册/查询/中止（`setActiveEmbeddedRun`、`abortEmbeddedPiRun`）                           |
| `session-manager-init.ts`           | `prepareSessionManagerForRun` — 初始化 / 修复 session 文件                                        |
| `session-manager-cache.ts`          | SessionManager 预热与缓存                                                                         |
| `compact.ts` / `compact.runtime.ts` | 会话压缩（compaction）：历史太长时自动总结                                                        |
| `system-prompt.ts`                  | 构建系统提示（注入 AGENTS/SOUL/USER/TOOLS/Skills 等）                                             |
| `model.ts`                          | `resolveModel` — 根据 provider + modelId 解析出具体模型配置                                       |
| `extra-params.ts`                   | 注入额外参数（parallel_tool_calls、cache control 等）                                             |
| `history.ts`                        | 历史轮次限制（`limitHistoryTurns`、DM 历史限制）                                                  |
| `google.ts`                         | Google/Gemini 特殊处理（turn 顺序、tool schema 清洗）                                             |
| `thinking.ts`                       | 思维/推理标签处理（`<think>` 块剥离等）                                                           |
| `tool-split.ts`                     | `splitSdkTools` — 把 Pi SDK 工具和 OpenClaw 工具分开                                              |
| `tool-name-allowlist.ts`            | 工具名白名单（按 policy 过滤可用工具）                                                            |
| `tool-result-truncation.ts`         | 过大的 tool result 截断                                                                           |
| `tool-result-context-guard.ts`      | 防止 tool result 把上下文撑爆                                                                     |
| `extensions.ts`                     | 构建 Pi 扩展工厂（provider 特殊适配）                                                             |
| `abort.ts`                          | 运行中止逻辑                                                                                      |

### 3.2 工具注册与定义（`src/agents/`）

| 文件                            | 作用                                                                                      |
| ------------------------------- | ----------------------------------------------------------------------------------------- |
| **`pi-tools.ts`**               | **`createOpenClawCodingTools`** — 组装完整工具列表（核心入口）                            |
| `openclaw-tools.js`             | `createOpenClawTools` — OpenClaw 专属工具（sessions、browser、canvas、nodes、message 等） |
| **`tool-catalog.ts`**           | 工具目录定义：每个工具的 id、label、description、所属分类、profile                        |
| `tool-policy.ts`                | 工具名规范化、allow/deny 策略                                                             |
| `pi-tools.read.ts`              | read 工具实现（含自适应大小、图片清洗）                                                   |
| `pi-tools.before-tool-call.ts`  | 工具调用前的参数调整                                                                      |
| `pi-tool-definition-adapter.ts` | 工具定义适配（Claude 兼容等）                                                             |

### 3.3 exec 工具链（`src/agents/`）

| 文件                              | 作用                                                   |
| --------------------------------- | ------------------------------------------------------ |
| **`bash-tools.exec.ts`**          | exec 工具主入口：根据 host 路由到 sandbox/gateway/node |
| `bash-tools.exec-host-gateway.ts` | host=gateway：在 Gateway 本机执行 + allowlist/审批     |
| `bash-tools.exec-host-node.ts`    | host=node：通过 Gateway → node.invoke → 远程 node 执行 |
| `bash-tools.exec-runtime.ts`      | 底层执行：`runExecProcess`、PATH 处理、schema 等       |
| `bash-process-registry.ts`        | 后台进程注册/查询/tail                                 |

### 3.4 事件与订阅（`src/agents/`）

| 文件                                           | 作用                                                                    |
| ---------------------------------------------- | ----------------------------------------------------------------------- |
| **`pi-embedded-subscribe.ts`**                 | **`subscribeEmbeddedPiSession`** — 订阅 Pi session 事件的总入口         |
| `pi-embedded-subscribe.handlers.ts`            | 事件分发（调用下面各 handler）                                          |
| `pi-embedded-subscribe.handlers.tools.ts`      | **tool_execution_start / end** 处理：日志、事件推送、审批 pending、媒体 |
| `pi-embedded-subscribe.handlers.messages.ts`   | assistant 消息处理：文本 delta、流式输出                                |
| `pi-embedded-subscribe.handlers.lifecycle.ts`  | 生命周期事件：run 开始/结束、错误                                       |
| `pi-embedded-subscribe.handlers.compaction.ts` | compaction 事件处理                                                     |
| `pi-embedded-subscribe.handlers.types.ts`      | 所有 handler 的类型定义                                                 |
| `pi-embedded-subscribe.tools.ts`               | 工具结果提取/清洗辅助函数                                               |

### 3.5 会话工具（`src/agents/tools/`）

| 文件                                                            | 作用                                          |
| --------------------------------------------------------------- | --------------------------------------------- |
| `sessions-resolution.ts`                                        | session key 解析、spawned 可见性检查          |
| `sessions-list.ts` / `sessions-history.ts` / `sessions-send.ts` | 三个会话工具的实现                            |
| `nodes-utils.ts`                                                | node 列表/解析辅助                            |
| `gateway.ts`                                                    | `callGatewayTool` — 工具通过 Gateway RPC 调用 |

---

## 四、关键路径（带你走几条最重要的链路）

### 路径 1：一次完整的 Agent 运行

```
Gateway chat.send
  → dispatchInboundMessage (src/auto-reply/dispatch.ts)
    → getReplyFromConfig (src/auto-reply/reply/get-reply.ts)
      → agentCommandFromIngress (src/commands/agent.ts)
        → runEmbeddedPiAgent (src/agents/pi-embedded-runner/run.ts)
          ├─ enqueueSession(sessionLane) → enqueueGlobal(globalLane)
          ├─ resolveModel(provider, modelId)
          ├─ resolveAuthProfileOrder → getApiKeyForModel
          └─ runEmbeddedAttempt (src/agents/pi-embedded-runner/run/attempt.ts)
              ├─ SessionManager.load(sessionFile)
              ├─ prepareSessionManagerForRun
              ├─ buildEmbeddedSystemPrompt (注入 AGENTS/SOUL/USER/Skills 等)
              ├─ createOpenClawCodingTools (注册所有工具)
              ├─ subscribeEmbeddedPiSession (订阅事件)
              ├─ createAgentSession + streamSimple (→ LLM API)
              │   ├─ 模型返回文本 delta → handler → onPartialReply
              │   ├─ 模型返回 tool_calls → Pi 调 execute
              │   │   └─ 结果写回 transcript → 再调 LLM
              │   └─ 模型结束 → 最终回复
              └─ 返回 EmbeddedPiRunResult
```

### 路径 2：模型调用 exec 工具执行代码

```
LLM 输出 tool_call: { name: "exec", args: { command: "ls -la", host: "gateway" } }
  → Pi 运行时调 exec 工具的 execute (src/agents/bash-tools.exec.ts)
    ├─ normalizeExecHost(params.host) → "gateway"
    ├─ host === "sandbox" → Docker 沙箱执行
    ├─ host === "gateway" → processGatewayAllowlist (bash-tools.exec-host-gateway.ts)
    │   ├─ evaluateShellAllowlist → 检查 allowlist
    │   ├─ requiresExecApproval → 是否需要审批
    │   └─ runExecProcess → 本机起子进程执行
    └─ host === "node" → executeNodeHostCommand (bash-tools.exec-host-node.ts)
        ├─ listNodes → resolveNodeIdFromList
        ├─ callGatewayTool("node.invoke", { command: "system.run", ... })
        └─ node 侧执行 → 返回 stdout/stderr/exitCode
```

### 路径 3：Agent A 通过 sessions_send 调 Agent B

```
Agent A 的 run 中，模型调用 sessions_send({ sessionKey: "agent:main:work", message: "..." })
  → sessions_send 工具 execute (src/agents/tools/sessions-send.ts)
    → callGateway("chat.send", { sessionKey: "agent:main:work", message, provenance: "inter_session" })
      → Gateway 在目标 session 上入队新消息
      → runEmbeddedPiAgent (目标 session 的 run)
        → 目标 Agent 处理消息 → 产生回复
      → agent.wait (等目标 run 结束)
    ← 返回 { status: "ok", reply: "..." } 给 Agent A
  → Agent A 的模型看到 sessions_send 的结果，继续对话
```

### 路径 4：会话压缩（compaction）

```
runEmbeddedAttempt 中，模型返回 context overflow 错误
  → isLikelyContextOverflowError → true
  → compactEmbeddedPiSession (src/agents/pi-embedded-runner/compact.ts)
    ├─ SessionManager 读取当前 transcript
    ├─ 用 LLM 生成历史摘要
    ├─ 替换 transcript 中的旧消息为摘要
    └─ 重试 runEmbeddedAttempt（用压缩后的 session）
  → subscribeEmbeddedPiSession 收到 compaction 事件
    → handlers.compaction.ts 处理
```

### 路径 5：工具事件如何推送到前端

```
Pi 运行时调用工具 → AgentEvent { type: "tool_execution_start", toolName, args }
  → subscribeEmbeddedPiSession 收到事件
    → handleToolExecutionStart (pi-embedded-subscribe.handlers.tools.ts)
      ├─ emitAgentEvent({ runId, stream: "tool", data: { phase: "start", name, args } })
      │   → Gateway 的 agent event bus
      │     → 已注册的 WebSocket 客户端收到 tool event
      │       → WebChat / macOS / Android UI 显示 "正在执行 exec..."
      └─ onToolResult → 如果 verbose 模式，推送工具输出
```

---

## 五、Pi 的边界总结

| Pi 负责                          | Pi 不负责（由 Gateway/OpenClaw 处理） |
| -------------------------------- | ------------------------------------- |
| 调用 LLM（stream API）           | 选择哪个 agentId / sessionKey         |
| 管理工具调用循环                 | 通道收发（Telegram/Slack/...）        |
| SessionManager / transcript 读写 | DM 安全策略 / pairing / allowlist     |
| 系统提示构建                     | 配置文件解析 / workspace 路径         |
| 流式事件订阅                     | 消息投递到具体通道                    |
| Auth profile 轮换 / failover     | Skills 安装 / ClawHub                 |
| Compaction（上下文压缩）         | Cron / Webhook / Hook 触发            |
| 工具 execute 调用                | exec 的物理安全审批（allowlist/ask）  |

---

## 六、推荐阅读顺序

1. **先看参数**：`run/params.ts` — 理解一次 run 需要什么输入。
2. **看入口**：`run.ts` 的 `runEmbeddedPiAgent` — 入队、model 解析、auth、failover 循环。
3. **看单次尝试**：`run/attempt.ts` 的 `runEmbeddedAttempt` — 这是最核心的"一轮对话"逻辑。
4. **看工具注册**：`pi-tools.ts` 的 `createOpenClawCodingTools` — 理解工具从哪来。
5. **看一个具体工具**：`bash-tools.exec.ts` — 理解 exec 如何路由到 sandbox/gateway/node。
6. **看事件订阅**：`pi-embedded-subscribe.ts` + `handlers.tools.ts` — 理解工具事件如何推到前端。
7. **看 session 管理**：`session-manager-init.ts` + `compact.ts` — 理解 transcript 初始化与压缩。

---

_本文档基于 `src/agents/pi-embedded-runner/`、`src/agents/pi-_.ts`、`src/agents/bash-tools._.ts` 等源码整理。_
