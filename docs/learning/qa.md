# OpenClaw 学习 Q&A 文档

本文档是**统一的问答积累**：当你明确说「写入文档」时，当前问题和答案会以统一格式追加到这里。

---

## 格式说明

每条记录格式：

- **Q:** 你的问题（可简短概括）
- **A:** 答案摘要或要点
- _（可选）日期或来源_

---

## 问答记录

---

### Q: run.ts 详细分析；它和 runs.ts 的区别是什么？run.ts 是 agent 循环吗？

**A:**

**1. run.ts 是做什么的**

- **入口**：`runEmbeddedPiAgent(params)`（约 255 行起），是「跑一轮 Pi Agent」的入口，由 Gateway 在 `agentCommandFromIngress` 里调用。
- **入队**：先用 `enqueueSession(sessionLane)`、再 `enqueueGlobal(globalLane)` 把任务放进 command queue，保证同一 session 串行、不同 session 可并行。
- **准备**：解析 workspace、model、auth profiles、context window、hooks（before_model_resolve / before_agent_start）等；处理 Copilot token 刷新、auth profile 轮换。
- **核心循环**：`while (true)`（约 807 行）里每次调用 **`runEmbeddedAttempt(...)`**（在 `run/attempt.ts`）做「一次尝试」：
  - 加载 SessionManager、构建 system prompt、注册工具、调 LLM（streamSimple）、处理 tool_calls、把结果写回 transcript。
  - 成功则组装 `payloads` / `meta` 并 **return**，结束这一轮。
  - 失败则视情况：换 auth profile、降级 thinking level、做 compaction、做 tool result 截断、或抛出 `FailoverError` 让外层做 model fallback；然后 **continue** 再试一次。
- **上限**：`runLoopIterations` 达到 `MAX_RUN_LOOP_ITERATIONS` 就停止重试并返回错误。
- 因此 run.ts 是 **「一次用户请求」的编排层**：入队、准备、多次「尝试」直到成功或放弃，而不是单次 LLM 调用的实现。

**2. run.ts 和 runs.ts 的区别**

- **run.ts**：实现「怎么跑」——执行一次 `runEmbeddedPiAgent` 的完整流程（入队、重试、调 attempt、返回结果）。是**逻辑**。
- **runs.ts**：维护「当前有哪些 run 在跑」——用 `ACTIVE_EMBEDDED_RUNS`（Map: sessionId → handle）注册/注销；提供 `setActiveEmbeddedRun` / `clearActiveEmbeddedRun`、`queueEmbeddedPiMessage`、`abortEmbeddedPiRun`、`isEmbeddedPiRunActive`、`waitForEmbeddedPiRunEnd` 等。是**状态/注册表 + 控制接口**。
- 关系：run.ts 里某次执行在开始时会通过 attempt 侧把 handle 注册到 runs.ts；结束时 clear。外部要 abort/等待/排队消息时，只依赖 runs.ts，不直接依赖 run.ts 的内部循环。

**3. run.ts 是 agent 循环吗？**

- **不是**「模型 ↔ 工具」那种 agent 循环。**Agent 循环**（用户消息 → 模型 → 若有 tool_calls 则执行工具 → 结果再喂给模型 → 直到模型结束）在 **`run/attempt.ts`** 的 `runEmbeddedAttempt` 里，由 Pi 的 stream + 工具 execute 完成。
- run.ts 是 **外层重试/容错循环**：在一次用户请求下，可能多次调用 `runEmbeddedAttempt`（换 profile、降级 thinking、compaction 后重试等），直到某次 attempt 成功或达到重试上限。所以更准确的说法是：run.ts 是 **run 级别的重试与编排**，agent 循环在 attempt 内部。

---

### Q: run.ts 和 run/attempt.ts 区别在哪？分别都是什么作用？（图文并茂）

**A:**

**1. 一句话区别**

| 文件               | 作用                                                                                                                                 |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------ |
| **run.ts**         | **编排层**：一次用户请求的入口；入队、准备环境、**反复调用 attempt 直到成功或放弃**（重试 / 换 auth / compaction / 降级 thinking）。 |
| **run/attempt.ts** | **单次尝试层**：**只做一次**「加载会话 → 拼 prompt → 调 LLM → 执行 tool_calls → 再调 LLM …」直到本轮回复结束或失败。                 |

