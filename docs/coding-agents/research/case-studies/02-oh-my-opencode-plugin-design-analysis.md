# Oh My OpenCode / Oh My OpenAgent 插件设计思想调研

调研日期：2026-05-29

## 这篇文档回答什么

这里的 **Oh My OpenCode** 指 `code-yeongyu/oh-my-opencode` 项目和 npm 包 `oh-my-opencode`。需要先说明一个命名变化：截至本次调研，GitHub 仓库和文档已经逐步迁移到 **Oh My OpenAgent** / `oh-my-openagent` 这个名字，但 npm 包 `oh-my-opencode`、CLI 命令 `oh-my-opencode` 仍然存在，并且与 `oh-my-openagent` 处于兼容过渡期。

本文回答：

1. Oh My OpenCode / Oh My OpenAgent 是什么？
2. 它在 OpenCode 生态里处于什么位置？
3. 它的核心能力和设计原理是什么？
4. 为什么它会被很多人使用？
5. 它对 harness engineering 有什么启发？
6. 使用它时有哪些边界和风险？

核心结论：

> Oh My OpenCode 的本质不是“一个更强的 coding prompt”，而是一个重型 agent harness 插件：它把 OpenCode 这个可扩展 coding agent 底座，包装成一个带多模型调度、多专家 agent、后台并行任务、LSP/AST 工具、Hashline 编辑、hooks、skills、MCP、上下文注入和恢复机制的完整工程系统。它流行的原因也不只是“效果好”，而是它把很多用户原本要自己配置、试错、维护的 agent harness 经验打包成了开箱即用的默认方案。

## 来源说明

本次调研主要使用以下来源：

| 来源 | 类型 | 用途 |
|---|---|---|
| `code-yeongyu/oh-my-opencode` / `code-yeongyu/oh-my-openagent` GitHub 仓库 | 上游项目 | README、功能文档、安装文档、源码目录结构 |
| npm 包 `oh-my-opencode` | 包元数据 | 版本、包描述、依赖、CLI 命令、workspace 模块 |
| GitHub API | 项目采用信号 | stars、forks、仓库迁移信息 |
| npm downloads API | 包下载信号 | 最近一周和最近一月下载量 |
| OpenCode 官方插件文档 | 上游生态背景 | 确认 OpenCode 插件机制的设计位置 |
| 项目隐私政策和服务条款 | 风险边界 | 遥测、第三方服务、本地运行边界 |

截至 2026-05-29，公开数据大致是：

| 指标 | 数值 |
|---|---:|
| GitHub stars | 约 60,051 |
| GitHub forks | 约 4,882 |
| npm 最近一周下载量 | 77,500，统计区间为 2026-05-21 至 2026-05-27 |
| npm 最近一月下载量 | 318,112，统计区间为 2026-04-28 至 2026-05-27 |
| npm 最新版本 | `4.5.1` |
| npm 创建时间 | 2025-12-04 |
| npm 最近更新时间 | 2026-05-26 |
| npm 已发布版本数 | 219 |

这些数据只能说明“采用和关注度很高”，不能直接证明它在所有项目里效果一定更好。agent harness 的质量仍然需要结合任务类型、模型可用性、代码库规模、成本和验证机制来判断。

## 它是什么

一句话说：

```text
Oh My OpenCode / Oh My OpenAgent = 面向 OpenCode 的 batteries-included agent harness 插件。
```

npm 包自己的描述是：

```text
The Best AI Agent Harness - Batteries-Included OpenCode Plugin with Multi-Model Orchestration, Parallel Background Agents, and Crafted LSP/AST Tools
```

拆开看，它不是单点工具，而是一套组合系统：

- 它依赖 OpenCode 的插件机制，把 OpenCode 从一个可扩展 coding agent 变成一个更有主见的工作系统。
- 它内置多个 agent 角色，例如 Sisyphus、Hephaestus、Prometheus、Atlas、Oracle、Librarian、Explore、Momus、Metis 等。
- 它允许不同 agent 或任务 category 自动映射到不同模型，而不是要求用户手动来回切换模型。
- 它支持后台 agent 和 Team Mode，把单个 agent 的顺序工作变成多 agent 并行协作。
- 它把 LSP、AST-Grep、MCP、tmux、skills、hooks、AGENTS.md 注入、规则注入、会话恢复等能力整合到一个插件里。
- 它通过安装器、doctor、配置 schema、平台二进制和兼容层，降低用户自己拼装这套系统的成本。

用这个项目自己的类比说，如果 OpenCode 像底层 Debian / Arch，那么 Oh My OpenAgent 更像一个开箱即用、带大量默认配置和工具链的发行版。

## 名称演进：从 Oh My OpenCode 到 Oh My OpenAgent

