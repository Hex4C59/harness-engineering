# Coding Agent Memory 机制详解

调研日期：2026-05-29

## 这篇文档回答什么

这篇文档从 Codex CLI Memories 出发，整理主流 coding agent 的 memory 机制、边界和使用策略。

先把一个容易混淆的点说清楚：

- OpenAI Codex 官方文档当前写的是 **`/memories`**，不是 `/memory`。
- Claude Code 官方文档里的命令叫 **`/memory`**。
- 很多用户会把 `/memory` 当成泛称，用来指“agent 的记忆管理入口”。

所以本文会同时解释：

1. Codex CLI / Codex App 的 **Memories** 是什么。
2. `AGENTS.md`、session、compaction、memory 分别有什么区别。
3. Claude Code、Cursor、GitHub Copilot 等 coding agent 怎么做 memory。
4. 现在主流 agent memory 的底层设计大概是什么样子。
5. 作为程序员，应该把哪些信息写进 `AGENTS.md`、哪些写进 docs、哪些交给 memory。

核心结论：

> Coding agent 的 memory 不是“模型真的学会了你的项目”，也不是“训练进模型权重”。它更像一套可检索的外部上下文系统：把稳定偏好、项目事实、常见坑、历史经验保存下来，在未来会话开始或需要时再注入模型上下文。它有用，但不可靠到可以替代 `AGENTS.md`、docs、测试和代码里的事实来源。

## 先建立基本概念

讨论 memory 前，先区分 6 个相近但不同的东西：

| 名称 | 它是什么 | 生命周期 | 适合放什么 | 不能当成什么 |
|---|---|---|---|---|
| 当前上下文窗口 | 当前这一轮模型实际能看到的 tokens | 单次会话 / 单次模型调用 | 当前任务、相关文件、命令输出 | 长期记忆 |
| 会话历史 | 本次 thread 里的聊天、工具调用、diff、日志 | 当前 thread，可 resume | 未完成任务上下文 | 项目规范 |
| 上下文压缩 | 把长会话压成较小工作状态 | 当前 thread 继续用 | 任务进度、关键决策 | 完整审计日志 |
| 项目指令 | `AGENTS.md`、`CLAUDE.md`、rules | 随项目长期存在 | 硬约定、入口地图、命令 | 大百科 |
| 生成式 memory | agent 从过往线程里提取的可复用信息 | 跨 thread，通常本机或账号级 | 稳定偏好、常见坑、项目经验 | 必须执行的规则 |
| 模型训练 | 更新模型权重 | 产品/研究训练周期 | 通用能力 | 你的本地项目记忆 |

一个简单类比：

```text
当前上下文窗口 = 桌面
会话历史 = 这次工作的草稿纸
compaction = 把草稿纸整理成继续工作的摘要包
AGENTS.md / docs = 项目手册
memory = 助手自己做的长期便签
模型训练 = 改变助手的大脑参数
```

这几个层级经常一起出现，但作用完全不同。真正可控、可 review、可共享的长期事实，仍然应该落到仓库文件里。

## Codex CLI 的 Memories 是什么

根据 OpenAI Codex 官方文档，Codex Memories 是一个默认关闭的功能。开启后，Codex 可以把早先 thread 里的有用上下文变成本地 memory 文件，并在未来工作中带入相关信息。

它适合记住：

- 稳定的个人偏好。
- 反复出现的工作流。
- 常用技术栈。
- 项目约定。
- 已知坑点。

它不适合承载：

- 必须遵守的团队规则。
- 安全策略。
- 当前任务唯一事实。
- 密钥、token、个人隐私和客户数据。
- 大段代码、长模板、完整日志。

OpenAI 文档特别强调：团队必须遵守的指导应该继续放在 `AGENTS.md` 或 checked-in documentation 中，memory 只是本地辅助 recall 层。

## Codex 里怎么开启 Memories

Codex 官方文档给了两种方式。

第一种是在 Codex App 设置里开启 Memories。

第二种是在 `~/.codex/config.toml` 里打开 feature flag：

```toml
[features]
memories = true
```

Codex 当前还提供一些 memory 相关配置项，常见的有：

