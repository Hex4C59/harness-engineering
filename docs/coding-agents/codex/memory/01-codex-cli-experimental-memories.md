# Codex CLI 实验功能 Memories 调研

调研日期：2026-05-30

## 这篇文档回答什么

Codex CLI 里现在有一个实验功能叫 **Memories**。很多人会口头说成 `Memory` 或 `/memory`，但 OpenAI Codex 官方文档和 CLI slash command 当前使用的是复数形式：

```text
/memories
```

本文只记录 Codex Memories 的官方用法、源码观察和个人使用建议。跨工具的 coding agent memory 机制综述见 [`../../research/runtime/01-coding-agent-memory-mechanisms.md`](../../research/runtime/01-coding-agent-memory-mechanisms.md)。

本文回答：

1. Codex Memories 是什么。
2. 它和 `AGENTS.md`、当前会话、context compaction 有什么区别。
3. 官方文档能确认的使用方式是什么。
4. 从公开源码能观察到的大致实现原理是什么。
5. 这个功能和 agent memory 相关论文有什么关系。
6. 对个人 coding agent 工作流，应该怎么用、什么时候不用。

核心结论：

> Codex Memories 不是“模型真的学会了你的项目”，也不是把信息训练进模型权重。它是一套本地、可检查、默认关闭的外部记忆系统：Codex 从符合条件的历史 thread 中异步提取高信号经验，写入 `~/.codex/memories/` 等本地状态，再在未来 thread 中按配置注入或检索。它适合保存稳定偏好、反复工作流、已知坑点和项目经验，但不应该替代 `AGENTS.md`、项目 docs、测试、CI 和代码里的事实来源。

## 来源说明

本文按来源可靠性分层：