这个项目有一个容易让人混乱的地方：用户口中的 `oh-my-opencode` 和上游现在文档里的 `oh-my-openagent` 基本指向同一条演进线。

调研时看到的事实是：

- GitHub 旧仓库 `code-yeongyu/oh-my-opencode` 已重定向到 `code-yeongyu/oh-my-openagent`。
- npm 包名仍然是 `oh-my-opencode`，最新元数据里的 repository 和 homepage 指向 `oh-my-openagent`。
- npm 包的 `bin` 同时提供 `oh-my-opencode` 和 `oh-my-openagent` 两个命令。
- 安装文档说项目在重命名过渡期内双名兼容。
- 配置文件兼容 `oh-my-openagent.json[c]` 和旧的 `oh-my-opencode.json[c]`。

所以在阅读资料时可以这样理解：

```text
旧品牌 / npm 主包名：oh-my-opencode
新品牌 / 当前仓库名：oh-my-openagent
本文语境：二者按同一个项目演进线分析
```

这次命名变化也反映了项目定位的变化：它不再只是 “OpenCode 的增强包”，而是想抽象成一个更通用的 multi-harness agent operating system，未来可能支持 OpenCode、Codex、Pi 等多个 agent harness。

## 它在 OpenCode 生态里的位置

OpenCode 本身是一个开源 AI coding agent，提供终端、IDE、desktop、配置、模型、agents、rules、commands、formatters、permissions、skills、plugins 等扩展点。

OpenCode 官方插件文档对插件的定位是：插件可以通过事件 hooks 扩展行为、集成外部服务、修改默认行为，插件可以放在项目级 `.opencode/plugins/`，也可以放在全局 `~/.config/opencode/plugins/`，还可以通过 npm 包发布并在配置里引用。

Oh My OpenCode 就是利用这个扩展点做了一件更激进的事：它不是加一个小功能，而是把插件层变成了一个完整 harness 层。

可以把三层关系理解成：

```text
模型层
  Claude / GPT / Gemini / Kimi / GLM / MiniMax / Grok Code 等

Agent runtime 层
  OpenCode：会话、工具调用、模型接入、配置、插件、TUI/CLI

Harness 层
  Oh My OpenCode：agent 编排、模型路由、工具增强、hooks、skills、MCP、恢复、验证
```

它真正卖点不在于“替代 OpenCode”，而是把 OpenCode 暴露出来的开放性转化为一套带默认答案的工作流。

## 核心能力总览

### 1. 多 agent 角色

项目文档列出 11 个内置 agent，每个 agent 有不同职责、模型偏好和权限边界。

| Agent | 定位 |
|---|---|
| `Sisyphus` | 默认主调度器，负责计划、委托、执行复杂任务 |
| `Hephaestus` | GPT-native 深度自主执行者，适合复杂技术问题 |
| `Prometheus` | 访谈式战略规划师，先澄清需求再写计划 |
| `Atlas` | Todo / plan 执行调度器，按计划分派和验收任务 |
| `Oracle` | 架构、调试、代码审查顾问，通常只读 |
| `Librarian` | 文档、开源实现、多仓库知识检索 |
| `Explore` | 快速代码库探索和 contextual grep |
| `Multimodal-Looker` | 图片、PDF、图表等视觉内容分析 |
| `Metis` | 计划前 gap analyzer，找隐藏歧义和失败点 |
| `Momus` | 计划 reviewer，检查清晰度、可验证性、完整性 |
| `Sisyphus-Junior` | category-spawned executor，根据任务类别选择模型 |

这套角色设计的重点是：不要让一个 agent 做所有事。

单 agent 的典型问题是：

- 上下文被搜索、实现、调试、解释混在一起撑爆。
- 同一个模型既要当架构师，又要当前端，又要当测试工程师。
- agent 一旦卡住，会在同一条轨道里反复尝试。
- 很多任务天然可以并行，但单 agent 只能顺序做。

Oh My OpenCode 的解法是把工作拆成角色，让主 agent 更像 team lead：主 agent 负责判断和委托，其他 agent 负责检索、规划、审查、实现或视觉分析。

### 2. 多模型调度

它的一个核心观点是：模型不是简单的强弱关系，而是工作风格不同。

项目的 `Agent-Model Matching Guide` 里把模型当成团队里的不同开发者：

- Claude / Kimi / GLM 更适合长流程、细规则、复杂指令跟随和协调型 agent。
- GPT 更适合原则驱动、深度自主探索、复杂推理和独立解决问题。
- Gemini 更适合视觉、前端、创意和多模态场景。
- 快速便宜模型适合检索、grep、简单修改和工具型任务。

因此，Oh My OpenCode 的调度单位不只是“选择模型”，而是：

```text
任务意图 -> agent 或 category -> 模型族 / 具体模型 -> fallback chain
```