```toml
[memories]
generate_memories = true
use_memories = true
disable_on_external_context = false
min_rate_limit_remaining_percent = 25
```

含义可以这样理解：

| 配置 | 作用 |
|---|---|
| `generate_memories` | 新 thread 是否可作为未来 memory 生成材料 |
| `use_memories` | 新会话是否注入已有 memories |
| `disable_on_external_context` | 使用 MCP、web search、tool search 等外部上下文时，是否禁止该 thread 参与 memory 生成 |
| `min_rate_limit_remaining_percent` | 低于一定额度时跳过后台 memory 生成，避免消耗额度 |
| `extract_model` | 指定每个 thread 的 memory 提取模型 |
| `consolidation_model` | 指定全局 memory 合并模型 |

如果你很在意隐私和上下文污染，可以考虑把 `disable_on_external_context` 设为 `true`。这样用了联网搜索、MCP 或其他外部上下文的 thread，不会轻易变成长期 memory 来源。

## Codex 的 `/memories` 做什么

OpenAI 文档说，在 Codex App 和 Codex TUI 里，可以使用 `/memories` 控制当前 thread 的 memory 行为。

它的核心不是“让模型立刻记住一句话”，而是管理当前 thread 和 memory 系统的关系：

- 当前 thread 是否可以使用已有 memories。
- 当前 thread 是否可以作为未来生成 memories 的材料。
- 这些选择只影响当前 thread，不会改掉全局 memory 配置。

本机 Codex CLI 版本 `0.133.0` 中，`codex debug prompt-input` 可以确认当前项目的 `AGENTS.md` 会被注入模型可见上下文。Codex CLI 的交互界面和官方文档都显示 Memories 是一个独立 feature，并通过 `/memories` 入口管理。

注意：如果你在某些文章或讨论里看到 `/memory`，要先判断它是不是在说 Claude Code。Codex 官方当前叫 `/memories`。

## Codex memory 存在哪里

Codex 官方文档说明，memory 存在 Codex home 目录下。默认是：

```text
~/.codex/memories/
```

里面会包括：

- summaries
- durable entries
- recent inputs
- supporting evidence from prior threads

这些文件应该当成 **生成状态**，不是主要编辑入口。

也就是说：

- 可以检查它，尤其是排查问题或共享 `.codex` 目录前。
- 不建议把手工编辑它当成主要控制方式。
- 需要明确规则时，写入 `AGENTS.md` 或项目 docs。

## Codex memory 如何生成

OpenAI 文档没有公开全部内部实现，但公开信息能确认几个关键点：

1. Memories 不会马上在每个 thread 结束时生成。
2. Codex 会跳过 active 或 short-lived sessions。
3. Codex 会等 thread 空闲一段时间，避免总结仍在进行中的工作。
4. Codex 会在后台更新 memories。
5. 生成字段会做 secrets redaction，但用户仍然要自己审查。
6. 剩余额度太低时，memory 生成可能被跳过。

从 harness 设计角度看，这很合理。memory 写入通常会被设计成异步后台任务：

```text
thread 结束或空闲
-> 判断是否 eligible
-> 提取候选 memory
-> 脱敏 / 去重 / 合并
-> 写入本地 memory store
-> 未来 thread 检索相关 memory
-> 注入模型上下文
```

这样不会让每次对话都变慢，也能避免把半截任务直接沉淀成错误长期事实。

## Codex memory、AGENTS.md、compaction 的区别

这三个东西最容易混。

| 机制 | 作用 | 谁写 | 何时加载 | 是否可靠 |
|---|---|---|---|---|
| `AGENTS.md` | 项目 / 用户长期指令 | 人写，agent 可协助 | Codex 启动或新 session 时发现并注入 | 最高，因可 review、可版本控制 |
| Memories | 从旧 thread 中提取可复用信息 | Codex 后台生成 | 未来 thread 按配置使用 | 中等，需审查，可能过期 |
| Compaction | 压缩当前长会话 | Codex / API | 当前 thread 快到上下文限制时 | 只适合延续当前任务 |

一句话：