**2. 调用关系与职责（图）**

```
                    Gateway (chat.send / agent)
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  run.ts  ———  编排 / 重试外壳                                              │
│  runEmbeddedPiAgent()                                                     │
│    ├─ 入队 sessionLane + globalLane                                        │
│    ├─ 解析 workspace、model、auth、context、hooks                           │
│    └─ while (true) {  ◄———— 重试循环（不是 agent 循环）                     │
│           ├─ runEmbeddedAttempt()  ──────────────┐                        │
│           │    成功 → return 结果                 │                        │
│           │    失败 → 换 profile / 降级 thinking  │                        │
│           │         / compaction / 截断 tool 结果 │                        │
│           │         → continue 再试一次           │                        │
│           └─ 超过 MAX_RUN_LOOP_ITERATIONS → return 错误                    │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ 每次循环调用一次
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  run/attempt.ts  ———  单次尝试 / 真正的 agent 循环                         │
│  runEmbeddedAttempt()                                                     │
│    ├─ 加载 SessionManager（transcript）                                    │
│    ├─ 构建 system prompt（AGENTS/SOUL/USER/Skills/TOOLS）                  │
│    ├─ createOpenClawCodingTools() 注册工具                                 │
│    ├─ subscribeEmbeddedPiSession() 订阅事件                                │
│    └─ createAgentSession + streamSimple()  ────►  LLM API                  │
│           │                                                                 │
│           │  ◄———— 这里才是「agent 循环」：                                 │
│           │    模型返回 → 若有 tool_calls → 执行工具 → 结果写回 session      │
│           │    → 再请求模型 → … 直到模型结束                                │
│           │                                                                 │
│           ▼                                                                 │
│    返回 attempt 结果（assistantTexts、lastAssistant、usage…）              │
└─────────────────────────────────────────────────────────────────────────┘
```

**3. attempt 内部「单次尝试」里发生了什么（图）**

```
runEmbeddedAttempt 一次执行
        │
        ├─ SessionManager.load(sessionFile)
        ├─ prepareSessionManagerForRun
        ├─ buildEmbeddedSystemPrompt  ──►  system prompt 字符串
        ├─ createOpenClawCodingTools  ──►  [read, write, exec, browser, sessions_*, …]
        ├─ subscribeEmbeddedPiSession     ──►  流式事件推给 Gateway/UI
        │
        └─ createAgentSession + streamSimple(prompt, tools, …)
                    │
                    ▼
            ┌───────────────────┐
            │   LLM 返回         │
            │  (streaming)       │
            └─────────┬─────────┘
                      │
          ┌───────────┴───────────┐
          │                       │
          ▼                       ▼
    纯文本 delta            tool_calls
          │                       │
          │                       ▼
          │               执行工具 execute()
          │               结果写回 transcript
          │                       │
          │                       ▼
          │               再次请求 LLM（带新消息）
          │                       │
          └───────────┬───────────┘
                      │
                      ▼
              直到模型结束（无更多 tool_calls）
                      │
                      ▼
            返回 EmbeddedRunAttemptResult
```

**4. 总结表**

| 维度         | run.ts                                         | run/attempt.ts                                      |
| ------------ | ---------------------------------------------- | --------------------------------------------------- |
| **入口**     | `runEmbeddedPiAgent`                           | `runEmbeddedAttempt`                                |
| **调用关系** | 被 Gateway 调；内部多次调 attempt              | 只被 run.ts 调                                      |
| **循环**     | 外层 **重试循环**（换 profile、compaction 等） | 内层 **agent 循环**（LLM ↔ 工具）                   |
| **职责**     | 入队、环境准备、重试策略、返回最终 payload     | 单次「会话加载 → prompt → LLM → 工具 → LLM → 结果」 |
| **失败时**   | 决定是否重试、换账号、降级、或抛 FailoverError | 直接返回错误或结果，不负责重试                      |