例如：

- `visual-engineering` 可以默认路由到 Gemini 系列。
- `ultrabrain` 可以路由到 GPT high / xhigh reasoning。
- `quick` 可以路由到更快更便宜的小模型。
- `Sisyphus` 默认更偏 Claude-like 模型。
- `Hephaestus` 更偏 GPT-native 模型。

这对用户的价值是：用户不需要每次问“这个任务该用 Claude 还是 GPT 还是 Gemini”。harness 把这个判断编码成 category、agent 和 fallback 规则。

### 3. Category system：按任务类型委托，而不是按模型名委托

Oh My OpenCode 引入 category 作为 agent 配置预设。它不是问：

```text
用哪个模型？
```

而是问：

```text
这是什么类型的工作？
```

内置 category 包括：

| Category | 典型用途 |
|---|---|
| `visual-engineering` | 前端、UI/UX、设计、动画 |
| `ultrabrain` | 高难逻辑、架构决策、深度推理 |
| `deep` | 复杂问题的自主研究和执行 |
| `artistry` | 创意、视觉、表达类任务 |
| `quick` | 单文件修改、typo、简单修复 |
| `unspecified-low` | 未明确分类但低复杂度任务 |
| `unspecified-high` | 未明确分类但高复杂度任务 |
| `writing` | 文档、技术写作、说明文字 |

这个设计很重要。人类在委托任务时也不会说“请用某个大脑参数去做这个事”，而是说“找一个擅长前端的人”“找一个能做架构 review 的人”“找一个快速扫代码的人”。

Category system 把模型选择从“手动配置问题”变成“职责匹配问题”。

### 4. 后台 agent 和 Team Mode

普通 coding agent 最大的瓶颈之一是单线程。它要先搜索，再读文档，再实现，再测试，再 review。人类团队不会这样工作，多个工程师会并行推进。

Oh My OpenCode 支持两类并行：

1. **Background Agents**：让某个 agent 在后台做调研、调试、检索或实现，主 agent 继续工作，完成后再读取结果。
2. **Team Mode**：一个 lead agent 协调最多 8 个成员，成员通过 `team_*` 工具、共享 mailbox、共享 task list 和可选 tmux pane 协作。

项目文档里 Team Mode 默认关闭，需要配置启用。启用后提供 12 个 `team_*` 工具，例如：

- `team_create`
- `team_send_message`
- `team_task_create`
- `team_task_list`
- `team_task_update`
- `team_status`
- `team_delete`

这体现出一个核心思路：

```text
agent harness 不只是给模型更多上下文，而是给模型一个可协作的执行组织。
```

不过并行不是免费的。它会增加 token 成本、状态管理复杂度、冲突概率和验证负担。因此 Team Mode 默认关闭是合理的：只有在长任务、多模块改造、并行调研和安全审计这类场景里，才值得打开。

### 5. LSP + AST-Grep：让 agent 拥有 IDE 级代码操作能力

很多 coding agent 的失败不是模型“不知道”，而是工具太粗糙。

如果 agent 只能靠文本 grep 和全文替换，它很容易：

- 改错同名变量。
- 漏掉跨文件引用。
- 误判符号定义。
- 在重构时破坏调用点。
- 对语言结构没有稳定理解。

Oh My OpenCode 把 LSP 和 AST-Grep 做成内置工具：

| 工具类型 | 能力 |
|---|---|
| LSP | diagnostics、rename、goto definition、find references、symbols |
| AST-Grep | AST-aware search、AST-aware replace，支持多语言语法模式 |

这背后的 harness 思想是：

```text
模型负责判断和意图，确定性工具负责定位和执行。
```

这是 coding agent 很关键的工程分界。模型适合回答“应该怎么改”，但真正跨文件 rename、找引用、语法结构替换，应该尽量交给语言服务和 AST 工具。

### 6. Hashline：解决“改错行”的 harness 问题

项目 README 把 Hashline 作为一个重点能力：agent 读取文件时，每行带上类似 `LINE#ID` 的内容哈希标识；当 agent 发起编辑时，必须引用这些稳定标识；如果文件内容在期间变化，哈希验证失败，编辑会被拒绝。

这解决的是一个很实际的问题：

```text
传统 edit 工具依赖模型复写原文片段。
模型一旦少一个空格、漏一行、碰到相似代码，就可能替换错位置。
```

Hashline 的思想是：给 agent 一个稳定、可验证的“锚点”，让编辑操作具备并发安全和内容验证。

它和普通 patch/edit 的区别在于：

- 普通 edit 更依赖模型准确复述上下文。
- Hashline edit 依赖行内容哈希来验证目标仍然是它刚才读到的内容。
- 如果文件已变化，编辑不应悄悄继续，而应显式失败。