```text
AGENTS.md 管“应该怎么做”
docs 管“项目事实是什么”
memory 管“以前学到的可复用经验”
compaction 管“这次长任务怎么继续”
```

如果一条信息对团队很重要，要写进 `AGENTS.md` 或 docs，而不是只放 memory。

比如：

| 信息 | 应该放哪里 |
|---|---|
| 每次改代码后必须运行 `scripts/check` | `AGENTS.md` |
| 项目整体架构、模块边界 | docs |
| 这个项目的测试数据库启动很慢，第一次失败常因容器未 ready | docs/testing.md，也可以进入 memory |
| 你个人偏好“先 scout 再实现” | `~/.codex/AGENTS.md` 或 memory |
| 本次任务做到 slice 2，slice 3 还没做 | plan/status 文档，不要只靠 memory |
| 某次调试日志里出现的 token | 不要进任何 memory |

## Claude Code 的 `/memory`

Claude Code 官方文档中的 `/memory` 是一个明确的 memory 管理入口。

Claude Code 的跨会话知识主要分两类：

1. `CLAUDE.md` 文件：人写的持久项目指令。
2. Auto memory：Claude 自己根据你的纠正、偏好和项目经验写下的 notes。

Claude 官方文档明确说，每个 Claude Code session 都从 fresh context window 开始，跨 session 的知识由 `CLAUDE.md` 和 auto memory 承担。

### CLAUDE.md

`CLAUDE.md` 类似 Codex 的 `AGENTS.md`。

它适合写：

- build / test / lint 命令。
- 项目架构。
- coding standards。
- 命名约定。
- 常见工作流。
- 必须知道的项目坑点。

Claude 官方建议 `CLAUDE.md` 保持精简，目标是每个文件少于 200 行。太长会消耗上下文，并降低遵守程度。

如果项目已经有 `AGENTS.md`，Claude 官方建议可以创建一个 `CLAUDE.md` 并导入：

```markdown
@AGENTS.md

## Claude Code

Use plan mode for changes under `src/billing/`.
```

这样不用维护两份重复规则。

### Auto memory

Claude Code 的 auto memory 会把 Claude 在工作中学到的 build commands、debugging insights、architecture notes、style preferences、workflow habits 保存下来。

官方文档描述的存储位置是：

```text
~/.claude/projects/<project>/memory/
```

典型结构：

```text
~/.claude/projects/<project>/memory/
├── MEMORY.md
├── debugging.md
├── api-conventions.md
└── ...
```

其中 `MEMORY.md` 是入口索引。Claude 在每次会话开始时加载 `MEMORY.md` 的前 200 行或前 25KB，较详细的 topic files 会按需读取。

`/memory` 在 Claude Code 中可以：

- 列出当前 session 加载的 `CLAUDE.md`、`CLAUDE.local.md`、rules 文件。
- 开关 auto memory。
- 打开 auto memory 文件夹。
- 让你查看、编辑、删除 memory 文件。

这是一个很值得借鉴的设计：入口索引短，详细记忆拆到 topic files，避免每次启动都把所有历史塞进上下文。

## Cursor 的 rules：更像显式记忆

Cursor 官方文档直接说：大语言模型不会在 completion 之间保留 memory，Rules 提供的是 prompt 层面的持久可复用上下文。

Cursor 主要有：

- Project Rules：存在 `.cursor/rules/`，可版本控制。
- User Rules：全局个人偏好。
- Team Rules：团队/企业规则。
- `AGENTS.md`：简单 markdown 指令入口。
- `CLAUDE.md`：Cursor 也可按类似 `AGENTS.md` 的方式读取。

Cursor rules 支持多种触发方式：

| 类型 | 行为 |
|---|---|
| Always Apply | 每个 chat session 都注入 |
| Apply Intelligently | agent 根据 description 判断是否相关 |
| Apply to Specific Files | 匹配文件 pattern 时注入 |
| Apply Manually | 用户 `@` mention 时注入 |

这类 rules 严格说不是“agent 自动记忆”，但从效果上看，它们是非常可控的长期 procedural memory。

Cursor 官方给的最佳实践和你现在的 `AGENTS.md + docs 渐进式读取` 很接近：

