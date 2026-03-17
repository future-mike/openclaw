# OpenClaw 项目介绍（学习用）

本文档面向**刚开始接触 OpenClaw 的开发者**，帮助快速建立整体认知和阅读路径。

---

## 一、一句话是什么

**OpenClaw** 是在你自己设备上运行的**个人 AI 助手**：通过你已有的聊天渠道（WhatsApp、Telegram、Slack、Discord、WebChat 等）和你对话，支持语音、画布、多 Agent 路由；**Gateway 是控制面，产品形态是助手**。

---

## 二、核心目标与特点

| 维度         | 说明                                                                           |
| ------------ | ------------------------------------------------------------------------------ |
| **定位**     | 本地优先、快速、常驻的单用户个人助手                                           |
| **统一入口** | 多通道收件箱 + 统一工具（浏览器、Canvas、节点、Cron、会话等）                  |
| **技术栈**   | Node ≥22、TypeScript、pnpm；CLI 入口 `openclaw.mjs`；构建产物 `dist/`          |
| **扩展**     | 可选 macOS / iOS / Android 原生应用；插件在 `extensions/`，Skills 在 `skills/` |

---

## 三、整体架构（简化）

```
各渠道消息 (WhatsApp / Telegram / Slack / Discord / … / WebChat)
                    │
                    ▼
        ┌─────────────────────────┐
        │        Gateway          │  控制面
        │   ws://127.0.0.1:18789   │
        └────────────┬────────────┘
                     │
        ├─ Pi Agent (RPC)          ← 对话与工具调用
        ├─ CLI (openclaw …)       ← 命令行
        ├─ WebChat / Control UI   ← Web 界面
        └─ macOS / iOS / Android   ← 可选客户端
```

- **Gateway**：单一 WebSocket 控制面，负责会话、配置、Cron、Webhook、Control UI、Canvas 宿主等。
- **Pi Agent**：RPC 模式运行，支持工具调用与流式输出。
- **通道**：每个渠道一个实现（如 Telegram、Discord），消息进/出都经 Gateway 路由。

---

## 四、主要功能模块

1. **Gateway WebSocket 控制面** — 会话、在线状态、配置、Cron、Webhook、Control UI。
2. **多通道收件箱** — 各 IM 通道的接入与路由（含群组、@ 触发等）。
3. **Pi Agent RPC 运行时** — 对话、工具调用、流式输出、多 Agent 路由。
4. **Voice Wake / Talk Mode** — 语音唤醒与连续语音（macOS/iOS/Android）。
5. **Live Canvas (A2UI)** — 由 Agent 驱动的可视化画布。
6. **工具** — 浏览器、Canvas、节点（相机/录屏/通知等）、Cron、会话间通信等。
7. **入驻向导 (onboard)** — CLI 向导配置 Gateway、工作区、通道、Skills。
8. **Skills 平台（ClawHub）** — 内置/托管/工作区 Skills 的安装与管理。

---

## 五、目录结构（学习时怎么找代码）

| 目录/文件       | 作用                                                | 学习时可关注     |
| --------------- | --------------------------------------------------- | ---------------- |
| `src/`          | 核心 TypeScript 源码                                | 主阅读区         |
| `src/gateway/`  | Gateway WS 服务、会话、配置、Cron、Webhook          | 理解控制面       |
| `src/agents/`   | Pi Agent 运行时、RPC、工具调用与流式输出            | 理解对话与工具   |
| `src/channels/` | 各通道实现（Telegram、Discord、Slack 等）           | 理解多通道       |
| `src/cli/`      | CLI 命令（gateway、agent、send、wizard、doctor 等） | 理解命令行入口   |
| `src/config/`   | 配置解析与校验                                      | 理解配置模型     |
| `src/web/`      | WebChat、Control UI、登录/会话/媒体                 | 理解 Web 端      |
| `src/wizard/`   | 入驻向导 (onboard)                                  | 理解首次配置流程 |
| `extensions/`   | 插件/扩展（如更多频道）                             | 理解扩展方式     |
| `skills/`       | 内置与示例 Skills                                   | 理解 Skills 形态 |
| `ui/`           | Control UI 前端工程                                 | 前端技术栈       |
| `apps/`         | macOS / iOS / Android 原生应用                      | 可选学习         |
| `docs/`         | 官方文档（Mintlify，docs.openclaw.ai）              | 概念与操作       |
| `test/`         | 单元/集成/E2E 测试                                  | 理解行为与回归   |
| `openclaw.mjs`  | CLI 入口；`pnpm openclaw` 即由此进入                | 入口必看         |