这正是 harness engineering 的典型做法：不要相信模型“应该没看错”，而是让工具层验证模型引用的对象是否仍然成立。

### 7. Hooks：把经验变成生命周期控制

Oh My OpenCode 的 hook 系统很重。项目文档显示基础 hook 约 54 个，启用 Team Mode 后约 61 个，覆盖 session、tool guard、transform、continuation、skill、event、params 等层面。

这些 hook 大致可以分成几类：

| 类别 | 例子 | 目的 |
|---|---|---|
| 上下文注入 | AGENTS.md、README、rules 注入 | 让 agent 在读文件时获得局部规范 |
| 模式激活 | `ultrawork`、`ulw`、think mode | 从用户 prompt 触发特定工作模式 |
| 质量控制 | comment checker、write existing file guard | 减少 AI 味注释和误覆盖 |
| 恢复机制 | session recovery、context window recovery、json error recovery | 从常见运行失败中恢复 |
| 模型回退 | model fallback、runtime fallback | provider 或模型失败时切换备用模型 |
| 持续执行 | ralph loop、todo continuation enforcer | 防止 agent 半途停止 |
| 集成 | Claude Code hooks、interactive bash session | 兼容外部工作流和交互式终端 |

从设计角度看，hooks 的作用是把“老用户踩坑后的经验”固化到生命周期里。

例如：

- agent 想没读文件就覆盖已有文件：hook 拦截。
- agent 生成大量废话注释：comment checker 拦截。
- agent 接近上下文窗口：提前 compaction 或恢复。
- provider 报错：进入 fallback。
- agent 没做完就停：continuation enforcer 拉回来。

这不是 prompt 层能稳定解决的问题。prompt 会被遗忘、被压缩、被模型风格影响；hook 是执行层的约束。

### 8. Skills + skill-embedded MCP

Oh My OpenCode 也有 skills 系统。它把 skill 定义为“特殊知识 + 工具 + 工作流”的组合，而不是简单 prompt 模板。

项目文档提到的内置 skills 包括：

- `git-master`：提交、rebase、历史考古。
- `playwright`：浏览器自动化、测试、截图。
- `agent-browser` / `dev-browser`：浏览器任务。
- `frontend-ui-ux`：UI/UX 实现。
- `review-work`：实现后并行 review。
- `ai-slop-remover`：移除 AI 生成痕迹。

更值得注意的是 **skill-embedded MCP**：某个 skill 可以带自己的 MCP server，并且只在需要时启动，任务完成后销毁。

这个设计解决一个很常见的问题：

```text
全局挂太多 MCP 会撑爆上下文，也会增加工具选择噪音。
```

按需加载 MCP 更符合 harness engineering 的原则：

- 让工具只在相关任务里出现。
- 降低上下文污染。
- 降低模型工具选择难度。
- 让 skill 从“提示词”升级成“领域工作包”。

### 9. Claude Code 兼容层

项目强调兼容 Claude Code 的 hooks、commands、skills、agents、MCP 和插件配置。这一点是它流行的重要原因之一。

很多开发者已经在 Claude Code 生态里积累了：

- 自定义 commands。
- hooks。
- skills。
- agents。
- MCP servers。
- AGENTS.md 或项目规则。

如果切到 OpenCode 后这些资产都要重写，迁移成本会很高。Oh My OpenCode 通过兼容层降低迁移成本，让用户可以把已有 Claude Code 工作流带到 OpenCode 生态里。

这也是一个很典型的 adoption 设计：

```text
不是要求用户抛弃旧工作流，而是吸收旧工作流。
```

### 10. CLI、安装器、doctor 和 schema

Oh My OpenCode 不只是一个插件入口，还提供 CLI：

- `install`
- `run`
- `doctor`
- `mcp-oauth`
- `refresh-model-capabilities`
- `get-local-version`

安装文档推荐用 `bunx oh-my-openagent install`，并且包提供多个平台的 standalone binaries。配置支持 JSONC 和 schema URL，用于补全和校验。

`doctor` 的意义很大。复杂 harness 最怕“装是装好了，但不知道哪里坏了”。doctor 可以检查插件注册、配置、模型解析、环境、Team Mode 等问题。

这说明作者意识到：agent harness 的难点不只是功能，而是可安装、可诊断、可维护。

## 设计原理拆解

### 原理一：从 prompt engineering 走向 harness engineering

Oh My OpenCode 的核心不是写一个超长万能 prompt。虽然它确实有大量 prompt 和 agent persona，但更重要的是它把 prompt 放进一个执行系统里。

它试图控制的是完整链路：

```text
用户意图
-> IntentGate / keyword detector
-> 主 agent
-> planning / delegation
-> category / model routing
-> specialized tools
-> hooks guard
-> background / team execution
-> diagnostics / tests / review
-> recovery / fallback
```