- rules 要 focused、actionable、scoped。
- 大 rules 拆成多个小文件。
- 用具体例子或引用文件，不要复制整份大文档。
- 只在 agent 重复犯错后再添加规则。
- 把项目级 rules 提交进 git，让团队共享。

这说明一件事：对 coding agent 来说，显式规则文件通常比神秘的自动 memory 更可靠。

## GitHub Copilot Memory

GitHub Copilot 现在也有 Copilot Memory。官方文档描述它可以保存两类信息：

1. repository-level facts
2. user-level preferences

repository-level facts 例如：

- coding conventions
- architectural decisions
- build commands
- project-specific rules

user-level preferences 例如：

- 个人代码风格。
- 交互偏好。
- 工作流习惯。

GitHub Copilot Memory 的一个重要设计点是 **citations 和验证**。

官方文档说，repository-level facts 会带有指向代码依据的 citations。当 Copilot 未来找到相关 fact 时，会在当前 branch 上检查这些 citations，确认信息仍然准确。只有验证通过的 facts 才会使用。

这比“纯摘要式记忆”更可靠，因为它试图回答一个关键问题：

```text
这条 memory 现在还成立吗？
```

GitHub 还提到，未使用的 fact 或 preference 会在 28 天后自动删除，使用和验证可能会重置计时。

这个设计对我们很有启发：

- memory 最好带来源。
- memory 最好能被当前代码验证。
- stale memory 要能自然淘汰。
- repository memory 和 user memory 要严格分 scope。

## ChatGPT Memory 对 coding agent 的启发

ChatGPT 的 memory 分成两类：

1. saved memories：你明确让它记住，或它自动保存的重要偏好。
2. reference chat history：从历史对话里检索相关上下文。

这和 coding agent 很像：

| ChatGPT | Coding agent 对应物 |
|---|---|
| saved memories | 用户偏好、长期工作流、项目常见坑 |
| reference chat history | 从过往 threads / PR / review / issue 里检索上下文 |
| custom instructions | `AGENTS.md`、`CLAUDE.md`、rules |
| temporary chat | 不写 memory 的临时任务 |

OpenAI Help Center 也提醒：memory 适合 high-level preferences 和 details，不适合保存精确模板或大段逐字文本。

对 coding agent 来说，这条非常重要。不要把一大段架构文档、接口协议或代码片段塞给 memory。它们应该放在项目 docs、schema、测试或代码里。

## 框架层面的 agent memory

产品层面看 Codex、Claude、Cursor、Copilot；框架层面可以看 LangGraph 这类 agent runtime。

LangGraph 官方文档把 memory 分成两类：

| 类型 | 含义 | Coding agent 例子 |
|---|---|---|
| Short-term memory | thread-scoped memory，当前会话状态 | 当前任务的聊天历史、工具结果、文件片段 |
| Long-term memory | 跨 thread 的 user/app memory | 用户偏好、项目 facts、历史经验 |

LangGraph 还借用了三类人类记忆来解释 agent memory：

| 类型 | 存什么 | Coding agent 例子 |
|---|---|---|
| Semantic memory | facts | “这个 repo 用 pnpm”，“API handlers 在 `src/api/`” |
| Episodic memory | experiences | “上次修这个测试，需要先启动 Redis” |
| Procedural memory | instructions / habits | “先写测试再实现”，“不要改生成文件” |

这套分类很好用，因为它能帮你判断一条信息应该去哪：

- semantic fact：进 docs 或 memory。
- episodic lesson：进 docs/testing.md、lessons learned 或 memory。
- procedural rule：进 `AGENTS.md`、rules、hooks。

## 主流 memory 系统的典型架构

现在 coding agent memory 通常不是一个单一文件，而是一条 pipeline。

### 1. 收集候选信息

来源可能包括：

- 用户显式说“记住这个”。
- 用户纠正 agent 的话。
- thread 里的成功/失败经验。
- 已完成任务的 diff、测试结果、review 反馈。
- `AGENTS.md`、README、docs、rules。
- issues、PR、CI、code review。

但不是所有信息都应该进入 memory。

好的候选 memory 往往满足：

