# 后端执行、工具调用与智能体隔离/通信（学习用）

本文档面向**关心“每个问题如何在后端执行、大模型如何调用工具、问题之间如何隔离、智能体之间如何通信”**的读者，不展开 Gateway 连接各渠道的细节。

---

## 一、一条请求在后端如何执行（从入口到 Agent 跑完）

### 1.1 入口：chat.send 或 agent 命令

- **WebChat / Control UI / 各渠道** 发消息时，最终都会走到 Gateway 的 **`chat.send`** 或 **`agent`** RPC。
- 实现位置：`src/gateway/server-methods/chat.ts`（`chat.send`）、`src/gateway/server-methods/agent.ts`（`agent`）。

`chat.send` 会：

1. 校验参数、清理消息、解析 `sessionKey`；
2. 用 **`loadSessionEntry(rawSessionKey)`** 解析出 `sessionKey` 和会话元数据；
3. 构造 **`MsgContext`**（包含 `SessionKey`、消息内容、来源通道等）；
4. 调用 **`dispatchInboundMessage`**，把“回复”交给 **`dispatchReplyFromConfig`**（`src/auto-reply/reply/dispatch-from-config.ts`），最终走到 **`getReplyFromConfig`**（`src/auto-reply/reply/get-reply.ts`）。

### 1.2 会话与 Agent 作用域

- **`sessionKey`** 决定“这是哪个会话”：格式一般为 `agent:<agentId>:<rest>`，例如 `main`（会规范成当前 agent 的 main）、`agent:main:telegram:group:xxx` 等。
- **`agentId`** 由 **`resolveSessionAgentId({ sessionKey, config })`** 从配置和 sessionKey 解析出来（见 `src/agents/agent-scope.js`、`src/routing/session-key.ts`）。
- 每个会话对应一个 **session store 条目** 和 ** transcript 文件**：`~/.openclaw/agents/<agentId>/sessions/` 下 `sessions.json` + `<sessionId>.jsonl`。

### 1.3 入队：按会话串行（lane）

- 真正跑模型的是 **`runEmbeddedPiAgent`**（`src/agents/pi-embedded-runner/run.ts`）。
- 每次运行会先算两条“车道”：
  - **sessionLane** = `resolveSessionLane(sessionKey)` → 形如 `session:${sessionKey}`；
  - **globalLane** = `resolveGlobalLane(lane)` → 默认 `main`。
- 任务通过 **`enqueueCommandInLane`**（`src/process/command-queue.ts`）入队：
  - 先 **`enqueueSession(...)`**：同一 session 的任务在同一 session lane 里**串行**；
  - 再在 global lane 里跑实际逻辑，避免不同 lane 之间互相抢资源。
- 因此：**同一会话内，请求是一个接一个执行的；不同会话之间可以并行（不同 session lane）**。

### 1.4 一次 Agent 运行里做了什么

- **`agentCommandFromIngress`**（`src/commands/agent.ts`）根据入口参数准备 prompt、会话目录、超时等，然后调用 **`runEmbeddedPiAgent`**。
- **`runEmbeddedPiAgent`** 内会：
  - 解析 workspace、model、auth、hooks 等；
  - 用 **`runEmbeddedAttempt`**（`src/agents/pi-embedded-runner/run/attempt.ts`）做**单次尝试**：
    - 加载/准备 **SessionManager**（Pi 的会话与 transcript）；
    - 构建 **system prompt**（含 AGENTS.md、SOUL.md、USER.md、skills 等）；
    - 调用 **`streamSimple`**（或等价流式 API）向大模型发请求，并传入 **OpenClaw 工具集** 和 **subscribe** 回调。
- 大模型流式返回：**文本 delta** 与 **tool_calls**。Pi 运行时每收到 tool_call 就调用对应工具的 **execute**，把结果写回 session，再继续流式请求，直到模型不再调用工具并结束当轮回复。

---

## 二、大模型如何调用工具能力（工具注册与执行）

### 2.1 工具从哪来