这和单纯 prompt engineering 的区别是：

- prompt engineering 主要回答“怎么说服模型按我想的做”。
- harness engineering 主要回答“模型没按我想的做时，系统如何约束、检测、恢复和验证”。

Oh My OpenCode 的价值正在后者。

### 原理二：把模型当“团队成员”，不是当“单个神谕”

它的多模型设计隐含一个判断：

```text
未来不会是一个模型统治所有场景，而是多个模型在不同任务中组合使用。
```

因此它不是问“哪个模型最强”，而是问：

- 哪个模型适合长流程协调？
- 哪个模型适合深度代码推理？
- 哪个模型适合视觉和前端？
- 哪个模型适合便宜快速搜索？
- 哪个模型作为 fallback 最稳？

这比 benchmark 排名更贴近实际 coding agent 使用。因为一个 agent 系统最终的瓶颈往往不是单题能力，而是：

- 长任务能否持续。
- 多文件能否稳定。
- 失败能否恢复。
- 成本能否接受。
- 工具调用是否合适。
- 上下文是否被污染。

### 原理三：主 agent 不应亲自做所有低价值探索

在复杂代码库里，主 agent 如果亲自 grep、读文档、搜 API、分析无关文件，很快就会耗尽上下文和注意力。

Oh My OpenCode 把 Explore、Librarian、Oracle、background agents 设计成主 agent 的外部工作记忆和外部专家。

这种设计的好处是：

- 主 agent 上下文更干净。
- 搜索和调研可以并行。
- 便宜模型可以承担低价值任务。
- 高价值模型把 token 花在决策和集成上。
- 不同 agent 可以带不同权限，降低误操作风险。

这和人类团队也类似：技术负责人不应该把所有时间花在 grep 文件上，而应该让不同成员调研后汇总。

### 原理四：把“继续做完”变成系统行为

项目里有 `ultrawork`、`ulw`、Ralph Loop、todo continuation enforcer、Atlas、task system、boulder state 等一系列围绕“不要半途停”的机制。

这背后是对 LLM agent 常见失败模式的判断：

```text
agent 经常不是完全不会做，而是做到 60% 就停、遇到阻力就换方向、或者把未验证的状态说成完成。
```

所以 Oh My OpenCode 的很多机制都在对抗“半成品”：

- `ultrawork` 触发更激进的自动执行。
- Todo continuation enforcer 让 agent 回到未完成事项。
- Ralph Loop / `/ulw-loop` 做自我继续。
- Atlas 按计划持续分派和验收。
- session recovery 和 context recovery 减少中断。

这类机制很有争议，因为它可能增加 token 成本，也可能让 agent 更难停下。但它确实抓住了 coding agent 的一个核心痛点：完成度比单次回答质量更重要。

### 原理五：确定性工具优先，模型只做模型擅长的事

LSP、AST-Grep、Hashline、diagnostics、doctor、schema、配置解析、fallback chain，这些都体现同一个原则：

```text
能用确定性工具解决的问题，不要交给模型猜。
```

模型擅长：

- 读意图。
- 取舍方案。
- 解释上下文。
- 生成候选实现。
- 做跨领域综合。

工具擅长：

- 找符号引用。
- 验证重命名是否合法。
- 匹配 AST 模式。
- 检查文件是否被改过。
- 校验配置。
- 跑测试和诊断。

好的 harness 应该让模型和工具各做擅长的事，而不是让模型用自然语言模拟 IDE。

### 原理六：开箱即用比“完全自由”更容易形成采用

OpenCode 的优势是开放、可配置、可扩展。但开放也意味着用户要做很多选择：

- 选哪些模型？
- 怎么配置 agents？
- 哪些 hooks 要开？
- 要不要加 MCP？
- 如何处理 Claude Code 的迁移？
- 怎样避免 context 爆炸？
- 哪个工具适合前端？
- 失败时怎么恢复？

Oh My OpenCode 的流行，很大程度上来自它给出了 opinionated defaults。

它不是说“你可以配置一切”，而是说：

```text
我已经试过很多组合，把能跑的默认值给你装好。
```

这非常符合工具采用规律。高级用户喜欢自由，但大多数用户首先需要一套能工作的默认答案。

## 为什么很多人用

### 1. 它站在 OpenCode 快速增长的生态上

OpenCode 本身是开源 coding agent，提供多模型、多端、插件和配置能力。用户选择 OpenCode 往往是因为：

- 不想被单一供应商锁定。
- 想使用 Claude、GPT、Gemini、Kimi、GLM 等多种模型。
- 想要开源、可扩展、可配置的 agent runtime。
- 想在终端和本地环境里工作。