| 来源 | 类型 | 主要用途 |
|---|---|---|
| [OpenAI Codex Memories](https://developers.openai.com/codex/memories) | 官方文档 | Memories 的定义、开启方式、存储位置、限制和配置入口 |
| [OpenAI Codex Chronicle](https://developers.openai.com/codex/memories/chronicle) | 官方文档 | Chronicle 如何用屏幕上下文扩展 memory、隐私与安全风险 |
| [OpenAI Codex Configuration Reference](https://developers.openai.com/codex/config-reference) | 官方文档 | `features.memories` 和 `[memories]` 配置项 |
| [OpenAI Codex Slash Commands](https://developers.openai.com/codex/cli/slash-commands) | 官方文档 | `/memories` slash command 的用途 |
| [Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop) | 官方工程文章 | Codex agent loop、上下文组织、`AGENTS.md` 和 compaction 背景 |
| [Unlocking the Codex harness](https://openai.com/index/unlocking-the-codex-harness/) | 官方工程文章 | Codex harness、thread lifecycle、persistence、App Server 架构 |
| [Running Codex safely at OpenAI](https://openai.com/index/running-codex-safely/) | 官方安全文章 | agent 安全边界、权限、telemetry、审计 |
| `openai/codex` 源码，commit `cb9178e` | 公开源码观察 | 两阶段 memory pipeline、SQLite 表、TUI 设置项、文件布局 |
| [Claude Code memory docs](https://code.claude.com/docs/en/memory) | 官方文档 | 对比 `CLAUDE.md` 与 auto memory 的产品设计 |
| [Cursor Rules docs](https://cursor.com/docs/rules) | 官方文档 | 对比 rules / `AGENTS.md` 这种 prompt-level persistent context |
| 论文：Generative Agents、Reflexion、MemoryBank、MemGPT、Voyager、A-Mem | 论文 | 建立 agent memory 的设计谱系 |

需要区分：

- **官方事实**：OpenAI 文档明确写出的行为，例如默认关闭、`~/.codex/memories/`、`/memories`、配置项。
- **源码观察**：从 `openai/codex` 当前公开源码看到的实现形态，可能随版本变化。
- **个人整理框架**：面向个人 coding agent 工作流的建议，不代表官方承诺。

本机观察：

```text
Codex CLI: 0.135.0
codex features list: memories experimental false
```

也就是说，截至本次调研，本机 `memories` feature 仍是 experimental，且默认未启用。

## Memories 是什么

OpenAI 官方文档把 Memories 定义为：让 Codex 把早先 thread 中有用的上下文带到未来工作中。

它适合记住：

- 稳定个人偏好。
- 反复出现的工作流。
- 常用技术栈。
- 项目约定。
- 已知坑点。

它不适合承载：

- 团队必须遵守的规则。
- 安全策略。
- 当前任务的唯一事实来源。
- 密钥、token、密码、客户数据。
- 大段代码、完整日志、一次性调试输出。

一个简化模型：

```text
历史 thread
-> 提取高信号经验
-> 本地 memory 文件 / SQLite 状态
-> 新 thread 中注入或检索相关 memory
-> 模型基于这些额外上下文行动
```

注意，memory 并不改变模型权重。它更接近“外部可检索上下文层”。

## 和 AGENTS.md、Compaction 的区别

这三个机制很容易混：

| 机制 | 生命周期 | 谁写 | 主要作用 | 可靠性 |
|---|---|---|---|---|
| `AGENTS.md` | 随项目长期存在 | 人写，agent 可协助 | 项目硬规则、入口地图、命令、协作约定 | 高，可 review、可版本控制 |
| Memories | 跨 thread，本地生成状态 | Codex 后台生成，也可能由用户审查 | 稳定偏好、项目经验、反复坑点 | 中，需要审查，可能过期 |
| Compaction | 当前 thread 内 | Codex / API | 压缩长上下文，让当前任务继续 | 中，只适合任务延续 |

一句话：

```text
AGENTS.md 管“必须怎么做”
docs 管“项目事实是什么”
Memories 管“以前学到的可复用经验”
Compaction 管“这次长任务怎么继续”
```

如果一条信息“必须每次都遵守”，不要只放 memory。应该写进 `AGENTS.md`、项目 docs、hook、测试或 CI。

## 怎么开启

官方文档给出两种入口。

第一种是在 Codex App 设置里启用 Memories。

第二种是在 `~/.codex/config.toml` 中加 feature flag：

```toml
[features]
memories = true
```

CLI 也可以查看 feature 状态：

```bash
codex features list
```

本机 `0.135.0` 中可见：

```text
memories  experimental  false
```

也可以使用 CLI feature 子命令写入配置：

```bash
codex features enable memories
codex features disable memories
```

如果只想临时开启一次会话，可以使用命令行覆盖：

```bash
codex --enable memories
```

## 常用配置

官方配置参考中，Memories 相关常用项如下：

```toml
[features]
memories = true

[memories]
generate_memories = true
use_memories = true
disable_on_external_context = false
min_rate_limit_remaining_percent = 25
```

含义：

| 配置 | 默认倾向 | 作用 |
|---|---|---|
| `features.memories` | `false` | 总开关，启用 Memories 功能 |
| `memories.generate_memories` | `true` | 新 thread 是否可作为未来 memory 生成材料 |
| `memories.use_memories` | `true` | 新 thread 是否注入已有 memories |
| `memories.disable_on_external_context` | `false` | 使用 MCP、web search、tool search 等外部上下文时，是否禁止该 thread 参与 memory 生成 |
| `memories.min_rate_limit_remaining_percent` | `25` | 低于指定剩余额度时跳过后台 memory 生成 |
| `memories.extract_model` | 未指定 | 覆盖每个 thread 的 memory 提取模型 |
| `memories.consolidation_model` | 未指定 | 覆盖全局 memory 合并模型 |

配置参考还列出这些控制项：

| 配置 | 作用 |
|---|---|
| `memories.max_raw_memories_for_consolidation` | 控制参与全局合并的近期 raw memories 上限 |
| `memories.max_unused_days` | 太久未使用的 memory 不再参与合并 |
| `memories.max_rollout_age_days` | 只考虑一定天数内的历史 thread |
| `memories.max_rollouts_per_startup` | 每次启动最多处理多少候选 thread |
| `memories.min_rollout_idle_hours` | thread 空闲多久后才考虑提取，避免总结仍在进行的工作 |

如果你担心外部内容污染长期记忆，可以优先使用：

```toml
[memories]
disable_on_external_context = true
```

这样用过 MCP、web search 或 tool search 的 thread 不会轻易进入 memory 生成。

如果你只想读已有 memories，不想后台生成新 memories，可以用：

```toml
[memories]
generate_memories = false
use_memories = true
```

这对控制 token / rate limit 消耗有意义。

## /memories 做什么

OpenAI slash command 文档中，`/memories` 的用途是：

```text
Configure memory use and generation.
```

也就是在 Codex TUI 或 App 中控制当前 thread 的 memory 行为：

- 当前 thread 是否使用已有 memories。
- 当前 thread 是否允许作为未来 memory 生成材料。
- 这些选择不会改掉全局配置。

公开源码里的 TUI 面板也能看到三个菜单项：

| 菜单项 | 作用 |
|---|---|
| `Use memories` | 后续 thread 使用 memories，通常在下一个 thread 生效 |
| `Generate memories` | 当前 thread 也可参与生成 memories |
| `Reset all memories` | 清理当前 Codex home 下的本地 memory 文件和摘要，已有 threads 不受影响 |

这里也能看出，Codex 把“读 memory”和“写 memory”拆开控制。

## 存在哪里

官方文档说，Codex home 默认是：

```text
~/.codex
```

主要 memory 文件在：

```text
~/.codex/memories/
```

这些文件包括：

- summaries
- durable entries
- recent inputs
- supporting evidence from prior threads

源码观察中还有一个 SQLite 状态库，例如本机能看到：

```text
~/.codex/memories_1.sqlite
```

其中核心表包括：

```sql
stage1_outputs
jobs
```

`stage1_outputs` 保存每个 thread 的阶段 1 输出：

```text
thread_id
source_updated_at
raw_memory
rollout_summary
rollout_slug
generated_at
usage_count
last_usage
selected_for_phase2
```

`jobs` 负责后台任务的 lease、retry、watermark 等状态。

这些都应该看作 **generated state**，不是主要编辑入口。需要稳定规则时，仍然写入 `AGENTS.md` 或 docs。

## 实现原理：官方可确认部分

官方文档确认了这些行为：

1. Memories 默认关闭。
2. 启用后，Codex 可以把 eligible prior threads 转成本地 memory 文件。
3. Codex 会跳过 active 或 short-lived sessions。
4. Codex 会等待 thread 足够空闲，避免总结仍在进行的工作。
5. 生成 memory 会在后台更新，不是每个 thread 结束立刻同步完成。
6. 生成字段会做 secrets redaction，但用户仍需审查。
7. 当 Codex rate limit 剩余额度低于阈值时，memory 生成可被跳过。
8. `/memories` 可以控制当前 thread 是否用 memory、是否参与未来 memory 生成。

官方文档没有把内部 pipeline 每个实现细节都当作产品 API 承诺。所以接下来的实现说明属于源码观察。

## 实现原理：源码观察

`openai/codex` 当前源码中，memory pipeline 拆成两个 crate：

| crate | 职责 |
|---|---|
| `codex-rs/memories/read` | 读路径：memory 注入、citation 解析、读使用 telemetry |
| `codex-rs/memories/write` | 写路径：阶段 1 / 阶段 2 prompt、文件 artifact、workspace diff、extension pruning |

源码文档说明，pipeline 在 root session 启动时触发，但需要满足：

- 不是 ephemeral session。
- memory feature 已启用。
- 不是 sub-agent session。
- state DB 可用。

启动后，它异步执行：

```text
Phase 1: rollout extraction
-> Phase 2: global consolidation
```

### Phase 1：单个 Thread 提取

Phase 1 做的是“把历史 thread 转成结构化 raw memory”。

简化流程：

```text
从 state DB claim 一批 eligible rollouts
-> 读取 rollout items
-> 过滤适合 memory 的 response items
-> 调模型生成结构化输出
-> redaction
-> 写入 stage1_outputs
```

阶段 1 输出 schema 是：

```json
{
  "rollout_summary": "string",
  "rollout_slug": "string | null",
  "raw_memory": "string"
}
```

源码里的筛选逻辑还会避免把某些上下文片段当作 durable memory 输入，例如：

- `AGENTS.md` 注入片段。
- `<skill>...</skill>` 技能内容片段。

这很合理：`AGENTS.md` 和 skills 本来就是稳定上下文来源，不应该被再次总结成不透明 memory，避免重复、污染和失真。

### Phase 2：全局合并

Phase 2 做的是“把阶段 1 的 raw memories 合并成可长期使用的本地 memory workspace”。

简化流程：

```text
claim 全局 phase-2 lock
-> 准备 ~/.codex/memories/ workspace
-> 从 DB 选择一批 stage1_outputs
-> 同步 raw_memories.md 和 rollout_summaries/
-> 生成 phase2_workspace_diff.md
-> 如 workspace 有变化，启动内部 consolidation agent
-> consolidation agent 更新 MEMORY.md、memory_summary.md、skills/
-> 成功后重置 memory workspace baseline
```

源码文档里能看到，`~/.codex/memories/` 内部被当成一个 git-baseline workspace 使用。这不是为了让用户提交，而是为了让 consolidation agent 看见“这次 memory 输入相对上次成功合并发生了什么变化”。

Phase 2 的内部 consolidation agent 被锁得很紧：

- `ephemeral = true`
- `generate_memories = false`
- `use_memories = false`
- 禁用 MCP servers
- 禁用 apps / plugins / recursive collaboration
- approval policy 为 `Never`
- sandbox 只允许写 memory root，本地无网络

这说明 memory consolidation 被设计成一个受限后台 agent，不是普通用户工作线程。

### 文件形态

源码的 Phase 2 prompt 要求 memory folder 支持 progressive disclosure。典型结构：

```text
~/.codex/memories/
├── memory_summary.md
├── MEMORY.md
├── raw_memories.md
├── rollout_summaries/
└── skills/
```

各文件大致职责：

| 文件 / 目录 | 作用 |
|---|---|
| `memory_summary.md` | 高密度导航摘要，供未来 agent 快速判断该查什么 |
| `MEMORY.md` | durable handbook，按任务组聚合经验 |
| `raw_memories.md` | 阶段 1 raw memories 的机械合并输入 |
| `rollout_summaries/` | 每个被选中 thread 的摘要文件 |
| `skills/` | 可能沉淀出的可复用 procedure |

这套结构很像“先粗粒度注入导航，再按需检索细节”的 progressive disclosure 设计。

## Chronicle：屏幕上下文增强 Memory

Chronicle 是 Codex App 里的一个 opt-in research preview，不等同于普通 CLI Memories，但它扩展了 memory 的来源。

官方文档说明：

- Chronicle 目前是 macOS 上的研究预览。
- 需要 ChatGPT Pro。
- 需要 macOS Screen Recording 和 Accessibility 权限。
- 会用最近屏幕上下文帮助生成 memories。
- 屏幕截图临时保存在本机，过旧截图会被清理。
- 生成的 Chronicle memories 存在 `$CODEX_HOME/memories_extensions/chronicle/`。
- 生成 memories 会消耗 rate limits。
- 会增加 prompt injection 风险。
- 生成的 memories 是未加密 Markdown 文件。

对 CLI 用户来说，Chronicle 的启发是：memory 越“自动”和“环境感知”，越要重视隐私、prompt injection 和误记风险。

## 论文脉络：Agent Memory 解决什么问题

Codex Memories 在产品上是 coding agent 的记忆功能，但它背后的设计问题和近几年 agent memory 论文高度一致：

```text
固定上下文窗口
-> 长任务和跨会话会丢状态
-> 需要外部 memory store
-> 需要选择、压缩、检索、合并、遗忘
```

下面按设计思想梳理几篇关键论文。

### Generative Agents

[Generative Agents: Interactive Simulacra of Human Behavior](https://arxiv.org/abs/2304.03442) 是 agent memory 经典起点之一。

它提出的架构包含：

- memory stream：保存 agent 观察到的经验。
- retrieval：根据当前情境检索相关 memories。
- reflection：把低层经验综合成高层反思。
- planning：用记忆和反思生成长期计划。

对 Codex Memories 的启发：

```text
不是保存完整历史，而是把经验沉淀成可检索、可计划、可复用的上下文。
```

### Reflexion

[Reflexion: Language Agents with Verbal Reinforcement Learning](https://arxiv.org/abs/2303.11366) 的核心是：agent 不更新模型权重，而是把任务反馈转成语言反思，放进 episodic memory buffer，让后续尝试变好。

对 coding agent 很直接：

```text
测试失败、用户纠正、review 反馈
-> 总结成“下次怎么避免”
-> 放进外部记忆
-> 下次同类任务少犯错
```

这解释了为什么 Codex Phase 1 prompt 会强调 failure patterns、verification checklists、known landmines。

### MemoryBank

[MemoryBank: Enhancing Large Language Models with Long-Term Memory](https://arxiv.org/abs/2305.10250) 关注长期对话里的用户画像和个性化记忆。

它强调：

- 从历史交互中召回相关 memories。
- 持续更新用户理解。
- 借鉴遗忘曲线，根据时间和重要性做选择性保留与强化。

对 Codex Memories 的启发：

```text
不是所有历史都值得记住；memory 需要选择、更新和遗忘。
```

Codex 配置中的 `max_unused_days`、`max_rollout_age_days`、`max_raw_memories_for_consolidation` 可以看作产品实现里的保留窗口和选择机制。

### MemGPT

[MemGPT: Towards LLMs as Operating Systems](https://arxiv.org/abs/2310.08560) 把 LLM 上下文管理类比成操作系统的虚拟内存：固定 context window 相当于主存，外部存储相当于磁盘，agent 需要主动读写外部 memory。

对 Codex Memories 的启发：

```text
Context window 不是无限的。
Agent runtime 需要决定哪些信息留在当前 prompt，哪些放进外部 memory，什么时候再取回来。
```

这和 Codex 把 `memory_summary.md`、`MEMORY.md`、`rollout_summaries/` 分层存储很接近。

### Voyager

[Voyager: An Open-Ended Embodied Agent with Large Language Models](https://openreview.net/forum?id=ehfRiF0R3a) 在 Minecraft 环境中维护一个不断增长的 skill library，把成功行为保存为可复用代码技能。

对 coding agent 的启发：

```text
高价值 memory 不一定只是“事实”，也可以是“可执行 procedure”。
```

这解释了为什么 Codex Phase 2 的 memory folder 允许生成 `skills/`：如果某类工作流反复出现，最好的记忆可能不是一句话，而是一套可复用步骤或脚本。

### A-Mem

[A-Mem: Agentic Memory for LLM Agents](https://openreview.net/forum?id=FiM0M8gcct) 更进一步，主张 memory 不只是平铺的存储和检索，而应能动态组织、链接和演化。它借鉴 Zettelkasten，把 memory 做成带上下文、关键词、标签和链接的知识网络。

对 Codex Memories 的启发：

```text
长期 memory 的难点不是“能存”，而是“怎么组织、怎么更新、怎么避免旧信息误导新任务”。
```

Codex 当前的 `MEMORY.md`、`memory_summary.md`、`rollout_summaries/` 是偏文件系统和 progressive disclosure 的做法；A-Mem 则代表更 graph-like、self-organizing 的研究方向。

## 博客和技术报告里的实践线索

除了论文，产品和社区实践里有几个共同趋势。

### Claude Code：CLAUDE.md + Auto Memory

Claude Code 官方 memory 文档把记忆拆成两类：

| 机制 | 谁写 | 放什么 |
|---|---|---|
| `CLAUDE.md` | 用户 / 团队 | instructions、rules、架构、工作流 |
| Auto memory | Claude | build commands、debugging insights、preferences、发现的 patterns |

Claude 的 auto memory 存在：

```text
~/.claude/projects/<project>/memory/
```

这个设计和 Codex Memories 的结论一致：硬规则用可版本控制文件，自动 memory 用于“学到的经验”。

### Cursor：Rules / AGENTS.md 是 prompt-level persistent context

Cursor 官方 rules 文档明确说：大模型不会在 completion 之间保留记忆，rules 是在 prompt 层提供 persistent reusable context。

Cursor 支持：

- Project Rules：`.cursor/rules`
- User Rules
- Team Rules
- `AGENTS.md`

这说明另一个方向：不自动生成 memory，而是让人把稳定知识写进规则文件，由 agent 在开头加载。

对 Codex 用户来说：

```text
AGENTS.md / rules = 可控、显式、可共享
Memories = 自动、辅助、需要审查
```

### 社区反馈：自动生成要有可见性和边界

OpenAI Codex GitHub issues / discussions 中，memory 相关反馈集中在：

- 希望有 project-scoped memory，避免跨项目污染。
- 希望 memory citations 更可见，方便知道哪个 memory 影响了行为。
- 后台生成会消耗 rate limit，需要更清楚的可见性和控制。
- 对 coding agent 来说，项目约定比纯全局偏好更重要，但错误项目记忆的代价也更高。

这些反馈不一定代表官方实现，但很适合用来制定个人使用策略。

### 官方工程文章：Memory 是 harness 问题

OpenAI 的 Codex 工程文章没有把 Memories 当作孤立功能，而是放在更大的 harness 设计里理解。

- `Unrolling the Codex agent loop` 解释了 Codex 如何把用户输入、工具定义、`AGENTS.md`、权限说明、历史 item 和工具输出组织成模型上下文；这说明 memory 的本质是 context management 的一个长期层。
- `Unlocking the Codex harness` 解释了 Codex core、thread lifecycle、persistence、App Server 和多端复用；这说明 memory 需要和 thread 存储、resume、fork、App / CLI / IDE surfaces 协同，而不是单独一份 prompt 文件。
- `Running Codex safely at OpenAI` 强调 sandbox、approval、rules 和 telemetry；这提醒我们 memory 不能替代安全边界，反而应该被纳入可审查、可控制、可审计的 agent runtime。

换句话说：

```text
Memories = 长期上下文层
但真正可靠的 coding agent = memory + docs + rules + sandbox + approval + tests + telemetry
```

## 怎么用比较稳

推荐从保守配置开始：

```toml
[features]
memories = true

[memories]
use_memories = true
generate_memories = false
disable_on_external_context = true
min_rate_limit_remaining_percent = 25
```

含义：

- 先允许 Codex 读已有 memories。
- 暂时不让后台自动生成新 memories。
- 用过 MCP / web search / tool search 的 thread 不进入 memory 生成。
- 避免额度较低时继续跑后台 memory。

当你确认 memory 文件质量、隐私边界和 quota 消耗可接受后，再打开：

```toml
[memories]
generate_memories = true
```

日常使用建议：

1. 每隔一段时间检查 `~/.codex/memories/`。
2. 发现错记、过期、敏感内容时删除或清理相关文件。
3. 经常重复出现的硬规则，提升到 `AGENTS.md`。
4. 经常重复出现的长流程，提升为 docs、skill 或 script。
5. 当前任务状态不要只靠 memory，写进 plan/status 文档。

## 哪些信息该放哪里

| 信息类型 | 推荐位置 | 原因 |
|---|---|---|
| “所有正式文档默认简体中文” | `AGENTS.md` | 必须稳定执行 |
| “这个项目文档按 `NN-topic.md` 命名” | `AGENTS.md` / README | 团队共享约定 |
| “我个人喜欢先调研再写文档” | 用户级 `AGENTS.md` / memory | 稳定个人偏好 |
| “某次调研发现 Codex Memories 默认关闭” | 当前文档 / references | 需要日期和来源 |
| “某个测试第一次失败常因服务未 ready” | docs/testing.md，也可进入 memory | 可复用项目经验 |
| “这次任务写到第 3 步，还差第 4 步” | plan/status 文档 | 当前任务状态，不适合长期 memory |
| API key、token、cookie | 不要存 | 敏感信息 |

## 风险和边界

### 1. 错误记忆会污染未来任务

Memory 一旦被注入，模型可能把它当作背景事实。错误 memory 的影响比一次错误回答更隐蔽。

缓解方式：

- 定期审查 `~/.codex/memories/`。
- 把重要事实写进 docs，让 agent 可以重新验证。
- 对跨项目差异大的工作，考虑分 CODEX_HOME 或关闭 generate。

### 2. 自动生成会消耗额度

官方文档明确 memory generation 受 rate-limit 阈值控制；Chronicle 文档也提醒相关后台 agent 会较快消耗 rate limits。

缓解方式：

- 对默认工作流使用 `generate_memories = false`。
- 只在阶段性复盘或长项目里打开生成。
- 设置较保守的 `max_rollouts_per_startup` 和 `min_rollout_idle_hours`。

### 3. 外部上下文可能带来 prompt injection

如果 thread 使用了 web search、MCP 或屏幕上下文，里面可能含有第三方指令。把这些内容总结进长期 memory，会放大污染风险。

缓解方式：

```toml
[memories]
disable_on_external_context = true
```

Chronicle 场景尤其要谨慎，因为屏幕内容本身可能包含恶意或敏感文本。

### 4. 本地文件未必加密

官方 Chronicle 文档明确说 Chronicle 生成的 memories 是本地未加密 Markdown。普通 Memories 文档也要求在共享 Codex home 前审查 memory 文件。

缓解方式：

- 不把 `~/.codex` 打包分享。
- 不把 memory 目录提交到项目仓库。
- 清理敏感或客户相关内容。

### 5. 它不是审计日志

Memory 是提炼后的经验，不是完整历史。要审计行为，应看 thread history、git diff、terminal logs、CI、review 记录。

## 建议的个人工作流

对这个研究型仓库，可以这样用：

1. `AGENTS.md` 继续保存协作规则、目录结构、写作约定。
2. `docs/coding-agents/codex/` 保存 Codex 专属机制详解，通用 runtime 研究放在 `docs/coding-agents/research/runtime/`。
3. Memories 只用来辅助记住个人偏好和反复踩坑。
4. 每次完成重要调研，优先写成正式 `.md` 文档，不依赖 memory。
5. 如果 memory 里出现有价值经验，把它人工提升成文档或 playbook。

推荐配置：

```toml
[features]
memories = true

[memories]
use_memories = true
generate_memories = false
disable_on_external_context = true
min_rate_limit_remaining_percent = 25
max_rollouts_per_startup = 4
min_rollout_idle_hours = 12
```

等你明确需要 Codex 自动复盘多个长 session 时，再临时打开：

```toml
[memories]
generate_memories = true
```

## 参考资料

官方文档：

- [OpenAI Codex: Memories](https://developers.openai.com/codex/memories)
- [OpenAI Codex: Chronicle](https://developers.openai.com/codex/memories/chronicle)
- [OpenAI Codex: Configuration Reference](https://developers.openai.com/codex/config-reference)
- [OpenAI Codex CLI: Slash commands](https://developers.openai.com/codex/cli/slash-commands)
- [Claude Code: How Claude remembers your project](https://code.claude.com/docs/en/memory)
- [Cursor Docs: Rules](https://cursor.com/docs/rules)

技术报告 / 工程文章：

- [OpenAI: Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop)
- [OpenAI: Unlocking the Codex harness](https://openai.com/index/unlocking-the-codex-harness/)
- [OpenAI: Running Codex safely at OpenAI](https://openai.com/index/running-codex-safely/)
- [OpenAI: How OpenAI uses Codex](https://cdn.openai.com/pdf/6a2631dc-783e-479b-b1a4-af0cfbd38630/how-openai-uses-codex.pdf)

源码：

- [openai/codex: `codex-rs/memories/README.md`](https://github.com/openai/codex/blob/main/codex-rs/memories/README.md)
- [openai/codex: memory write pipeline](https://github.com/openai/codex/tree/main/codex-rs/memories/write)
- [openai/codex: memory read crate](https://github.com/openai/codex/tree/main/codex-rs/memories/read)
- [openai/codex: memories TUI settings view](https://github.com/openai/codex/blob/main/codex-rs/tui/src/bottom_pane/memories_settings_view.rs)

论文：

- [Generative Agents: Interactive Simulacra of Human Behavior](https://arxiv.org/abs/2304.03442)
- [Reflexion: Language Agents with Verbal Reinforcement Learning](https://arxiv.org/abs/2303.11366)
- [MemoryBank: Enhancing Large Language Models with Long-Term Memory](https://arxiv.org/abs/2305.10250)
- [MemGPT: Towards LLMs as Operating Systems](https://arxiv.org/abs/2310.08560)
- [Voyager: An Open-Ended Embodied Agent with Large Language Models](https://openreview.net/forum?id=ehfRiF0R3a)
- [A-Mem: Agentic Memory for LLM Agents](https://openreview.net/forum?id=FiM0M8gcct)

社区和博客：

- [Mem0: Codex CLI Memory: How It Works + What Mem0 Adds](https://mem0.ai/blog/how-memory-works-in-codex-cli)
- [OpenAI Codex GitHub Discussion: Memories in Codex](https://github.com/openai/codex/discussions/12567)
- [OpenAI Codex GitHub Issue: Long-term Memory](https://github.com/openai/codex/issues/8368)
- [OpenAI Codex GitHub Issue: Scoped memory management](https://github.com/openai/codex/issues/18343)