- OpenClaw 的工具列表由 **`createOpenClawCodingTools()`**（及扩展）生成，在 **`runEmbeddedAttempt`** 里传给 Pi 的 session/stream（见 `src/agents/pi-tools.js`、`src/agents/pi-embedded-runner/run/attempt.ts`）。
- 每个工具是符合 Pi/Agent 约定的 **AgentTool**：有 `name`、`description`、参数 schema、**`execute(args, context)`**。
- 例如：`exec` / `bash`、`read`、`write`、`edit`、`browser.*`、`sessions_list`、`sessions_send`、`sessions_history` 等，定义分散在 `src/agents/`（如 `bash-tools.exec.ts`、`pi-tools.read.ts`、`tools/sessions-*.ts` 等）。

### 2.2 模型发出 tool_call 之后

- Pi 运行时（`@mariozechner/pi-agent-core` / `pi-coding-agent`）在流式循环里收到 **tool_calls** 后，会调用每个工具的 **execute**，并把返回的 **tool result** 写回当前 session 的 transcript，再继续请求模型。
- OpenClaw 通过 **`subscribeEmbeddedPiSession`** 订阅同一 session 的 **AgentEvent**（`src/agents/pi-embedded-subscribe.js`），在 **`tool_execution_start`** / **`tool_execution_end`** 时做额外处理：
  - **`handleToolExecutionStart`** / **`handleToolExecutionEnd`**（`src/agents/pi-embedded-subscribe.handlers.tools.ts`）：打日志、发 Gateway 的 tool 事件给已注册的客户端、处理 exec 审批 pending、媒体 URL、消息类工具发送等。
- 因此：**“运行代码”等能力 = 模型在回复中发出 tool_calls → Pi 调用 OpenClaw 工具的 execute（如 exec、read、write）→ 结果回写 transcript → 模型看到结果并继续**。执行都发生在 **当前 run 所在进程**（Gateway 本机或通过 node host 转发到 node 机）。

### 2.3 和“运行代码”直接相关的工具

- **exec / bash**：在配置的 **host**（`sandbox` / `gateway` / `node`）上跑 shell 命令；实现见 `src/agents/bash-tools.exec.ts`，gateway 本机执行在 `bash-tools.exec-host-gateway.ts`，node 执行在 `bash-tools.exec-host-node.ts`。
- **read / write / edit**：工作区文件读写，受 workspace 与沙箱约束；实现见 `src/agents/pi-tools.read.ts` 等。

---

## 三、不同问题/请求之间如何隔离

### 3.1 会话维度：sessionKey

- **每个“会话”由 sessionKey 唯一标识**（如 `main`、`agent:main:discord:group:123`）。
- 会话状态与 transcript 按 **`~/.openclaw/agents/<agentId>/sessions/`** 存储；**不同 sessionKey = 不同会话 = 不同 transcript、不同 session store 条目**。
- **DM 隔离**：通过 **`session.dmScope`**（如 `per-channel-peer`）可以把不同用户/不同通道的 DM 拆成不同 sessionKey，避免串会话（见 `docs/concepts/session.md`）。

### 3.2 执行维度：session lane 串行

- 同一 **sessionKey** 的多次请求会进入 **同一 session lane**（`session:${sessionKey}`），**同一 lane 内串行执行**，因此同一会话内不会出现两条请求交叉执行、互相写同一 transcript 的情况。
- 不同 sessionKey 的请求进入不同 lane，可以并行。

### 3.3 runId 与单次运行

- 每次 `runEmbeddedPiAgent` 会有一个 **runId**（通常也是该轮 chat 的 idempotencyKey / clientRunId）。
- **runId** 会挂在 **AgentRunContext**（`src/infra/agent-events.js`），工具执行、审批、事件推送都会带上 runId，便于区分“当前是哪一轮对话在跑”。

### 3.4 沙箱（可选）

- 若开启 **sandbox**（`agents.defaults.sandbox`），exec 可配置为在 **Docker 沙箱** 中执行，文件类工具也可限制在工作区/沙箱内，进一步隔离（见 `docs/gateway/sandboxing`、`docs/tools/exec.md`）。

---

## 四、智能体之间如何通信（多 Agent / 多会话）

### 4.1 会话即“智能体”的可见范围