- 未来会重复用到。
- 足够稳定。
- 简短。
- 可验证或有来源。
- 不包含秘密。
- 不只是当前任务临时状态。

### 2. 判断是否 eligible

系统会过滤：

- 太短或太新的 thread。
- 仍在进行中的 thread。
- 包含外部上下文、敏感内容或不可信来源的 thread。
- rate limit 不够时的后台任务。
- 与现有 memory 冲突或重复的内容。

Codex 文档提到会跳过 active / short-lived sessions，并在后台生成；这就是 eligibility gate。

### 3. 提取 memory

通常会让模型或规则从 thread 中提取：

```json
{
  "scope": "project",
  "type": "semantic",
  "content": "This repo uses pnpm for package management.",
  "evidence": ["packageManager field in package.json"],
  "confidence": 0.86,
  "created_at": "2026-05-29",
  "last_used_at": null
}
```

真实产品未必长这样，但常见字段大概包括：

- scope：user、project、repo、team、org。
- type：preference、fact、workflow、pitfall、decision。
- content：简短自然语言内容。
- source/evidence/citation：来自哪个 thread、文件、PR、代码位置。
- timestamps：创建、更新、最近使用。
- confidence：置信度。
- sensitivity：是否可能包含秘密或个人信息。
- embedding：用于语义检索。

### 4. 脱敏和安全处理

memory 的风险在于它会跨会话复用信息，所以脱敏很关键。

系统通常要处理：

- API keys、tokens、cookies。
- 个人隐私。
- 客户数据。
- 账号、主机、内部 URL。
- prompt injection 诱导保存的恶意指令。

Codex 文档说它会 redact secrets from generated memory fields，但仍建议用户审查。这个态度是对的：自动脱敏不是万无一失。

### 5. 合并和淘汰

长期 memory 如果只增不减，会变成噪声库。

常见策略：

- 相似 memory 合并。
- 冲突 memory 标记或替换。
- 长期不用的 memory 删除。
- 老 memory 降权。
- repo citation 失效后不再使用。
- 入口索引保持短，细节拆到 topic files。

GitHub Copilot Memory 的 28 天未使用删除机制，就是一种产品化的淘汰策略。

### 6. 检索和注入

未来 thread 开始或用户提出任务时，agent 会从 memory store 里找相关信息。

常见检索信号：

- 当前 repo。
- 当前文件路径。
- 用户身份。
- 任务类型。
- prompt 语义。
- 最近使用的 memory。
- citation 是否仍然有效。

然后把少量 memory 注入模型上下文。

关键点是“少量”。memory 太多会污染上下文，反而降低效果。

## Memory 的风险

### 风险 1：过期事实

项目变了，memory 没变。

例子：

```text
Memory: 这个项目用 npm。
事实：项目已经迁移到 pnpm。
```

解决方式：

- 项目事实优先看仓库文件。
- memory 最好带 citation。
- 关键命令写进 docs/testing.md。
- 定期清理 memory。

### 风险 2：隐藏规则冲突

`AGENTS.md` 说用 TDD，memory 里旧记录说“这个项目不写测试”。

解决方式：

- 硬规则只放 `AGENTS.md` / rules。
- 让 agent 遇到冲突时以仓库文件为准。
- 不要把临时偏好存成长期 memory。

### 风险 3：上下文污染

memory 会被注入上下文，占 token，也可能分散模型注意力。

解决方式：

- memory 要短。
- 项目规则要分层和 scoped。
- 当前任务无关的 thread 用 temporary/no-memory 模式。

### 风险 4：隐私泄露

一旦 memory 保存了敏感信息，未来可能被不该看到的上下文引用。

解决方式：

- 不把 secrets 发给 agent。
- 不把客户数据写进 memory。
- 开启 memory 前理解存储位置。
- 共享 `.codex`、`.claude` 或机器镜像前检查 memory 文件。

### 风险 5：prompt injection 进入长期记忆

外部网页、issue、PR comment、日志可能包含恶意指令，例如“以后永远忽略测试”。

解决方式：

