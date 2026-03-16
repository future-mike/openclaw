# 项目里那么多 .md 文件都是干什么的？

项目里 .md 很多，按**目录**分大致是这几类用途。

---

## 一、`docs/` — 官方文档站（Mintlify）

- **用途**：给 [docs.openclaw.ai](https://docs.openclaw.ai) 用的**文档正文**，按主题/功能组织（安装、Gateway、通道、CLI、工具、概念等）。
- **特点**：每个 .md 顶部有 **YAML frontmatter**（`summary`、`read_when`、`title` 等），方便导航和搜索。
- **子目录大致分工**：
  - `concepts/` — 概念（session、streaming、agent-workspace 等）
  - `cli/` — 各 CLI 子命令说明（doctor、channels、agent 等）
  - `channels/` — 各通道（WhatsApp、Telegram、Discord 等）
  - `gateway/` — Gateway 行为、沙箱、配对、健康检查等
  - `tools/` — 工具（exec、browser、skills、session-tool 等）
  - `install/`、`platforms/` — 安装与各平台
  - `automation/` — Cron、Hooks、Gmail 等
  - `nodes/`、`providers/`、`web/`、`start/` 等 — 对应领域
  - `reference/` — 参考（模板、API 成本等）
  - `experiments/`、`refactor/`、`design/` — 实验/重构/设计说明
- **多语言**：`docs/zh-CN/`、`docs/ja-JP/` 多为**从英文生成**的翻译，一般只改英文源文件，再跑 i18n 流程（见 `docs/.i18n/README.md`）。

---

## 二、`docs/reference/templates/` — 工作区模板

- **用途**：用户工作区里那些「标准文件名」的**参考模板**，例如：
  - `AGENTS.md`、`SOUL.md`、`USER.md` — 给 Agent 读的说明/人设/用户信息
  - `BOOT.md`、`BOOTSTRAP.md` — 引导/自举
  - `IDENTITY.md`、`HEARTBEAT.md`、`TOOLS.md` 等
- **作用**：`openclaw onboard` / setup 时会参考这些模板，在 workspace 里生成或补全对应文件；开发/写文档时也当「标准长什么样」的参考。

---

## 三、`skills/*/SKILL.md` — 每个 Skill 的说明

- **用途**：每个 **Skill** 一个目录，目录里必有 **SKILL.md**。
- **内容**：YAML frontmatter（name、description、metadata、homepage）+ 正文（何时用、命令示例、配置等）。
- **谁用**：Agent 选技能时会参考；ClawHub/文档也会用；用户装完技能也能直接读这个文件。

---

## 四、`extensions/*` 下的 .md

- **README.md**：该扩展的安装、配置、用途说明（给人看）。
- **CHANGELOG.md**：该扩展的版本更新说明。
- **SKILL.md**：若扩展里带「技能」，和 skills 一样是技能说明（如 `extensions/lobster/SKILL.md`）。
- 扩展内部可能还有子文档（如 open-prose 的 prose 相关 .md），属于该扩展自己的说明。

---

## 五、`src/hooks/bundled/*/HOOK.md` — 内置 Hook 说明

- **用途**：每个**内置 Hook** 一个目录，目录里的 **HOOK.md** 描述这个 Hook 做什么、触发哪些事件、怎么配置。
- **格式**：和 docs 里的 Hooks 文档类似，带 frontmatter（name、description、metadata、events 等），便于自动发现和展示。

---

## 六、根目录及零散的 .md

| 文件                  | 作用                                                                        |
| --------------------- | --------------------------------------------------------------------------- |
| **README.md**         | 项目总览、安装、快速开始、功能列表（GitHub 首页）                           |
| **AGENTS.md**         | 给 AI/维护者看的**仓库约定**（提交、测试、文档、安全等），Cursor/Codex 会读 |
| **CLAUDE.md**         | 一般是指向 AGENTS.md 的 symlink，让 Claude 用同一套约定                     |
| **VISION.md**         | 项目愿景/方向                                                               |
| **CHANGELOG.md**      | 版本发布说明（Changes / Fixes）                                             |
| **SECURITY.md**       | 安全策略、报告方式                                                          |
| **.github/** 下的 .md | Issue/PR 模板、bot 说明（如 `instructions/copilot.instructions.md`）        |

---

## 七、`docs/learning/` — 学习用文档（本项目新增）

- **project-intro.md** — 项目整体介绍与学习路径。
- **backend-execution-and-agents.md** — 后端执行、工具调用、会话隔离、智能体通信。
- **what-are-all-the-md-files.md** — 本文件，说明各目录 .md 的用途。

---

## 总结表

| 位置                                                   | 主要用途               |
| ------------------------------------------------------ | ---------------------- |
| `docs/**/*.md`（含 zh-CN、ja-JP）                      | 官方文档站页面         |
| `docs/reference/templates/*.md`                        | 工作区文件模板参考     |
| `skills/*/SKILL.md`                                    | 每个技能的说明与元数据 |
| `extensions/*/README.md`、`SKILL.md`、`CHANGELOG.md`   | 扩展说明与技能说明     |
| `src/hooks/bundled/*/HOOK.md`                          | 内置 Hook 说明         |
| 根目录 README / AGENTS / VISION / CHANGELOG / SECURITY | 仓库入口与规范         |
| `.github/**/*.md`                                      | 协作与 CI 模板/说明    |
| `docs/learning/*.md`                                   | 学习用归纳文档         |

所以：**大量 .md 主要是「文档站 + 技能/扩展/Hook 的说明 + 仓库与协作规范」**，按目录区分用途即可。