- 这里“智能体”可以理解为：**一个 agentId 下的一个会话**（一个 sessionKey 对应一个 transcript 和一次次的 run）。
- 不同 sessionKey = 不同会话，彼此默认**不共享上下文**；要协作就通过 **会话工具** 显式通信。

### 4.2 会话工具：sessions_list / sessions_history / sessions_send

- **`sessions_list`**：列出当前 agent 下的会话（main、group、cron、hook、node 等），返回 sessionKey、sessionId、updatedAt、token 等。
- **`sessions_history`**：按 **sessionKey（或 sessionId）** 拉取某会话的 transcript 消息（可选是否包含 tool 结果）。
- **`sessions_send`**：向**另一个会话**发一条消息，触发对方会话的一次 Agent 运行；可选等待对方跑完再返回（timeoutSeconds > 0）或仅“发出即返回”（fire-and-forget）。

文档：`docs/concepts/session-tool.md`。

### 4.3 sessions_send 的流程（Agent A 调 Agent B）

1. 当前 run（Agent A）在工具里调用 **sessions_send**，传入 **target sessionKey** 和 **message**。
2. Gateway 侧处理 **sessions_send**（通过 callGateway 或等价 RPC）：在目标 session 上 **入队一条新消息**（与 chat.send 类似），并标记为 **inter_session**（`message.provenance.kind = "inter_session"`），便于 transcript 里区分是“另一个 agent 发来的”。
3. 若调用方设置了 **timeoutSeconds > 0**，Gateway 会 **agent.wait** 等目标 run 结束，再把目标会话的回复返回给调用方；否则只返回“已接受”。
4. 可选 **reply-back**：目标会话跑完后，可以再向发起方会话回一条消息，形成多轮 ping-pong（见 session-tool 文档中的 REPLY_SKIP / 回复回环说明）。

### 4.4 子 Agent / sessions_spawn

- **sessions_spawn**：在一个会话里“ spawn ”出一个新的子会话（subagent），常用于把大任务拆成多个专注会话。子会话的 sessionKey 通常带 `subagent` 等标识；可见性、spawnedBy 等见 **`sessions_list` 的 `spawnedBy` 过滤** 和 `src/agents/tools/sessions-resolution.ts`（如 `listSpawnedSessionKeys`）。

---

## 五、代码索引（按主题）

| 主题                             | 主要文件/目录                                                                                                                                                                          |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 请求入口（chat.send → dispatch） | `src/gateway/server-methods/chat.ts`，`src/auto-reply/dispatch.ts`，`src/auto-reply/reply/dispatch-from-config.ts`，`src/auto-reply/reply/get-reply.ts`                                |
| 会话 key / agent 解析            | `src/routing/session-key.ts`，`src/agents/agent-scope.js`，`src/config/sessions.js`                                                                                                    |
| 入队与 lane（同一会话串行）      | `src/process/command-queue.ts`，`src/agents/pi-embedded-runner/lanes.ts`，`src/agents/pi-embedded-runner/run.ts`（enqueueSession / enqueueGlobal）                                     |
| 单次 Agent 运行（Pi + 工具）     | `src/commands/agent.ts`（agentCommandFromIngress），`src/agents/pi-embedded-runner/run.ts`（runEmbeddedPiAgent），`src/agents/pi-embedded-runner/run/attempt.ts`（runEmbeddedAttempt） |
| 工具定义与执行                   | `src/agents/pi-tools.js`，`src/agents/bash-tools.exec.ts`，`src/agents/pi-tools.read.ts`，`src/agents/tools/`（sessions-\*.ts 等）                                                     |
| 工具事件与订阅                   | `src/agents/pi-embedded-subscribe.js`，`src/agents/pi-embedded-subscribe.handlers.tools.ts`                                                                                            |
| 会话工具（list/history/send）    | `src/agents/tools/sessions-*.ts`，`docs/concepts/session-tool.md`                                                                                                                      |
| 会话存储与隔离                   | `docs/concepts/session.md`，`src/config/sessions.js`                                                                                                                                   |

---

_本文档基于代码与 `docs/concepts/session.md`、`docs/concepts/session-tool.md`、`docs/tools/exec.md` 整理，便于聚焦后端执行与智能体行为。_