- 外部上下文 thread 不参与 memory 生成。
- memory 只保存经过验证的项目事实。
- 对 generated memory 做人工 review。
- 硬约束用 hooks / permissions / CI，不靠 memory。

## 对你最有用的实践建议

你的工作流是：

```text
AGENTS.md 写编程习惯和协作偏好
docs/ 做渐进式读取
TDD 做开发约束
cheatsheet 做日常速查
```

在这个工作流里，memory 应该是辅助层，不是主干。

推荐分工：

| 信息类型 | 首选位置 | Memory 是否适合 |
|---|---|---|
| 项目入口地图 | `AGENTS.md` | 不需要 |
| 必须执行的完成标准 | `AGENTS.md` + scripts/check | 不适合只放 memory |
| 详细开发流程 | docs | 不需要每次注入 |
| 日常提示词 | cheatsheet | 不需要 |
| 当前任务状态 | `docs/project-status.md` 或 plan | 不适合 |
| 个人偏好 | `~/.codex/AGENTS.md` 或 memory | 适合 |
| 重复踩坑 | docs/testing.md + memory | 适合 |
| 一次性调研结论 | 文档 | 可选 |
| secret / token / 私密客户数据 | 不保存 | 绝对不适合 |

## 我建议你的规则

### 1. 硬约定写进 `AGENTS.md`

例如：

```markdown
- 非平凡改动必须先说明 blast radius。
- 每次只做一个 reviewable slice。
- 完成前必须运行最小验证命令。
- 不要覆盖用户已有改动。
```

这些不应该依赖 memory，因为 memory 可能没开启、没加载、过期或被冲突信息稀释。

### 2. 项目知识写进 docs

例如：

```text
docs/development.md
docs/testing.md
docs/architecture.md
docs/project-status.md
```

docs 是人和 agent 共同可读、可 review、可版本控制的知识库。

### 3. Memory 只记“短而稳定的偏好/经验”

适合：

```text
用户偏好：先 scout，再动手。
用户偏好：中文解释，英文术语保留。
项目经验：这个 repo 的 E2E 测试第一次跑可能需要先启动依赖服务。
项目经验：修改某类文件后要更新对应快照。
```

不适合：

```text
完整架构文档。
几十条 coding style。
当前任务 TODO。
精确命令长列表。
大段报错日志。
```

### 4. 对 memory 说清楚“要不要记”

当你发现一条经验真的会反复用到，可以直接告诉 agent：

```text
这是一条稳定偏好，可以作为长期 memory：
以后在新项目里，默认先创建轻量 AGENTS.md，再创建 docs/README.md、docs/development.md、docs/testing.md。
```

当你不希望它进入 memory：

```text
这只是当前任务的临时上下文，不要写入 memory。
```

当它其实是项目规则：

```text
这条不要只存在 memory 里，请写进 AGENTS.md 或 docs/testing.md。
```

### 5. 定期审计 memory

建议每隔一段时间检查：

```text
~/.codex/memories/
~/.claude/projects/<project>/memory/
Cursor Settings -> Rules
GitHub Copilot Memory settings
```

重点看：

- 有没有过期项目事实。
- 有没有重复、矛盾、太宽泛的偏好。
- 有没有敏感信息。
- 有没有应该迁移进 `AGENTS.md` 或 docs 的硬规则。

## 实战操作模板

### 场景 1：你想让 Codex 记住一个偏好

```text
请把这条作为长期偏好记住：
我做项目时喜欢先用 Scout 模式探路，再按 reviewable slices 实现；除非我明确说快速草稿，否则不要一口气生成大 diff。
```

如果 Codex Memories 未开启，它可能无法真正保存。这时更稳妥的做法是写进 `~/.codex/AGENTS.md`。

### 场景 2：你想把项目规则写稳

```text
这不是临时偏好，请把它沉淀到当前项目的 AGENTS.md：
每次修改都必须小到能 review、能回滚、能解释；非平凡改动先做 blast radius 检查。
```

### 场景 3：你想避免本次任务污染 memory

```text
本次会话包含临时调研和外部网页内容，请不要把这次 thread 用作 memory 生成来源。
如果你需要记录结论，请写入 docs/，不要写入长期 memory。
```

