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

---

### Q: `command-queue.ts` 里的 `enqueueCommandInLane` 函数什么作用？它调用了哪些函数？

**A:**

`enqueueCommandInLane(lane, task)` 是一个**进程内的异步命令队列**入口，用来把“要执行的异步任务”按 `lane` 分组并串行/限并发执行。它的核心作用是：把任务放入对应 `lane` 的队列，然后由队列调度器 `drainLane` 逐个取出执行，最后用队列里的 `resolve/reject` 把结果返回给调用者。

调用链（按执行顺序）：

1. **`enqueueCommandInLane`**
   - 如果网关正在重启排空（`gatewayDraining === true`），会直接 `reject(new GatewayDrainingError())`，避免“任务进队但会被 shutdown 丢弃”。
   - 对 `lane` 做规范化：`cleaned = lane.trim() || CommandLane.Main`，空字符串则归到默认 `main` lane。
   - 通过 `getLaneState(cleaned)` 取/建该 lane 的状态（队列、并发数、运行中任务集合等）。
   - 把 `task` 以工厂形式入队：`task: () => task()`（保证真正执行发生在 dequeue 之后）。
   - 调用 `logLaneEnqueue(cleaned, state.queue.length + state.activeTaskIds.size)` 输出诊断/事件。
   - 调用 `drainLane(cleaned)` 唤醒队列调度执行。

2. **`getLaneState(lane)`**
   - 从全局 `lanes: Map<string, LaneState>` 里拿该 lane 的 `LaneState`。
   - 若不存在则创建：`queue=[]`、`activeTaskIds=Set()`、`maxConcurrent=1`（默认串行）、`draining=false`、`generation=0`。

3. **`logLaneEnqueue`（在 `src/logging/diagnostic.ts`）**
   - `diag.debug(...)` 并发出诊断事件 `queue.lane.enqueue`，同时 `markActivity()`。

4. **`drainLane(lane)`**
   - 这是“真正开始执行队列”的地方。
   - 防重入：如果 `state.draining === true` 直接返回；否则置为 `true`。
   - 内部定义 `pump()` 并立刻调用：
     - `while (activeTaskIds.size < maxConcurrent && queue.length > 0)`：持续从队列 `shift` 出条目。
     - 计算等待时长 `waitedMs = Date.now() - entry.enqueuedAt`：
       - 若超出 `entry.warnAfterMs`，调用 `entry.onWait?.(waitedMs, queueAhead)`（回调失败会记录 error），并打 warn。
     - 打诊断：`logLaneDequeue(lane, waitedMs, state.queue.length)`。
     - 分配执行 id：`taskId = nextTaskId++`，并抓取 `taskGeneration = state.generation`。
     - 把 `taskId` 加入 `activeTaskIds`。
     - 启动一个异步块执行 `entry.task()`：
       - 成功：`completeTask(state, taskId, taskGeneration)` 若仍是当前 generation 则 `pump()` 继续；然后 `entry.resolve(result)`。
       - 失败：同样通过 `completeTask` 判断是否仍有效；非 probe lane 会记录 error；然后 `entry.reject(err)`。
   - `finally` 里把 `state.draining = false`。

5. **`completeTask(state, taskId, taskGeneration)`**
   - 用来处理“重启/重置后旧任务完成回调仍然回来”的情况。
   - 若 `taskGeneration !== state.generation`，直接返回 `false`（表示这次完成不应再触发后续 pump，也不应当删掉当前 active 集合）。
   - 否则从 `activeTaskIds` 删除该任务 id，并返回 `true`。

相关的配套函数（理解整体语义会用到）：

- **`setCommandLaneConcurrency(lane, maxConcurrent)`**：修改某个 lane 的 `maxConcurrent` 并重新 `drainLane`。
- **`enqueueCommand(task)`**：默认入 `CommandLane.Main`。
- **`clearCommandLane(lane)`**：清空 lane 的 `queue`，并对所有待执行条目 `reject(new CommandLaneClearedError(lane))`。
- **`resetAllLanes()`**：在某些进程内重启场景里，递增 `generation` 并清空 `activeTaskIds`，同时对仍有队列的 lane 重新 drain，避免 stale active id 永久卡死。

---

### Q: OpenClaw 如何支持“自动查询 skill / 自动写脚本 / 自动安装依赖”？大模型如何知道自己“手里有多少牌”？

**A:**