Oh My OpenCode 正好补上 OpenCode 的另一面：OpenCode 很开放，但用户需要自己搭 harness。Oh My OpenCode 把这些搭建成本打包了。

### 2. 它降低了多模型使用门槛

很多人知道“不同模型适合不同任务”，但真正使用时会遇到麻烦：

- 不知道哪个任务该用哪个模型。
- 不知道不同 provider 的模型名。
- 不知道如何设置 fallback。
- 不想每次手动切换。
- 不知道哪个模型便宜但够用。

Oh My OpenCode 把这些经验做成 agent、category 和 fallback chain。用户只需要说任务，系统决定用什么角色和模型。

这比“模型列表越多越好”更实用。模型多本身会制造选择负担，调度层才把模型多变成优势。

### 3. 它把 Claude Code 用户资产带到 OpenCode

Claude Code 用户已经习惯了 commands、hooks、skills、MCP、AGENTS.md 等工作流。Oh My OpenCode 提供兼容层，让用户迁移到 OpenCode 时不用从零搭建。

这会吸引两类用户：

- 喜欢 Claude Code 体验，但想要更开放模型选择的人。
- 已经投资 Claude Code 配置，希望复用资产的人。

兼容层让它不只是“新工具”，而像一个迁移桥。

### 4. 它解决了真实痛点，而不是只展示 demo

项目里的很多功能都对应 coding agent 的真实失败模式：

| 真实痛点 | 对应机制 |
|---|---|
| agent 半途停 | todo continuation、Ralph Loop、ultrawork |
| 改错行 | Hashline edit |
| 跨文件重构不稳 | LSP rename / references |
| 文本搜索噪音大 | AST-Grep、Explore |
| 文档和开源实现过时 | Librarian、websearch、Context7、grep.app |
| provider 报错 | model fallback、runtime fallback |
| 上下文撑爆 | compaction、skill-embedded MCP、background agents |
| AI 味注释太多 | comment checker |
| 配置复杂 | installer、doctor、schema |
| 长任务需要多人并行 | background agents、Team Mode |

这类功能不一定每个都完美，但它们对准的是用户每天真的遇到的问题。

### 5. 它有强烈的产品叙事

Oh My OpenCode 的 README 很会讲故事：

- `ultrawork` / `ulw` 作为一个简单入口词。
- Sisyphus、Prometheus、Atlas、Oracle 等神话化角色。
- “agent 像团队”而不是“一个聊天机器人”。
- “OpenCode 像底座，Oh My OpenAgent 像发行版”。
- “不要让人 babysit agent”。

这些叙事并不等于技术质量，但对传播很重要。复杂系统如果没有清晰入口，用户很难愿意试。`ulw` 这种入口词把复杂 harness 包装成一个可以记住的动作。

### 6. 它的发布节奏和采用信号很强

公开数据说明它不是一个安静的小实验：

- 从 2025-12-04 创建 npm 包，到 2026-05-26 最新版本 `4.5.1`，已经发布 219 个版本。
- 最近一周 npm 下载约 77,500，统计区间为 2026-05-21 至 2026-05-27。
- 最近一月 npm 下载约 318,112，统计区间为 2026-04-28 至 2026-05-27。
- GitHub stars 约 60,051，forks 约 4,882。
- 仓库最近更新和 push 都很活跃。

高频发布有两面性：

- 正面：说明作者持续迭代，快速追模型、工具和 OpenCode 生态变化。
- 负面：说明接口和行为可能变化很快，生产环境要注意版本锁定和升级验证。

### 7. 它满足“我不想再自己拼 agent harness”的需求

很多 agent 重度用户最后都会走向同一个问题：

```text
我不是缺一个模型，我缺一套能长期工作的 agent 操作系统。
```

自己拼这套系统需要很多时间：

- 比较模型。
- 写 agents。
- 写 prompts。
- 接 MCP。
- 调 hooks。
- 写 commands。
- 写验证流程。
- 处理上下文。
- 处理失败恢复。
- 调整权限。

Oh My OpenCode 的吸引力就是：它把这套繁琐的经验蒸馏成一个包。

## 和 Superpowers 的区别

前一篇文档分析的 Superpowers 更像：

```text
工作纪律库 / workflow gates / software development methodology
```

Oh My OpenCode 更像：

```text
运行时增强插件 / agent orchestration harness / batteries-included OpenCode distribution
```

二者都属于 harness engineering，但层次不同：

| 维度 | Superpowers | Oh My OpenCode |
|---|---|---|
| 核心对象 | skills / 工作流纪律 | OpenCode plugin / runtime harness |
| 重点 | 让 agent 按工程流程做事 | 给 agent 更多角色、工具、调度和恢复能力 |
| 典型能力 | brainstorming、TDD、debugging、verification、review | multi-agent、multi-model、LSP、AST、hooks、MCP、Hashline |
| 约束方式 | skill 规则和流程 gates | hooks、tools、agents、config、runtime behavior |
| 适配范围 | 多个 coding harness | 主要围绕 OpenCode，正在向 OpenAgent 演进 |
| 使用体验 | 调用对应 skill 进入流程 | 安装后获得完整增强系统 |