---

## 六、本地运行（最小闭环）

1. **环境**：Node ≥22，推荐 pnpm（或 npm/bun）。
2. **安装与构建**：`pnpm install` → 首次 `pnpm ui:build` → `pnpm build`。
3. **入驻**：`pnpm openclaw onboard --install-daemon`（向导配置 Gateway、工作区、通道、Skills，并可安装守护进程）。
4. **开发**：`pnpm gateway:watch`（TS 变更自动重载）；或直接 `pnpm openclaw gateway --port 18789 --verbose`。
5. **访问**：Gateway 默认 `ws://127.0.0.1:18789`；Control UI / WebChat 由 Gateway 提供。

建议：先按上述跑通一次，再带着「一条消息从通道进来到 Agent 回复出去」的路径去读代码。

---

## 七、推荐学习路径

1. **先跑起来**：按「六」完成安装、入驻、`gateway:watch`，用 WebChat 或 CLI 发几条消息。
2. **理解数据流**：从「某条通道收到消息」→ Gateway 路由 → Agent 处理 → 回复回通道，在 `src/channels/`、`src/gateway/`、`src/agents/` 里顺一条链路。
3. **后端执行与智能体**（不关注渠道时可优先看）：每条请求如何入队、大模型如何调工具跑代码、不同问题/会话如何隔离、智能体之间如何通信，见 **[后端执行与智能体](backend-execution-and-agents.md)**。
4. **Pi Agent 深入**：Pi 运行时的文件地图、5 条关键路径（完整 run、exec 执行、Agent 间通信、compaction、事件推送），见 **[Pi Agent 深入](pi-agent-deep-dive.md)**。
5. **看配置与 CLI**：`src/config/`、`src/cli/`，配合 `openclaw onboard` 和 `openclaw doctor` 理解配置与健康检查。
6. **看一个通道**：任选一个你熟悉的渠道（如 Telegram 或 Discord），在 `src/channels/` 或 `extensions/` 里看完「连接 → 收消息 → 发回」。
7. **看 Agent 与工具**：`src/agents/` 里 Pi RPC、工具注册与调用、流式输出。
8. **按需深入**：Canvas、Voice、Cron、Skills、插件 SDK 等，结合 [docs.openclaw.ai](https://docs.openclaw.ai) 对应章节。

---

## 八、常用命令速查

| 命令                                           | 说明                       |
| ---------------------------------------------- | -------------------------- |
| `pnpm openclaw onboard --install-daemon`       | 入驻向导并安装守护进程     |
| `pnpm gateway:watch`                           | 开发模式，TS 变更自动重载  |
| `pnpm openclaw gateway --port 18789 --verbose` | 直接启动 Gateway（调试用） |
| `pnpm openclaw agent --message "..."`          | 与 Agent 对话（CLI）       |
| `pnpm openclaw doctor`                         | 配置与环境诊断             |
| `pnpm build`                                   | 构建，产出 `dist/`         |
| `pnpm test`                                    | 运行测试                   |

---

## 九、延伸阅读

- **项目根目录**：`README.md` — 安装、快速开始、功能列表。
- **官方文档**：<https://docs.openclaw.ai> — 概念、配置、各通道/工具/平台说明。
- **安全与默认策略**：<https://docs.openclaw.ai/gateway/security>（DM 配对、沙箱等）。
- **架构与协议**：<https://docs.openclaw.ai/concepts/architecture>。

仓库：<https://github.com/openclaw/openclaw>。

---

_本文档基于 `.cursor/rules/project-overview.mdc` 与根目录 `README.md` 整理，便于在 IDE 内快速查阅。_