你看到的这些“自动化”本质上是：OpenClaw 在每次 agent run 前，先把“可用动作清单（工具 + skills）”组装进 system prompt，然后再把对应的后端能力（exec、写文件、skills.install 等）提供给模型调用；模型能调用什么，取决于 system prompt 里注入了哪些内容以及是否被截断/限用。

1. **“牌的数量/清单”来自 system prompt 注入（而不是模型自行猜）**
   - 在 `src/auto-reply/reply/commands-system-prompt.ts` 的 `resolveCommandsSystemPromptBundle()` 中：
     - 通过 `buildWorkspaceSkillSnapshot(...)` 得到 `skillsSnapshot`，并取 `skillsSnapshot.prompt` 作为 `skillsPrompt`。
     - 通过 `createOpenClawCodingTools(...)` 得到 `tools`，并把 `toolNames` / `toolSummaries` 一起传给 `buildAgentSystemPrompt(...)`。
     - 最终 `skillsPrompt` 会作为参数传入 `buildAgentSystemPrompt`，也就是会出现在模型的 system prompt 文本里（模型“看见”的就是这份文本）。

   - 在 `src/auto-reply/reply/commands-context-report.ts` 里，`/context` 报告会直接输出：
     - `Skills list (system prompt text): ... (N skills)`
     - `Tools: ...`  
       说明模型看到的“skills/tools 数量”是由后端生成并统计出来的。

2. **为什么有时“牌看起来不全”：skills prompt 会按预算截断**
   - 在 `src/agents/skills/workspace.ts` 的 `applySkillsPromptLimits()` 中，会对要注入 prompt 的 skills 做：
     - `maxSkillsInPrompt` 的数量截断
     - `maxSkillsPromptChars` 的字符预算截断
   - 所以“手里有多少牌”不是固定值，而是受 prompt 字符预算影响（`/context detail` 能看到截断与否）。

3. **“自动查询 skill”通常来自一个可用的 skill（例如内置 `clawhub`）**
   - 仓库里有 `skills/clawhub/SKILL.md`，它的描述明确写了：
     - 使用 ClawHub CLI 在 `clawhub.com` 上搜索/安装/更新/发布 skills
   - 该 skill 同时声明了需要的二进制 `clawhub`，并给出安装方式（见其 `metadata.openclaw.requires.bins` 与 `metadata.openclaw.install`）。
   - 当这个 skill 被扫描并纳入 `skillsSnapshot.prompt` 后，模型就“知道”能用它来完成 skill 的检索与安装。

4. **“自动安装依赖”由 `skills.install` + `installSkill()` 实现**
   - 在 `src/gateway/server-methods/skills.ts` 里存在网关方法：
     - `skills.install`：校验参数后调用 `installSkill(...)`，把安装结果返回给调用方。
   - `src/agents/skills-install.ts` 中：
     - 根据 skill frontmatter 的 `install` 元数据生成安装命令（`brew/node/go/uv/download` 等）
     - 对 `node` 类型，会根据 `skills.install.nodeManager` 选择 `pnpm/yarn/bun/npm` 具体命令（例如 `pnpm add -g --ignore-scripts ...`）
   - 另外，`installSkill()` 里还有“安全扫描/告警”等处理（例如安装前做目录扫描并记录 suspicious/critical pattern 的 warning）。

5. **“自动写脚本 / 安装包（npm install 等）”来自工具链：写文件 + exec**
   - 在 `src/agents/pi-tools.ts` 的 `createOpenClawCodingTools()` 中，工具会包含：
     - `write` / `edit`（以及 `apply_patch`，取决于沙箱/权限）
     - `exec`（通过 `createExecTool` 创建，传入 `ask/security/safeBins/...` 等策略）
   - 所以模型能“写脚本”是因为它能把脚本内容写进工作区文件；能“安装需要的包”是因为它能用 `exec` 工具执行诸如 `npm install` / `pnpm add ...` / `pip install ...`（是否需要审批、是否在沙箱内运行，取决于 `exec` 的策略与 sandbox 设置）。

总结一下：OpenClaw 的“自动化”不是模型凭空拥有能力，而是每次 run 前由后端把“skills/tools 清单”注入到 system prompt，并把可执行能力（exec、写文件、skills.install 等）挂到模型可调用的工具上；模型看到的 skills 数量与工具清单，就是你说的“手里有多少牌”。
