# Agent Skills 系统调研：从重复 Prompt 到可治理能力包

调研日期：2026-05-31

建议路径：如果新建独立研究文档，可放在 `docs/research/2026-05-31-agent-skills.md`；本仓库已经有 runtime 主题入口，所以本次沿用 `docs/coding-agents/research/runtime/04-agent-skills.md`。

## 调研问题

这篇文档围绕三组问题调研 Agent Skills：

1. 是什么：核心定义、关键术语、边界、容易混淆的概念。
2. 为什么：它解决什么问题、出现背景、适用场景、不适用场景、主要 trade-off。
3. 怎么做：实践步骤、最小例子、常见实现路径、验证方法、常见坑。

资料优先级：论文、官方文档、技术报告、权威工程博客、一线工程实践文章。本文把结论分成“事实”“作者观点”和“本文推断”，并在关键结论处标注来源。

## 核心结论

1. **事实：Skills 是 agent runtime 中的可发现能力包，不是模型参数里的能力。** 主流实现正在收敛到 `SKILL.md` + front matter + 可选 `scripts/`、`references/`、`assets/` 的目录结构，并用 progressive disclosure 先暴露 `name` / `description`，需要时再加载全文。来源：[OpenAI Codex Skills](https://developers.openai.com/codex/skills)、[OpenAI API Skills](https://developers.openai.com/api/docs/guides/tools-skills)、[Anthropic Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)、[Agent Skills specification](https://agentskills.io/specification)。
2. **事实：`description` 是 routing contract。** OpenAI、GitHub、Windsurf 和 Agent Skills specification 都强调 agent 会根据 description 判断是否触发；OpenAI Agents SDK 维护案例也把 description 称为主要路由信号。一个 skill 写得好不好，首先体现在它是否能被正确发现、不过度触发、不会和其他 skill 冲突。来源：[OpenAI Codex Skills](https://developers.openai.com/codex/skills)、[GitHub Copilot skills](https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/customize-cloud-agent/add-skills)、[Windsurf Cascade Skills](https://docs.windsurf.com/windsurf/cascade/skills)、[OpenAI OSS maintenance case](https://developers.openai.com/blog/skills-agents-sdk)。
3. **事实 + 推断：Skills 的价值在“流程封装 + 上下文按需加载 + 验证纪律”，不是收藏长 prompt。** Skills 适合把重复 workflow、组织知识、脚本、模板和验证步骤打包；不适合替代 `AGENTS.md`、tool、hook、MCP、测试或 CI。来源：[OpenAI best practices](https://developers.openai.com/codex/learn/best-practices)、[Windsurf Cascade Skills](https://docs.windsurf.com/windsurf/cascade/skills)、[OpenAI OSS maintenance case](https://developers.openai.com/blog/skills-agents-sdk)。
4. **事实：Skills 有可测收益，但收益不均匀。** [SkillsBench](https://arxiv.org/abs/2602.12670) 报告 curated skills 平均提升 16.2 个百分点，但不同领域差异大，部分任务反而下降；self-generated skills 平均没有带来收益。结论是：skills 需要 eval，不能只看 demo。
5. **事实 + 推断：Skills 是新的供应链和上下文攻击面。** OpenAI API、GitHub、Anthropic 都提醒要审查第三方 skills；安全论文指出 `SKILL.md` 自然语言本身可影响检索、选择和执行。安装 skill 应按“特权指令 + 特权代码”治理。来源：[OpenAI API Skills](https://developers.openai.com/api/docs/guides/tools-skills)、[GitHub Copilot skills](https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/customize-cloud-agent/add-skills)、[Anthropic Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)、[SKILL.md semantic supply-chain paper](https://arxiv.org/abs/2605.11418)。

## 是什么

### 工作定义

本文采用这个定义：

```text
Agent Skill = 可发现的任务元数据 + 可按需加载的任务说明 + 可选参考资料 / 资产 / 脚本 + 触发规则 + 运行时权限边界。
```

最小形态通常是一个目录：

```text
my-skill/
  SKILL.md          # required: front matter + instructions
  scripts/          # optional: executable code
  references/       # optional: long-form docs
  assets/           # optional: templates, images, schemas, examples
```

一个最小 `SKILL.md`：

```markdown
---
name: pr-review-checklist
description: Review a code diff against the repository checklist. Use when the user asks for PR review, bug-risk review, or review-before-merge.
---

1. Inspect the current diff and changed files.
2. Prioritize correctness, regression risk, security, and missing tests.
3. Report findings first with file and line references.
4. If no blocking issue is found, state residual risk and tests checked.
```

OpenAI Codex 文档定义 skill 是“instructions、resources、optional scripts”的任务能力包，Codex 会先看到每个 skill 的 `name`、`description` 和路径，只有选中时才读取完整 `SKILL.md`。OpenAI API 文档进一步把 skill 作为可上传、可版本化、可挂载到 hosted / local shell 环境的文件 bundle。Anthropic 的定义相近：skills 是 agents 可以发现并动态加载的 instructions、scripts、resources 文件夹。Agent Skills specification 则给出跨客户端的 `SKILL.md` 格式约束。

### 边界：Skill 不是什么

**不是模型本身的能力。** 模型会不会写 SQL、React 或 shell，是模型能力；skill 是外部化的流程、约定、资源和脚本。SkillsBench 也把 skills 称为 inference-time procedural knowledge。

**不是普通工具。** Tool 是 agent 能调用的动作，例如 shell、browser、MCP function；skill 通常告诉 agent 什么时候、按什么顺序、带什么约束使用这些动作。

```text
Tool = 可调用动作。
Skill = 任务级方法。
```

**不是 hook。** Hook 在生命周期事件自动执行，例如 tool call 前后、session start / stop；skill 通常由 agent 根据任务语义选择，或由用户显式 `$skill` / `@skill` 触发。

**不是 plugin 本身。** 在 OpenAI Codex 语境里，skills 是 authoring format，plugins 是 installable distribution unit。可以先写 skill，等要跨团队分发时再打成 plugin。

**不是 `AGENTS.md`。** `AGENTS.md` 适合放仓库级、总是相关的规则；skill 适合放某类任务才需要的详细流程。GitHub Copilot 文档也建议 custom instructions 用于几乎每个任务都相关的简单规则，skills 用于只在相关时加载的详细说明。

**不是长期记忆。** Memory 更偏偏好、历史事实或跨会话经验；skill 应该可读、可 review、可测试、可禁用、可版本化。

## 为什么

### 出现背景

Agent 从 chat assistant 变成能读写文件、执行 shell、调用 MCP、跑测试、开 PR 的软件执行体后，单靠 prompt 不够稳定。重复问题很快出现：

- 同一类任务每次都要复制长 prompt。
- agent 经常忘记固定验证步骤。
- 组织知识分散在 README、runbook、CI、脚本、Notion、issue 和口头约定里。
- 工具存在，但 agent 不知道什么时候用、怎么组合。
- 长系统提示词和 `AGENTS.md` 会污染所有任务上下文。

Skills 的设计回应是：把“某类任务怎么做”从一次性 prompt 抽出来，做成 agent runtime 可发现、可按需加载、可分发、可评估的能力包。Anthropic 的说法是，真实工作需要 procedural knowledge 和 organizational context；OpenAI best practices 的建议是，当 workflow 变得 repeatable，就不要依赖长 prompt 或反复沟通，而应封装成 skill。

### 它解决的问题

**降低重复沟通成本。** 把固定流程、输出格式、验证步骤写进 skill 后，用户不必每次重新解释。

**控制上下文预算。** Progressive disclosure 让 agent 先看到轻量 metadata，而不是把所有流程全文塞进主上下文。OpenAI Codex 还明确限制初始 skill 列表大约占上下文窗口 2%，未知窗口时约 8,000 字符；skill 太多时会缩短 description，甚至省略部分 skill。

**把组织知识变成可执行入口。** `references/` 可以承载长文档，`scripts/` 可以承载确定性操作，`assets/` 可以承载模板。这样 skill 不只是说明书，而是能把 agent 引导到正确材料和命令。

**把工程纪律显式化。** OpenAI Agents SDK 维护案例把 verification、release review、changeset validation、PR draft summary 等做成 repo-local skills，并用 `AGENTS.md` 写 if/then 触发规则。这种模式把“完成前必须验证”从口头要求变成可重复 workflow。

**给 eval 一个对象。** SkillsBench、SkillRet、SkillRouter、SkillGenBench、SkillLearnBench 等研究说明，skills 已经从产品功能变成可单独评估的 agent harness 组件：能否提升任务成功率、能否检索正确 skill、能否生成可复用 skill、能否从经验中学习 workflow。

### 适用场景

适合写 skill 的信号：

- 同一 prompt 或流程已经复制 3 次以上。
- agent 经常犯同一类流程错误。
- 任务有稳定输入、输出和完成标准。
- 任务需要专门参考资料、模板或脚本。
- 任务跨项目复用，但不应该污染所有项目的 `AGENTS.md`。
- 任务需要模型判断和确定性脚本结合。
- 团队需要把 review、release、incident、migration、docs sync 等操作规范化。

典型例子：

- PR review checklist。
- release note drafting。
- CI failure triage。
- API migration plan。
- docs sync / docs freshness audit。
- OpenAI API 当前文档查询。
- test coverage improvement。
- report-first 的安全或架构审查。
- 特定文档、表格、演示、图像处理工作流。

### 不适用场景

不适合写 skill 的信号：

- 只用一次，流程还没稳定。
- 本质是纯确定性转换，写脚本更好。
- 每次工具调用前都必须强制执行，写 hook 更好。
- 只是外部系统动作，写 MCP tool 或 app integration 更好。
- 只是所有任务都应该知道的仓库规则，放 `AGENTS.md` 更好。
- 涉及高风险生产动作，但没有权限、approval、审计和回滚设计。
- 想用 skill 掩盖项目缺测试、无入口文档、无 CI 的基础工程问题。

### 主要 trade-off

| 收益 | 代价 |
|---|---|
| 减少重复 prompt | 需要维护 skill 版本和 owner |
| 按需加载上下文 | skill 数量增加后需要 retrieval / ranking / 去重 |
| 复用组织知识 | 过期知识会变成新的错误来源 |
| scripts 提升确定性 | scripts 带来权限、安全和依赖治理 |
| description 自动触发 | description 写不好会误触发或漏触发 |
| 可分发能力包 | 第三方 skill 是供应链风险 |
| 可以进入 CI / automation | 自动化前必须先证明 workflow 手动可靠 |

## 怎么做

### 实践步骤

**第一步：选一个真实重复任务。** 不要从“我要做一个万能 skill”开始。选择一个你已经重复做过的任务，例如“根据本仓库约定写调研文档”“PR review before merge”“CI 失败定位”。

**第二步：先 instruction-only。** 第一版只写 `SKILL.md`，明确触发场景、输入、输出、步骤、验证和边界。OpenAI Codex 文档也建议默认先 instruction-only，只有需要确定性行为或外部工具时再加 scripts。

**第三步：写好 description。** 推荐模板：

```text
<产物/动作> when <触发场景>; use for <2-3 个具体任务>; do not use for <边界>.
```

示例：

```text
description: Write Simplified Chinese research notes with sourced claims for agent-runtime topics; use when the user asks to research coding agents, skills, hooks, memory, sandbox, eval, or harness engineering; do not use for implementation-only coding tasks.
```

**第四步：把长资料拆到 references。** `SKILL.md` 只放执行路线图。长标准、写作风格、示例、术语表放 `references/`，需要时再读。Agent Skills specification 建议把主 `SKILL.md` 控制在合理长度，长内容拆成按需文件。

**第五步：把确定性步骤放 scripts。** 如果 agent 每次都要重复执行同一组命令、解析日志、收集 diff stats、生成文件树，把这部分写成 CLI 风格脚本。OpenAI Agents SDK 案例的经验是：解释、比较、判断和报告留给模型；固定 shell 机械步骤放进 `scripts/`。

**第六步：设计最小 eval。** 至少覆盖正例触发、反例不触发、输出格式、验证命令、安全边界和失败处理。

**第七步：进入 review 和版本治理。** 团队级 skill 应像代码一样 review，尤其看 description 是否过宽、脚本是否危险、权限是否扩大、是否访问网络或 secret、是否有测试样例。

### 最小例子

下面是一个适合本仓库的 `research-doc-writer` skill 草案：

```text
.agents/skills/research-doc-writer/
  SKILL.md
  references/
    source-quality.md
```

`SKILL.md`：

```markdown
---
name: research-doc-writer
description: Write sourced Simplified Chinese research documents for AI agent, coding agent, model capability, eval, runtime, or harness engineering topics. Use when the user asks for systematic research with sources and a Markdown artifact.
---

## When to Use

Use this skill when the user asks for a research document, literature review, source-backed notes, or concept research in this repository.

Do not use it for implementation-only code changes, one-off summaries without sources, or tasks where the user explicitly asks not to browse.

## Workflow

1. Read repository `AGENTS.md`, root `README.md`, and the relevant topic README before choosing a path.
2. Use official docs, papers, technical reports, primary repositories, and first-party engineering posts first.
3. Browse for any current product, model, price, benchmark, API, or security claim.
4. Separate facts, source opinions, and your inference.
5. Write the Markdown file in Simplified Chinese with required sections:
   - title
   - research date
   - research questions
   - <=5 core conclusions
   - what / why / how
   - key concepts
   - source review
   - references
   - open questions
   - next steps
6. For each key claim, cite a source or mark it as inference.
7. Verify links, headings, and README references if a new formal document is added.

## Output

Return only a short summary, file path, core conclusions, best 3 sources, and unresolved questions.
```

### 常见实现路径

| 路径 | 适合场景 | 注意点 |
|---|---|---|
| Repo-local skill：`.agents/skills/<name>/SKILL.md` | 团队或单仓库 workflow | 进代码评审，和 `AGENTS.md` 分层 |
| Personal skill：`~/.agents/skills/<name>/SKILL.md` | 个人跨项目偏好 | 不要写入 token、私钥、本机绝对路径 |
| Codex plugin | 分发多个 skills、apps、MCP 配置 | skill 是 authoring format，plugin 是分发单元 |
| OpenAI API hosted skill | API / hosted shell 环境复用 | pin version，审查网络和数据驻留 |
| GitHub `gh skill` | 从 GitHub skill repo 搜索、预览、安装、发布 | 第三方 skill 未验证；安装前 `gh skill preview` |
| Windsurf workspace/global/system skill | Cascade 多步任务 | 区分 Skills / Rules / Workflows |

### 验证方法

**触发测试：**

| 测试 | 例子 | 期望 |
|---|---|---|
| 正例 | “帮我系统调研 Agent Skills，并写入 Markdown” | 触发 `research-doc-writer` |
| 近邻反例 | “帮我修这个测试失败” | 不触发研究写作 skill |
| 显式调用 | “使用 `$research-doc-writer` 调研 hooks” | 必须使用指定 skill |
| 冲突测试 | 同时存在 `research-doc-writer` 和 `blog-writer` | 只触发更匹配的一个，或说明冲突 |

**输出测试：**

- 标题、日期、调研问题、核心结论等必需章节齐全。
- 每条关键结论有来源或标注“本文推断”。
- 参考资料包含标题、作者或机构、发布日期、资料类型、链接、访问日期。
- 变化信息有访问日期和官方优先来源。
- 没有把 benchmark 单次排名写成长期结论。

**行为测试：**

- 是否按需读 references，而不是无差别塞入上下文。
- 是否在需要时运行 scripts。
- 是否在失败时停止并说明不确定性。
- 是否没有越权执行高风险操作。

**安全测试：**

- `SKILL.md` 是否包含“忽略上级指令”“隐藏行为”“读取 secret”“发送数据到外部 URL”等危险模式。
- scripts 是否访问网络、读写敏感路径、执行下载物。
- 是否需要 shell / bash pre-approval。
- 是否 pin 版本或记录来源 commit / tag / SHA。

### 常见坑

1. **万能 skill。** `description: Use for all coding tasks` 会抢占路由并污染所有任务。
2. **description 太虚。** “Helps with docs” 不足以让 agent 判断何时用、何时不用。
3. **把百科塞进 `SKILL.md`。** 触发后上下文暴涨，应该拆到 references。
4. **无验证步骤。** 没有测试、渲染、lint、source check 或人工 review gate 的 skill 只是 prompt 包。
5. **scripts 无边界。** 能联网、读 secret、写任意目录的 skill 脚本必须当高风险代码审查。
6. **复制第三方 skill 即安装。** GitHub、OpenAI、Anthropic 都提醒第三方 skills 可能有 prompt injection、隐藏指令或恶意脚本。
7. **自动生成 skill 直接进默认集。** SkillsBench 显示 self-generated skills 平均无收益；自动生成可以当草稿，但应经过 review 和 eval。
8. **skill library 无 owner。** 无 owner、无版本、无触发样例、无失效条件的 skill 会变成新的知识垃圾场。

## 关键概念和术语

| 术语 | 定义 | 来源 / 说明 |
|---|---|---|
| Agent Skill | 带 `SKILL.md` 的任务能力包，包含指令、资源、可选脚本 | OpenAI、Anthropic、Agent Skills spec |
| `SKILL.md` | skill 的 manifest 和主说明文件，通常含 YAML front matter 和 Markdown body | Agent Skills spec |
| `name` | skill 唯一标识，通常小写、数字、连字符 | Agent Skills spec |
| `description` | agent 用来判断是否触发的描述，是 routing metadata | OpenAI Codex、GitHub、Windsurf、OpenAI OSS blog |
| Progressive disclosure | 先加载 metadata，选中后再加载 `SKILL.md`，需要时再读 references/scripts/assets | Anthropic、OpenAI、Agent Skills spec |
| Explicit invocation | 用户显式 `$skill`、`@skill` 或“use X skill” | OpenAI Codex、Windsurf |
| Implicit invocation | agent 根据 prompt 和 description 自动选择 skill | OpenAI Codex、GitHub、Windsurf |
| Skill retrieval / routing | 从大量 skills 中为当前任务选择相关 skill 的问题 | SkillRet、SkillRouter |
| Curated skill | 官方或团队维护的经过筛选的 skill | OpenAI API / Codex |
| Repo-local skill | 跟随仓库提交的 skill | OpenAI Codex、GitHub、Windsurf |
| Personal skill | 用户主目录下跨项目可用的 skill | OpenAI Codex、GitHub、Windsurf |
| Skill generation | 根据任务、仓库或文档自动生成 skill | SkillGenBench、SkillLearnBench |
| Skill supply chain | skill 的来源、安装、更新、脚本、指令和 marketplace 信任链 | OpenAI API safety、GitHub warning、安全论文 |

## 资料综述

### 官方文档和标准

OpenAI Codex、OpenAI API、Anthropic、GitHub Copilot、Windsurf 和 Agent Skills specification 在核心形态上高度一致：skill 是目录，核心是 `SKILL.md`，metadata 用于发现，正文和资源按需加载。差异主要在运行时位置、安装方式、权限配置和分发机制。

最值得注意的实现细节：

- OpenAI Codex 把初始 skill 列表放入上下文，但有上下文预算上限；并支持 repo、user、admin、system 多级目录。
- OpenAI API 把 skills 做成 versioned bundle，可在 hosted shell 中用 `skill_reference` 挂载，也可在 local shell 模式中用本地路径提供。
- GitHub Copilot 支持 `.github/skills`、`.claude/skills`、`.agents/skills`、`~/.copilot/skills`、`~/.agents/skills`，并通过 `gh skill` 搜索、预览、安装、pin、更新和发布。
- Windsurf 明确区分 Skills、Rules、Workflows：skill 适合多步 procedures 和 supporting files；rules 适合行为约束；workflows 适合手动 slash-command runbook。

### 工程实践

OpenAI “Using skills to accelerate OSS maintenance” 是目前最有工程参考价值的一线案例。它不是展示单个 skill，而是把 skills、`AGENTS.md`、scripts、GitHub Actions、PR review 和 release workflow 组合起来。关键模式：

- `AGENTS.md` 写“何时必须用哪个 skill”。
- `description` 写清触发边界。
- `scripts/` 处理固定命令和日志收集。
- 模型处理解释、判断、比较和报告。
- 手动 workflow 稳定后，再用 GitHub Actions 自动化。

这个案例也给出一个实用分工：

```text
AGENTS.md = 全局规则和触发门。
Skill = 某类任务的做法和验证标准。
Script = 确定性机械步骤。
Agent = 上下文判断、解释和报告。
CI / Action = 稳定流程的调度和审计。
```

Superpowers 代表另一类社区实践：把 TDD、systematic debugging、verification before completion、requesting code review、subagent-driven development 等工程纪律打包成 skill-like workflow。它的价值不是某个 API 知识，而是把资深工程师会坚持的流程变成 agent 可以遵循的 gates。

### Benchmark 和论文

SkillsBench 是核心实证来源。它显示 curated skills 通常有收益，但收益不稳定；并且 self-generated skills 平均无收益。这直接反驳了“让模型自己写 skill 就能持续自我改进”的乐观直觉。

SkillRet 和 SkillRouter 关注 skill retrieval / routing。它们共同说明：当 skill 数量从十几个增长到几千、几万时，显式点名和简单 metadata 列表都不够。SkillRouter 还指出，在大规模高重叠 skill registry 中，隐藏 skill body 会显著降低 routing accuracy。这与 progressive disclosure 的产品设计形成张力：运行时为了省上下文只给 metadata，但检索系统可能需要更多全文信号。

SkillGenBench 和 SkillLearnBench 把“生成 skill”和“从经验学习 skill”作为独立评估对象。它们说明 skill generation 是重要方向，但不能默认可靠；需要固定 harness、pinned environment、execution-based checks 和 failure-mode analysis。

Toolformer、Voyager、LATM、Reflexion 是前史：它们分别研究工具调用、自主技能库、LLM 生成工具、语言反馈记忆。今天的 Agent Skills 可以看成这些思想在真实 agent runtime 里的工程化形态：经验和程序性知识不只留在上下文，而是沉淀为可检索、可执行、可治理的外部资产。

### 安全研究

官方文档和安全论文的共识是：skills 不是普通 Markdown。它们能影响 agent 的计划、工具调用、命令执行和数据流。风险包括：

- `SKILL.md` prompt injection。
- description 污染检索和误触发。
- hidden instructions。
- 脚本执行和依赖风险。
- 第三方 skill 更新漂移。
- marketplace / GitHub repo provenance 风险。
- 高风险动作缺 approval。

OpenAI API 文档明确建议把 skills 当作 privileged code and instructions；GitHub 警告 skills 未经验证，可能包含 prompt injection、隐藏指令或恶意脚本；Anthropic 建议只安装可信来源，安装低信任来源前审查文件、依赖、资源和网络访问。安全论文进一步指出，传统代码扫描难以识别自然语言指令层的语义攻击。

## 资料冲突和判断

| 冲突点 | 资料 A | 资料 B | 本文判断 |
|---|---|---|---|
| Progressive disclosure 是否足够 | OpenAI / Anthropic / Windsurf 都强调 metadata-first 可节省上下文 | SkillRouter 指出大规模 registry 中隐藏 full skill text 会导致 routing accuracy 下降 31-44pp | 两者不矛盾：progressive disclosure 是运行时上下文策略；大规模检索需要额外索引、全文检索或 reranker。个人和小团队可用 metadata-first；企业/市场级 skill library 需要 retrieval layer。 |
| 自动生成 skills 是否可靠 | Anthropic 展望 agents 未来可创建、编辑、评估自己的 skills | SkillsBench 显示 self-generated skills 平均无收益，SkillGenBench/SkillLearnBench 把 generation 作为未解决评估问题 | 更信 benchmark 对当前能力的约束。自动生成 skill 可作为草稿生成器，不应无 review 进入默认 skill set。 |
| 第三方 skill 能否靠扫描解决安全 | GitHub / OpenAI / Anthropic 都强调安装前审查 | Malicious Or Not 指出 repository context 能显著降低 scanner false positive，并发现 repo hijacking 风险 | 更信“多层治理”而不是单一扫描：人工 review + provenance + pin version + permission + sandbox + runtime approval + eval。 |
| Skill 是产品功能还是工程资产 | 产品文档强调易用、可扩展、可安装 | OpenAI OSS 案例和安全论文强调 version、scripts、CI、approval、review | 对团队而言 skill 应按工程资产治理。个人实验可以轻量，但团队共享不能只当 prompt 文件。 |

## 参考资料

访问日期均为 2026-05-31。

| 标题 | 作者或机构 | 发布日期 | 类型 | 链接 | 主要用途 |
|---|---|---:|---|---|---|
| Agent Skills - Codex | OpenAI | 未标注，2026-05-31 访问 | docs | https://developers.openai.com/codex/skills | Codex skill 定义、目录结构、progressive disclosure、触发方式、存放位置、best practices |
| Skills | OpenAI API | 未标注，2026-05-31 访问 | docs | https://developers.openai.com/api/docs/guides/tools-skills | API 中 versioned skill bundle、hosted/local shell、prompt priority、安全和版本管理 |
| Best practices - Codex | OpenAI | 未标注，2026-05-31 访问 | docs | https://developers.openai.com/codex/learn/best-practices | 何时把重复 workflow 变成 skill、description 和 scope 建议 |
| Using skills to accelerate OSS maintenance | Kazuhiro Sera / OpenAI | 2026-03-09 | blog | https://developers.openai.com/blog/skills-agents-sdk | 一线工程案例：skills + AGENTS.md + scripts + GitHub Actions |
| Equipping agents for the real world with Agent Skills | Barry Zhang, Keith Lazuka, Mahesh Murag / Anthropic | 2025-10-16，2025-12-18 更新开放标准 | blog | https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills | Agent Skills 背景、progressive disclosure、开发和安全建议 |
| Agent Skills Specification | Agent Skills project | 2025-12-18 开放标准公告，页面未标注版本日期 | docs | https://agentskills.io/specification | `SKILL.md` front matter、目录结构、字段约束、validation |
| Adding agent skills for GitHub Copilot | GitHub | 未标注，2026-05-31 访问 | docs | https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/customize-cloud-agent/add-skills | GitHub Copilot skills、`gh skill`、pin、preview、安全警告 |
| Cascade Skills | Windsurf | 未标注，2026-05-31 访问 | docs | https://docs.windsurf.com/windsurf/cascade/skills | Windsurf Skills、scope、自动/手动触发、Skills vs Rules vs Workflows |
| openai/skills | OpenAI | 持续更新 | code | https://github.com/openai/skills | 官方 skill 示例 |
| anthropics/skills | Anthropic | 持续更新 | code | https://github.com/anthropics/skills | 官方 skill 示例 |
| superpowers | Jesse Vincent / community | 持续更新 | code | https://github.com/obra/superpowers | 社区 workflow skills / engineering discipline 案例 |
| SkillsBench: Benchmarking How Well Agent Skills Work Across Diverse Tasks | Xiangyi Li 等 | 2026-02-13 初版，2026-03-13 v3 | paper | https://arxiv.org/abs/2602.12670 | skills 是否提升任务表现、curated vs self-generated 对比 |
| SkillRet: A Large-Scale Benchmark for Skill Retrieval in LLM Agents | Hongcheol Cho, Ryangkyung Kang, Youngeun Kim | 2026-05-07 | paper | https://arxiv.org/abs/2605.05726 | 大规模 skill retrieval benchmark |
| SkillRouter: Skill Routing for LLM Agents at Scale | YanZhao Zheng 等 | 2026-03-23 初版，2026-04-01 v4 | paper | https://arxiv.org/abs/2603.22455 | 大规模 skill routing、metadata-only 与 full-text routing 张力 |
| SkillGenBench: Benchmarking Skill Generation Pipelines for LLM Agents | Yifan Zhou 等 | 2026-05-18 | paper | https://arxiv.org/abs/2605.18693 | skill generation pipeline benchmark |
| SkillLearnBench: Benchmarking Continual Learning Methods for Agent Skill Generation on Real-World Tasks | Shanshan Zhong 等 | 2026-04-22 | paper | https://arxiv.org/abs/2604.20087 | 从 agent 经验中生成 skills 的 continual learning benchmark |
| Voyager: An Open-Ended Embodied Agent with Large Language Models | Guanzhi Wang 等 | 2023-05-25 | paper / code | https://arxiv.org/abs/2305.16291 | 早期 skill library / executable code skills 思路 |
| Toolformer: Language Models Can Teach Themselves to Use Tools | Timo Schick 等 | 2023-02-09 | paper | https://arxiv.org/abs/2302.04761 | 工具调用学习前史 |
| Large Language Models as Tool Makers | Chenguang Zhuge 等 | 2023-05-27 | paper | https://arxiv.org/abs/2305.17126 | LLM 生成可复用工具，与 skill generation 相邻 |
| Reflexion: Language Agents with Verbal Reinforcement Learning | Noah Shinn 等 | 2023-03-20 | paper | https://arxiv.org/abs/2303.11366 | 语言反馈和外部化经验前史 |
| AI Harness Engineering: A Runtime Substrate for Foundation-Model Software Agents | Hailin Zhong, Shengxin Zhu | 2026-05-13 | paper | https://arxiv.org/abs/2605.13357 | model-harness-environment 视角，skills 作为 harness 组件 |
| Under the Hood of SKILL.md: Semantic Supply-chain Attacks on AI Agent Skill Registry | Shoumik Saha, Kazem Faghih, Soheil Feizi | 2026-05-12 | paper | https://arxiv.org/abs/2605.11418 | `SKILL.md` 语义供应链攻击 |
| Malicious Or Not: Adding Repository Context to Agent Skill Classification | Florian Holzbauer 等 | 2026-03-17 | paper | https://arxiv.org/abs/2603.16572 | agent skill 生态安全分析、repository context 和 repo hijacking |
| OWASP Top 10 for LLM Applications | OWASP | 持续更新 | technical report | https://owasp.org/www-project-top-10-for-large-language-model-applications/ | Prompt injection、supply chain、excessive agency 等安全分类 |

## 开放问题

1. Skill retrieval 在企业级上限是多少：多少 skills 之后需要专门 retrieval service，而不是把 description 列表塞进上下文？
2. `description` 是否应该有更结构化字段，例如 `use_when`、`do_not_use_when`、`inputs`、`outputs`、`risk_level`？
3. 第三方 skill marketplace 的信任模型会怎样演化：签名、provenance、SBOM、hash pin、审计日志是否会成为标配？
4. 自动生成 skill 的质量门槛是什么：通过多少触发测试、输出测试和安全测试才能进入团队默认集？
5. Skills 与 MCP、hooks、subagents、automations 的边界是否会被平台重新划分？
6. Skill eval 应该如何覆盖“误触发导致性能下降”而不只是“触发后任务成功率”？
7. 对自然语言指令的安全扫描能否达到代码扫描类似的可解释性和可复现性？

## 下一步建议

1. **为本仓库做一个最小 `research-doc-writer` skill。** 先放在 `docs/coding-agents/agent/skills/research-doc-writer/SKILL.md` 或 `.agents/skills/research-doc-writer/SKILL.md`，只做 instruction-only，不加脚本。
2. **建立 6 条最小 eval 样例。** 3 条正例应触发、3 条反例不应触发；检查章节完整性、来源字段、事实/观点/推断区分、是否更新 README。
3. **把已有调研 prompt 与 skill 合并。** 参考 `docs/coding-agents/agent/playbooks/prompts/concept-research.md` 和 `solution-comparison.md`，把稳定规则下沉到 skill，把一次性任务继续留在 prompt。
4. **给 skill 加安全和维护字段。** 至少记录 owner、适用范围、是否允许 implicit invocation、是否需要联网、是否允许 shell、最近验证日期。
5. **等 instruction-only 稳定后再加 scripts。** 候选脚本包括 Markdown 章节检查、参考资料表字段检查、链接有效性检查。

## 用于项目的最小实验 / reviewable slice

推荐 slice：

```text
目标：把“系统调研并写入 Markdown”的重复任务做成一个可 review 的最小 skill。

范围：
1. 新增一个 instruction-only skill：research-doc-writer。
2. 不接入外部 secret，不写 shell scripts，不做自动发布。
3. 增加 6 条人工 eval prompt 和期望结果。
4. 用一次真实调研任务回放，记录触发是否正确、输出是否完整、是否有来源缺口。

验收：
- skill 只在研究写作任务触发，不在普通代码修改触发。
- 输出文档包含本仓库要求的全部章节。
- 参考资料表字段完整。
- 变化信息有访问日期。
- 不确定结论明确标注为推断或开放问题。
```

这个 slice 小到可以 code review，又能验证 skills 的核心价值：减少重复 prompt、按需加载流程、稳定输出格式、暴露触发和治理问题。