可以把它们组合理解：

```text
Superpowers 规定 agent 应该如何工作。
Oh My OpenCode 给 agent 提供一个更强的工作环境。
```

一个偏“流程纪律”，一个偏“运行时能力”。真正成熟的 agent harness 往往两者都需要。

## 对 harness engineering 的启发

### 启发一：评估 agent 不应只评估模型

同一个模型，在不同 harness 下表现会差很多。

差异来自：

- 工具有多精确。
- 上下文怎么注入。
- 是否有自动恢复。
- 是否有验证机制。
- 是否能并行调研。
- 是否能选择合适模型。
- 是否能拦截危险操作。

所以比较 Claude、GPT、Gemini 时，如果忽略 harness，就会误判模型能力。很多“模型不行”的问题，其实是工具、上下文、编辑器或执行环境不行。

### 启发二：多模型不是噱头，而是系统设计问题

多模型本身没价值，真正有价值的是：

```text
任务分类 + 模型匹配 + fallback + 成本控制 + 结果汇总
```

如果没有 harness，多模型只会让用户手动切换得更累。Oh My OpenCode 的 category system 说明，模型编排应该隐藏在任务意图后面。

### 启发三：agent 的工具应该越来越像 IDE 和 CI

早期 agent 工具是 `read`、`grep`、`edit`、`bash`。但成熟 coding agent 需要更专业的工程工具：

- LSP rename / references / diagnostics。
- AST-level search / rewrite。
- Hash-anchored edit。
- Test runner 和 formatter。
- Browser automation。
- Static analysis。
- Session search。
- Task system。

这意味着 coding agent 的能力上限，很大程度取决于它能否接入工程系统里已有的确定性工具。

### 启发四：完成度需要系统约束

人类评价 agent 时，常常更关心：

- 它有没有真的完成？
- 有没有跑测试？
- 有没有处理边界情况？
- 有没有把半成品说成完成？

Oh My OpenCode 的 continuation、todo、review、doctor、diagnostics、fallback 都在强化完成度。

这说明好的 harness 要把“完成”定义成一个可验证状态，而不是一句自然语言声明。

### 启发五：默认值是产品能力的一部分

很多工具理论上都能配出来，但用户不会配，或者不想每天调。Oh My OpenCode 把大量默认配置打包出来，本身就是产品能力。

对 agent harness 项目来说，重要的不只是：

```text
能不能做到
```

还包括：

```text
用户能不能在 10 分钟内做到
出问题时能不能诊断
升级时会不会崩
默认组合是不是足够好
```

## 使用边界和风险

### 1. 它不是 OpenCode 官方核心功能

它依赖 OpenCode 插件机制，但不是 OpenCode 核心。OpenCode、模型 provider、插件自身任一方变更，都可能带来兼容性问题。

对于重要项目，建议：

- 锁定版本。
- 阅读 changelog。
- 先在非关键仓库试用。
- 升级后跑 doctor 和项目验证。

### 2. 高频迭代意味着不稳定可能更高

219 个 npm 版本说明项目非常活跃，但也意味着行为变化可能很快。对个人探索这是优点，对团队生产工作流则要谨慎。

团队使用时最好建立：

- 固定版本。
- 固定配置模板。
- 升级窗口。
- 回滚方案。
- 基准任务集。

### 3. 多 agent 并行会放大成本和状态复杂度

Background Agents 和 Team Mode 很强，但也会带来：

- 更多 token 消耗。
- 更多 provider 调用。
- 更多上下文合并问题。
- 更多并发编辑冲突。
- 更多需要验证的输出。

并行适合高价值复杂任务，不适合所有小改动。

### 4. 强 continuation 可能让 agent 过度执行

`ultrawork`、Ralph Loop、todo continuation enforcer 这类机制的目标是“直到做完”。但如果需求本身不清楚，或任务边界不对，强继续可能导致：

- 扩大范围。
- 过度实现。
- 花费过多 token。
- 做出用户没要求的改动。

因此，对高风险任务，最好先用规划模式或明确范围，而不是直接 `ulw`。

### 5. LSP / AST 工具依赖项目环境

LSP 和 AST-Grep 能显著提高精度，但前提是项目语言、依赖、配置、workspace 能被正确识别。

如果项目本身构建失败、依赖未安装、语言服务配置错误，LSP 结果也会不可靠。

### 6. 遥测默认开启，需要按场景评估