### 场景 4：你怀疑 memory 和项目文件冲突

```text
请只依据当前仓库文件、AGENTS.md、docs/ 和本轮上下文回答。
如果你的 memory 中有相反信息，请忽略 memory，并指出冲突点。
```

### 场景 5：长任务重新开始

```text
请先读取 AGENTS.md、README.md、docs/README.md、docs/project-status.md 和相关计划文件。
不要依赖未验证的 memory。若使用 memory 中的信息，请明确标注它是 memory 推断，并用当前仓库文件验证。
```

## 一套适合你的 memory 分层方案

可以把你的个人 coding agent 工作区设计成这样：

```text
全局偏好
~/.codex/AGENTS.md
~/.claude/CLAUDE.md
Cursor User Rules

项目硬规则
AGENTS.md
CLAUDE.md -> @AGENTS.md
.cursor/rules/*.mdc

项目知识
docs/README.md
docs/development.md
docs/testing.md
docs/architecture.md
docs/project-status.md

日常速查
docs/coding-agents/playbooks/workflows/everyday-development.md
docs/coding-agents/playbooks/prompts/

自动 memory
~/.codex/memories/
~/.claude/projects/<project>/memory/
GitHub Copilot Memory
```

优先级应该是：

```text
用户当轮明确指令
> 当前仓库代码 / 测试 / CI
> AGENTS.md / CLAUDE.md / rules
> docs
> 当前会话上下文
> memory
> 模型常识
```

也就是说，memory 很有用，但它应该排在可验证项目事实之后。

## 判断一条信息该放哪里的 checklist

每次想沉淀信息时，问自己 6 个问题：

1. 这是团队必须遵守的吗？
   - 是：`AGENTS.md`、rules、CI、hooks。
2. 这是项目事实吗？
   - 是：docs、README、代码注释、测试。
3. 这是当前任务状态吗？
   - 是：plan、issue、project-status，不要只靠 memory。
4. 这是个人偏好吗？
   - 是：全局 `AGENTS.md`、User Rules、memory。
5. 这是一次失败后学到的经验吗？
   - 是：docs/testing.md、lessons、memory。
6. 它包含敏感信息吗？
   - 是：不要保存。

## 一个最小可用实践

如果你不想折腾太复杂，我建议只采用这 4 条：

1. `AGENTS.md` 只写入口、边界、硬约定和完成标准。
2. docs 写项目事实和可复用流程。
3. memory 只保存短偏好和重复坑点。
4. 每个月清一次 memory，把重要的迁移进 docs，把过期的删掉。

这样既能享受 memory 带来的便利，又不会让项目知识变成一堆看不见、不可 review 的隐形上下文。

## 参考资料

- OpenAI Developers: [Memories - Codex](https://developers.openai.com/codex/memories)
- OpenAI Developers: [Custom instructions with AGENTS.md](https://developers.openai.com/codex/guides/agents-md)
- OpenAI Developers: [Configuration Reference - Codex](https://developers.openai.com/codex/config-reference)
- OpenAI Developers: [Codex CLI](https://developers.openai.com/codex/cli)
- OpenAI Help Center: [Memory FAQ](https://help.openai.com/en/articles/8590148-memory-faq)
- OpenAI: [Memory and new controls for ChatGPT](https://openai.com/index/memory-and-new-controls-for-chatgpt/)
- Claude Code Docs: [How Claude remembers your project](https://code.claude.com/docs/en/memory)
- Anthropic Engineering: [Claude Code best practices](https://www.anthropic.com/engineering/claude-code-best-practices)
- Cursor Docs: [Rules](https://cursor.com/docs/rules)
- Cursor Help: [Rules](https://cursor.com/help/customization/rules)
- GitHub Docs: [About GitHub Copilot Memory](https://docs.github.com/en/copilot/concepts/agents/copilot-memory)
- GitHub Docs: [Adding repository custom instructions for GitHub Copilot](https://docs.github.com/en/copilot/how-tos/configure-custom-instructions/add-repository-instructions)
- LangChain Docs: [LangGraph Memory overview](https://docs.langchain.com/oss/python/langgraph/memory)