项目隐私政策说明：匿名遥测默认开启，用于估计 DAU / WAU / MAU；事件包含包版本、插件名、runtime、OS family、locale、timezone、经过单向哈希的安装标识符等；政策称不会通过该遥测路径收集 prompt、源文件、仓库内容、access tokens、API keys、原始 hostname 或 runtime error diagnostics。

它提供两个关闭环境变量：

```bash
export OMO_SEND_ANONYMOUS_TELEMETRY=0
export OMO_DISABLE_POSTHOG=1
```

在公司、客户代码、受监管环境或隐私敏感仓库中，应默认先评估遥测和第三方服务条款。

### 7. 许可证和条款需要单独确认

npm 包显示 license 为 `SUL-1.0`，GitHub API 返回 license 类型为 `Other` / `NOASSERTION`。这不是最常见的 MIT / Apache-2.0 / BSD 类型。

如果用于公司内部或商业产品链路，应让法务或项目负责人确认许可证和 Terms of Service。

## 适合什么场景

比较适合：

- 个人或小团队想深度使用 OpenCode。
- 已经有多模型订阅，想让模型自动分工。
- 经常做复杂重构、多文件迁移、长任务开发。
- 想把 Claude Code 配置迁移到 OpenCode。
- 想实验 multi-agent / background agent / team mode。
- 愿意接受较活跃项目带来的变化。

不太适合：

- 只做偶尔的一行小改。
- 对工具链可预测性要求极高、不能接受快速变化。
- 公司环境禁止默认遥测或第三方插件。
- 不能联网或不能安装外部包。
- 不愿意调试模型 provider、OpenCode、插件三方交互问题。

## 如果要评估它，应该怎么测

不要只看 README demo。可以用以下方式评估：

1. 准备 5-10 个真实任务

包括：

- 单文件小修。
- 多文件重构。
- 需要查文档的新 API 接入。
- 前端 UI 修改。
- bug 复现与修复。
- 需要测试和验证的功能。
- 长上下文代码库探索。

2. 比较三组 baseline

```text
OpenCode 原生配置
OpenCode + 自己少量配置
OpenCode + Oh My OpenCode
```

3. 记录指标

| 指标 | 为什么重要 |
|---|---|
| 完成率 | 最核心 |
| 首次可用结果时间 | 是否真的提高效率 |
| 人类干预次数 | 是否减少 babysitting |
| 误改文件次数 | 工具安全性 |
| 测试通过率 | 完成声明是否可信 |
| token / 成本 | 并行是否值得 |
| 上下文污染 | 多 agent 输出是否可管理 |
| 回滚难度 | 出错后是否容易恢复 |
| 配置维护成本 | 长期使用是否可持续 |

4. 对比不同模式

- 普通 prompt。
- `ultrawork` / `ulw`。
- Prometheus planning -> `/start-work`。
- background agents。
- Team Mode。

5. 检查风险项

- 是否默认开启遥测。
- 是否产生不必要注释。
- 是否修改未授权文件。
- 是否调用了意料之外的 provider。
- 是否在失败后自动恢复到可接受状态。

## 参考资料

- GitHub 仓库：<https://github.com/code-yeongyu/oh-my-openagent>
- 旧仓库重定向：<https://github.com/code-yeongyu/oh-my-opencode>
- npm 包：<https://www.npmjs.com/package/oh-my-opencode>
- npm registry 元数据：<https://registry.npmjs.org/oh-my-opencode>
- npm downloads API：<https://api.npmjs.org/downloads/point/last-month/oh-my-opencode>
- OpenCode 插件文档：<https://opencode.ai/docs/plugins>
- OpenCode 官方站点：<https://opencode.ai>
- Oh My OpenAgent Overview：<https://github.com/code-yeongyu/oh-my-openagent/blob/dev/docs/guide/overview.md>
- Oh My OpenAgent Orchestration Guide：<https://github.com/code-yeongyu/oh-my-openagent/blob/dev/docs/guide/orchestration.md>
- Oh My OpenAgent Team Mode Guide：<https://github.com/code-yeongyu/oh-my-openagent/blob/dev/docs/guide/team-mode.md>
- Oh My OpenAgent Features Reference：<https://github.com/code-yeongyu/oh-my-openagent/blob/dev/docs/reference/features.md>
- Oh My OpenAgent Configuration Reference：<https://github.com/code-yeongyu/oh-my-openagent/blob/dev/docs/reference/configuration.md>
- Oh My OpenAgent Privacy Policy：<https://github.com/code-yeongyu/oh-my-openagent/blob/dev/docs/legal/privacy-policy.md>
- Oh My OpenAgent Terms of Service：<https://github.com/code-yeongyu/oh-my-openagent/blob/dev/docs/legal/terms-of-service.md>
