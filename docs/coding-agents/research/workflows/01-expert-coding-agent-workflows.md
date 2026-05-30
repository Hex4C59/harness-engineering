# 编程高手如何使用 Coding Agent：经验调研

调研日期：2026-05-31

## 这篇文档回答什么

这篇文档调研一批资深工程师、重度 coding agent 用户和官方工程团队公开分享的经验，回答：

1. 真正会写代码的人，怎么用 coding agent？
2. 他们把 agent 当成什么角色？
3. 他们在写代码前、写代码中、写代码后分别做什么？
4. 哪些做法有共识，哪些做法存在分歧？
5. 这些经验对你现在的 `AGENTS.md + docs 渐进式上下文 + TDD` 工作流有什么启发？

核心结论：

> 高手不是把 coding agent 当“自动程序员”，而是把它放进一个明确的工程系统里：人类负责目标、边界、判断和 review；agent 负责搜索、实现、机械修改、测试补齐、方案探索和重复劳动。真正拉开差距的不是一句神奇 prompt，而是规格、上下文、测试、权限、review、并行隔离和失败回流。

## 先回答：`docs/coding-agents/agent` 算 harness 还是提示词？

你的 `docs/coding-agents/agent/` 更准确地说是 **agent harness assets**，不是单纯 prompt。

它里面既有 prompt，也有更高层的 harness 组件：

| 目录 | 更像什么 | 作用 |
|---|---|---|
| `playbooks/prompts/` | Prompt library | 可复制给 agent 的单次任务指令 |
| `playbooks/workflows/` | Workflow harness | 规定某类任务的步骤、入口和退出条件 |
| `playbooks/checklists/` | Review / gate harness | 让任务在开始前、实现中、完成前有放行条件 |
| `playbooks/templates/` | Project harness bootstrap | 复制到项目里的 `AGENTS.md`、docs、scripts 模板 |
| `playbooks/principles/` | Decision framework | 在依赖、架构、TDD、reviewability 等场景给判断标准 |
| `skills/` | Runtime skill assets | 可升级成 Codex / Claude / Copilot 等 runtime 可发现的能力包 |
| `mcp/` | Tool connection assets | 记录 MCP 工具连接、权限和安全边界 |

所以它的定位可以写成：

```text
这不是“提示词合集”，而是一套轻量 agent harness 工具箱。
Prompt 是其中一层；真正的目标是把上下文、流程、验证、权限和 review 组织起来。
```

这和 OpenAI Codex、Claude Code、GitHub Agent HQ、Jesse Vincent 的 Superpowers、社区 Claude Code workflow 讨论里的方向一致：高手越来越少依赖一次性长 prompt，更多把常用流程拆成 `AGENTS.md` / `CLAUDE.md`、commands、skills、hooks、scripts、subagents、worktrees、CI 和 PR review。

## 来源说明

本文优先使用作者原文、官方工程博客和带实验设计的技术报告。社区讨论只作为线索，不作为主要依据。

| 来源 | 作者 / 团队 | 类型 | 主要价值 |
|---|---|---|---|
| [How OpenAI uses Codex](https://cdn.openai.com/pdf/6a2631dc-783e-479b-b1a4-af0cfbd38630/how-openai-uses-codex.pdf) | OpenAI | 技术报告 / 工作流报告 | OpenAI 内部如何把 Codex 用在 full-stack、product、infrastructure、security、docs 和 enterprise workflows |
| [Codex best practices](https://developers.openai.com/codex/learn/best-practices) | OpenAI | 官方文档 | `AGENTS.md`、任务拆分、验证、skills、review、长任务和并行工作的实践建议 |
| [Thoughts on coding agents](https://dennybritz.com/posts/coding-agents) | Denny Britz | 资深工程师个人经验 | Codex / Claude CLI 工作流、规划模式、diff review、并行 agent 的边界 |
| [My LLM coding workflow going into 2026](https://addyosmani.com/blog/ai-coding-workflow) | Addy Osmani | 资深工程师实践总结 | spec 先行、小步迭代、上下文打包、测试作为 guardrail |
| [Embracing the parallel coding agent lifestyle](https://simonwillison.net/2025/Oct/5/parallel-coding-agents) | Simon Willison | 资深开发者实践总结 | 并行 agent 的适用场景、review 瓶颈、research / maintenance / POC 模式 |
| [How I'm using coding agents in September, 2025](https://blog.fsck.com/2025/10/05/how-im-using-coding-agents-in-september-2025) | Jesse Vincent | 重度用户流程分享 | worktree、brainstorm、plan、architect / implementer 双会话 |
| [Superpowers: How I'm using coding agents in October 2025](https://blog.fsck.com/2025/10/09/superpowers) | Jesse Vincent | skill / plugin 方法论 | 把 brainstorm、planning、TDD、review、worktree 编码成可复用 skills |
| [Just Talk To It](https://steipete.me/posts/just-talk-to-it) | Peter Steinberger | Codex CLI 重度使用经验 | blast radius、短 prompt、截图、并行终端、同目录多 agent 的高强度个人流派 |
| [The 7 Prompting Habits of Highly Effective Engineers](https://sketch.dev/blog/seven-prompting-habits) | Josh Bleecher Snyder / Sketch | agent prompting 技巧 | scout、示范一次再让 agent 复制、把人类意图转成可执行上下文 |
| [Claude Code best practices](https://www.anthropic.com/engineering/claude-code-best-practices) | Anthropic | 官方最佳实践 | `CLAUDE.md`、上下文管理、subagents、hooks、skills、writer / reviewer 模式 |
| [Claude Code power user tips](https://support.claude.com/en/articles/14554000-claude-code-power-user-tips) | Anthropic | 官方帮助文档 | 多会话、多 worktree、计划模式、IDE 集成、custom slash commands、安全和上下文技巧 |
| [How and when to use subagents in Claude Code](https://claude.com/blog/how-and-when-to-use-subagents-in-claude-code) | Anthropic | 官方工程博客 | subagent、skill、hook 的边界，以及何时把工作委托给专门 agent |
| [Pick your agent: Use Claude and Codex on Agent HQ](https://github.blog/news-insights/company-news/pick-your-agent-use-claude-and-codex-on-agent-hq) | GitHub | 官方产品 / 工程实践 | 多 agent 比较方案、PR 内 review、企业权限和审计 |
| [Karpathy skills on OpenClaw](https://www.augmentcode.com/blog/karpathy-skills-on-openclaw-agents-don-t-write-better-code-but-they-do-it-more-efficiently) | Augment Code | 实验报告 | `AGENTS.md` 规则对不同 agent harness 的影响，强调轻量规则要校准 |
| [Claude.md, rules, hooks, agents, commands, skills...](https://www.reddit.com/r/ClaudeCode/comments/1pxou18/claudemd_rules_hooks_agents_commands_skills/) | r/ClaudeCode 社区 | 社区讨论 | 开发者如何区分 `CLAUDE.md`、skills、hooks、commands、agents、MCP；只作社区信号 |
| [Do you actually use hooks in Claude Code?](https://www.reddit.com/r/ClaudeCode/comments/1tkvg6t/do_you_actually_use_hooks_in_claude_code/) | r/ClaudeCode 社区 | 社区讨论 | hooks 常见用途：编辑后跑 lint/typecheck、阻止越界命令、替代反复写在 `CLAUDE.md` 的提醒 |

注意：这些人不是完全同一种流派。Simon Willison 和 Jesse Vincent 更偏“计划、隔离、并行但谨慎”；Peter Steinberger 更偏“短 prompt、高频并行、凭经验控制 blast radius”；Denny Britz 则明显提醒大家不要高估并行 agent，因为真正瓶颈经常是人类 review 和上下文判断。Reddit / HN 这类社区材料只作为“用户正在怎么组织工具”的信号，不作为事实结论的唯一依据。

## 总体共识

把这些经验压缩成一句话：

```text
不要让 agent 替你决定工程秩序。
你要先建立工程秩序，再让 agent 在秩序里高速执行。
```

更具体一点：

1. 先明确目标、非目标、验收标准，再让 agent 写代码。
2. 任务越大，越需要先写 spec / plan。
3. 测试、类型检查、lint、CI 是 agent 的外部大脑。
4. 每次修改都要小到能 review、能回滚、能解释。
5. 并行 agent 适合 research、POC、低风险维护、独立 worktree，不适合把多个大改同时塞进一个难 review 的变更。
6. `AGENTS.md` / `CLAUDE.md` / skills / slash commands 的价值不是“提示词玄学”，而是把重复工作流变成稳定入口。
7. 经验丰富的人并没有退出循环，他们只是从“敲代码的人”变成“上下文提供者、任务拆分者、reviewer 和系统设计者”。

## 共识一：人类负责判断，Agent 负责加速

Denny Britz 的观察很清楚：现在很多中等规模项目可以做到“人类不手写代码”，但这不等于软件工程消失了。工作只是从代码语言转移到自然语言、测试、review 和上下文管理里。

高手会把 agent 当成：

- 很快的执行者。
- 会读代码的研究助理。
- 会写样板和测试的初级同事。
- 可以尝试方案的实验分身。
- 可以做第一轮 review 的辅助 reviewer。

但他们不会把 agent 当成：

- 产品负责人。
- 最终架构判断者。
- 安全负责人。
- 不需要验收的自动提交机器。
- 替你理解业务的外包团队。

实际工作中，人类仍然要做这些判断：

- 这个功能到底要不要做。
- 哪些场景是非目标。
- 哪个方案符合长期架构。
- 哪些依赖、接口、抽象是过度设计。
- 哪些测试才是真正有效的验收。
- 生成代码是否值得合并。

这解释了为什么高手经常说“模型差异没你想象的大”。对复杂任务来说，模型表现很大程度取决于你给了什么上下文、如何拆任务、有没有测试和 review，而不只是模型榜单排名。

## 共识二：先规格和计划，再进入实现

Addy Osmani、Jesse Vincent、Anthropic 官方实践都强调：非平凡任务不要直接让 agent 写代码，先让 agent 帮你把需求问清楚、写成 spec，再拆成计划。

一个可用 spec 至少包含：

- 目标：这次要解决什么问题。
- 非目标：明确不做什么，防止 agent 扩大范围。
- 当前上下文：相关文件、模块、接口、已有约定。
- 设计选择：为什么选这个方案，不选另一个方案。
- 边界条件：异常输入、权限、并发、失败、兼容性。
- 测试计划：先写哪些失败测试，最后跑哪些验证命令。
- 验收标准：完成的证据是什么。

Jesse Vincent 的做法很适合借鉴：先让一个会话当 architect，负责和你 brainstorm、写设计和实施计划；再开一个新会话当 implementer，只读设计和计划，然后分批执行前 3-4 个任务；最后回到 architect 会话审查实现。

这个模式的关键不是“多一个 agent 很酷”，而是：

```text
计划上下文和实现上下文分离。
实现者按计划执行。
审查者用相对新鲜的视角看 diff。
```

### 什么时候不需要完整计划

不是每个改动都要长 spec。比较合理的分界线是：

| 任务 | 推荐方式 |
|---|---|
| 改一个 typo、改文案、改配置 | 直接改，但完成前仍要验证 |
| 小 bug、有明确复现 | 先写回归测试，再最小修复 |
| 新功能、影响多个文件 | 先写短计划 |
| 架构调整、迁移、跨模块变化 | 写完整 spec + plan |
| 需求还不清楚 | 先让 agent 采访你，不要写代码 |

一句实战规则：

```text
如果你自己也说不清验收标准，就不要让 agent 开始实现。
```

## 共识三：上下文是工程资产，不是聊天附件

高手普遍会把上下文沉淀在项目里，而不是每次重新和 agent 解释。

常见载体包括：

- `AGENTS.md` / `CLAUDE.md`：入口、全局规则、命令、边界。
- `README.md`：项目目标、运行方式、常见任务。
- `docs/architecture/`：架构、模块关系、关键约束。
- `docs/plans/`：当前任务计划和历史决策。
- `docs/testing.md`：测试策略、命令、如何写回归测试。
- `docs/troubleshooting.md`：踩坑记录和失败模式。
- skills / commands：把重复工作流固化成可调用能力。

但他们也警惕“上下文越多越好”。Anthropic 和 Ran Isenberg 都提醒，`CLAUDE.md` 太长会让重要规则淹没在噪声里。Augment 的实验也说明，轻量规则能降低工具调用、成本和时间，但不同 harness 上效果不一致，甚至可能让某些 agent 过度保守。

比较好的结构是：

```text
AGENTS.md：入口地图 + 硬约定 + 必跑命令
docs/README.md：渐进式读取路径
docs/*：按主题保存事实和流程
scripts/*：把可确定流程变成命令
skills/commands：把重复 agent 流程变成可复用入口
```

这和你现在的做法完全同向：`AGENTS.md` 不应该是百科全书，它应该像项目的“操作台”。

## 共识四：测试是 Coding Agent 的刹车、仪表盘和验收官

Addy Osmani 的总结很直接：有好测试时 agent 会飞；没有测试时 agent 只会“看起来差不多”。Denny Britz 也指出，强类型、编译器、集成测试、行为测试会让 agent 更可靠，因为它们给 agent 提供即时反馈。

高手使用测试的方式通常不是“最后补一下”，而是贯穿全过程：

1. 先让 agent 找到现有测试风格。
2. 对 bug 先写失败的回归测试。
3. 对新功能先写最小验收测试。
4. 实现只写到测试通过为止。
5. 重构后继续跑测试。
6. 完成前跑项目级 `check`。

这就是 TDD 对 coding agent 特别有用的原因：

```text
TDD 把“我觉得它对了”变成“有一个外部系统证明它更接近对”。
```

尤其适合 agent 的测试类型：

- 高层行为测试：能跨重构保留意图。
- 回归测试：锁住真实 bug。
- 类型检查：快速暴露接口破坏。
- lint / formatter：减少风格 review 成本。
- e2e / 浏览器截图：对前端和交互类任务特别重要。

不适合过度依赖的测试：

- 只测实现细节的脆弱单测。
- agent 为了覆盖率随手写的空洞测试。
- 没有断言真实业务行为的 snapshot。
- 和需求没有关系的“顺手补测试”。

## 共识五：小步、局部、可回滚

Andrej Karpathy 风格规则在 Augment 的实验中被总结成四类：先想再写、简单优先、局部修改、目标驱动。实验结论很有启发：这些规则不一定让代码质量显著提高，但经常能减少工具调用、时间、成本和无关改动。

这说明一件事：

```text
AGENTS.md 里的规则最有价值的作用，不是让模型突然变聪明，
而是减少它乱逛、乱改、乱加文档、乱做 side quest。
```

Peter Steinberger 用了一个很实用的概念：blast radius，也就是一次改动会影响多大范围。高手在给 agent 下任务前，会先估计：

- 会碰几个文件？
- 会不会改公共接口？
- 会不会牵动数据迁移？
- 会不会影响权限、支付、删除、网络、部署？
- 如果做错，回滚是否容易？

如果不确定，他们会先问：

```text
先不要改代码。请给我 2-3 个方案，说明每个方案会影响哪些文件、风险和验证方式。
```

这比“你看着办”安全得多。

## 共识六：Review 是真正瓶颈

Simon Willison 和 Denny Britz 都提醒：agent 生成代码很快，但人类 review 没有同等加速。并行越多，review 债越多。

高手的 review 重点不是格式，而是：

- 是否真的满足 spec。
- 是否改了不该改的地方。
- 是否引入隐式依赖。
- 是否破坏兼容性。
- 是否遗漏失败路径。
- 是否有测试证明。
- 是否能被未来的人理解。

GitHub Agent HQ 的思路也类似：agent 产物应该进入原本的 PR / issue / review 流程，而不是另起一套“AI 已经说可以”的流程。

一个很好的模式是 Writer / Reviewer：

```text
Session A：按计划实现。
Session B：只看计划和 diff，找 correctness gap。
Session A：修复真实问题。
人类：最终 review 和合并。
```

注意 reviewer prompt 要克制。不要让 reviewer 泛泛“找问题”，否则它可能为了完成任务发明问题。更好的要求是：

```text
只报告影响正确性、需求覆盖、测试缺口、兼容性或安全性的发现。
不要报告纯风格偏好。
```

## 共识七：并行 Agent 有用，但不是越多越好

这里有明显分歧。

Simon Willison 一开始怀疑并行 agent，后来接受了选择性并行。他认为适合并行的任务主要是：

- research：让 agent 调研方案、库、代码路径。
- POC：验证某个库能不能用。
- 小维护：修 warning、升级小依赖、补文档缺口。
- 低风险机械修改：在测试充分的前提下改 imports、rename、格式迁移。
- 方案比较：让不同 agent 给出不同实现路径。

Denny Britz 更保守：如果一个任务定义得足够好，agent 往往几分钟就完成；真正瓶颈是人类给反馈和 review。并行太多会制造上下文切换疲劳。

Jesse Vincent 采用 worktree 隔离和 architect / implementer 角色分工。Peter Steinberger 则高强度使用 3-8 个 Codex CLI，有时在同一目录并行工作，但这是建立在他对项目、git、任务范围和回滚有很强直觉的前提下。对大多数个人项目，直接照搬同目录多 agent 风险很高。

更稳妥的规则：

```text
Research 可以并行。
独立 worktree 可以并行。
低风险维护可以并行。
同一目录多 agent 只适合你非常熟悉项目、能快速 review 和回滚时使用。
```

## 共识八：权限和环境要分层

高手不靠 prompt 说“不要做危险事”，而是尽量让环境本身降低风险。

常见做法：

- 本地 agent 默认不碰生产凭据。
- 高风险任务放到 cloud agent、Codespaces、容器或临时 checkout。
- 外部网络、secret、部署、删除、支付、邮件等动作需要额外确认。
- 自动运行任务要限制工具权限。
- PR、CI、review、audit log 保留痕迹。
- 对不可信网页、issue、日志保持 prompt injection 警觉。

Simon Willison 提到，YOLO mode 只适合确认恶意指令不可能混入上下文的低风险任务。Anthropic 官方也强调权限、hooks、auto mode、sandboxing、checkpoint 等机制。

你的个人项目可以先用一个简单分层：

| 风险层级 | 可以给 agent 做什么 | 需要限制什么 |
|---|---|---|
| 低风险 | 文档、测试、局部重构、样板代码 | 完成前跑验证 |
| 中风险 | 改业务逻辑、跨文件功能 | 先计划、先测试、人工 review |
| 高风险 | 删除、迁移、凭据、部署、外部 API 写操作 | 先确认，最好隔离环境 |
| 不允许默认执行 | 生产数据、支付、批量删除、泄露 secret | 必须人工控制 |

## 共识九：把重复流程沉淀成 Skills、Commands 或 Scripts

Jesse Vincent 的 Superpowers、Anthropic 的 skills / hooks / commands、Peter 的 slash commands、以及你当前的 `scripts/bootstrap` / `scripts/check` 思路都在说明同一件事：

```text
重复 3 次以上的 agent 协作方式，不应该继续靠手打 prompt。
```

可以沉淀的东西有三类：

### 1. 确定性命令

例如：

```text
scripts/bootstrap
scripts/dev
scripts/test
scripts/check
scripts/lint
scripts/format
```

这些不需要模型自由发挥，写成脚本最稳。

### 2. 项目事实

例如：

```text
docs/development.md
docs/testing.md
docs/architecture/overview.md
docs/troubleshooting.md
docs/project-status.md
```

这些是上下文资产，方便新会话接手。

### 3. Agent 工作流

例如：

```text
fix-bug：复现 -> 回归测试 -> 根因 -> 修复 -> 验证
new-feature：spec -> plan -> TDD -> review -> check
review-diff：按计划审查 diff，只报 correctness gap
research-options：不改代码，只调研 2-3 个方案
```

这些可以变成 skills、slash commands 或固定 prompt 模板。

## 社区和高手的常见组合

把官方建议、个人博客和社区讨论放在一起看，成熟用户不是只用一个 prompt，而是组合多层机制。

### 组合一：`AGENTS.md` / `CLAUDE.md` 做入口，不做百科

常见写法：

```text
项目目标 + 目录地图 + 常用命令 + 禁止事项 + 完成标准
```

高手会避免把所有风格偏好、框架知识、长 checklist 都塞进去。原因很简单：入口文件会被频繁加载，越长越容易稀释真正硬约束。

你的做法可以继续沿用：

```text
AGENTS.md = 入口地图和硬边界
docs/ = 长期上下文
playbooks/ = 可选择的工作流
skills/ = 运行时可发现能力
scripts/ = 确定性验证命令
```

### 组合二：Plan mode / ask mode 先让 agent 探路

Claude Code、Codex、Denny Britz、Jesse Vincent、Sketch 的实践都有一个共同点：非平凡任务先让 agent 读代码、列方案、写计划，不直接改。

典型句式：

```text
先不要修改文件。请只读审计当前实现，列出相关文件、风险、最小方案和验证命令。
```

这对应你 playbooks 里的 `scout`、`existing-project-audit`、`solution-comparison`。

### 组合三：Scripts / CI 做硬验证

社区里很多 hooks 和 commands 的真实用途很朴素：编辑后跑 formatter、lint、typecheck、test；提交前跑 check；阻止危险命令。

这说明：

```text
不要把“记得验证”只写成自然语言提醒。
能脚本化的验证，应沉淀成 scripts/check、CI 或 hook。
```

### 组合四：Skill / command 做重复流程

当一个流程经常重复，例如 bugfix、review、release note、migration、docs update，高手会把它变成：

- Claude slash command。
- Codex / Claude skill。
- repo-local playbook。
- script + prompt 的组合。

判断标准很实用：

```text
如果你第三次复制同一段 prompt，就该考虑把它升级成 playbook / command / skill。
```

### 组合五：Subagent / parallel agent 只用于隔离任务

Anthropic 官方、Simon Willison、Jesse Vincent 和 Denny Britz 的共识是：并行有用，但 review 是瓶颈。

比较稳的并行方式：

- 一个 agent 做 research，另一个等人确认后实现。
- 一个 agent 做实现，另一个只 review diff。
- 不同方案放不同 worktree。
- 低风险维护任务并行，例如补文档、修 lint、清 warnings。

高风险方式：

- 多个 agent 在同一目录同时改公共接口。
- 没有测试保护的并行重构。
- 人还没看 diff 就让另一个 agent 继续堆改动。

### 组合六：Hooks 用来约束生命周期，不替代判断

社区讨论里，hooks 的常见价值不是“智能”，而是自动执行固定检查：

- 写文件后跑 formatter / lint。
- tool call 前阻止危险命令。
- stop 前要求检查 diff。
- session start 时提醒读取入口文档。

所以 hooks 更像 guardrail，不是 workflow 本身。适合放“每次都必须发生”的规则；不适合放需要语义判断的大流程。

## 分歧：计划派 vs 直接对话派

这批材料里最大的分歧是：到底要不要每次写计划文件？

### 计划派

代表：Addy Osmani、Jesse Vincent、Anthropic 官方、Superpowers。

适合：

- 需求复杂。
- 项目不熟。
- 多人协作。
- 需要长期维护。
- 有安全、权限、数据、兼容性风险。
- 你希望新会话可以接手。

优点：

- 容易 review。
- 容易恢复上下文。
- 容易 TDD。
- 容易控制范围。

代价：

- 前置成本高。
- 简单任务可能显得重。
- 如果 plan 写得差，agent 会高效执行错误计划。

### 直接对话派

代表：Peter Steinberger 的当前 Codex CLI 工作流。

适合：

- 你非常熟悉项目。
- 任务 blast radius 小。
- 模型和 harness 足够会读代码。
- 你能实时观察、打断、修正。
- 你愿意接受更多探索性迭代。

优点：

- 快。
- 流畅。
- 适合 UI 迭代和创意探索。
- 不会被流程仪式拖慢。

代价：

- 更依赖人的直觉。
- 更难复盘。
- 对新项目、新人、新会话不友好。
- 并行时更容易制造 review 债。

### 对你的建议

你的目标是长期积累项目约定、使用渐进式 docs、坚持 TDD，所以更适合“轻量计划派”：

```text
小任务：短计划，直接做，完成前验证。
中任务：写 docs/plans 下的计划，按 TDD 做。
大任务：先 spec，再 plan，再分批执行。
探索任务：先 research，不落地代码。
```

这样不会变成流程表演，也不会回到“agent 随便写”的状态。

## 高手工作流模板

下面这些模板可以直接放进你的个人项目约定或 future skills。

### 模板一：探索一个陌生代码区

适用：你想了解项目某块逻辑，但不想污染实现上下文。

```text
请只做代码调研，不要修改文件。

问题：

请完成：
1. 搜索相关入口、类型、函数和测试。
2. 画出调用链或数据流。
3. 标出关键文件和行号。
4. 总结当前实现的隐含约束。
5. 说明如果后续要改，最可能影响哪些地方。

输出要短，优先给我能继续工作的上下文。
```

这个对应 Simon Willison 说的 research / “how does that work again” 场景。

### 模板二：让 Agent 先当 Scout

适用：任务难、代码区不熟、不确定会踩什么坑。

```text
请先当 scout，不要保留任何代码改动。

任务：

你可以搜索代码、尝试方案、运行测试，但最终只输出：
1. 你认为需要改哪些文件。
2. 你发现的主要风险。
3. 最小可行方案。
4. 你建议先写哪些测试。
5. 哪些尝试失败了，为什么。

不要提交最终实现。
```

这个做法的价值是：让 agent 替你先踩坑，但不把试验代码直接带进主线。

### 模板三：新功能 TDD

适用：中等复杂度功能。

```text
我要做一个新功能，请遵守 TDD。

功能：
背景：
验收标准：
暂时不做：

请先读取 AGENTS.md、README.md、docs/README.md、docs/testing.md 和相关代码。
然后先写计划，不要改实现代码。

计划必须包含：
1. 目标和非目标。
2. 影响文件。
3. 先写哪些失败测试。
4. 最小实现步骤。
5. 验证命令。
6. 风险和回滚方式。

等我确认计划后，再按 Red-Green-Refactor 执行。
```

这个就是你 [`../../agent/playbooks/workflows/new-project-bootstrap.md`](../../agent/playbooks/workflows/new-project-bootstrap.md) 的核心实战入口。

### 模板四：Bug 修复

适用：所有非琐碎 bug。

```text
我要修一个 bug。请不要猜测式修改。

现象：
复现步骤：
期望：
实际：
相关日志：

请按顺序执行：
1. 找到最小复现路径。
2. 解释根因假设，并说明需要验证什么。
3. 写一个失败的回归测试，确认它因为这个 bug 失败。
4. 写最小修复。
5. 确认回归测试通过。
6. 跑项目 check。
7. 把失败模式写入 docs/troubleshooting.md。
```

这对应 Superpowers 的 systematic debugging + TDD + verification-before-completion。

### 模板五：Review Agent

适用：实现后让新上下文审查。

```text
请作为 reviewer 审查当前 diff。

审查依据：
1. 原始需求 / plan：
2. 验收标准：
3. 项目约定：

只报告这些问题：
1. 正确性问题。
2. 需求遗漏。
3. 测试缺口。
4. 兼容性或安全风险。
5. 明显超出范围的改动。

不要报告纯风格偏好。
每个发现都要给文件和行号，并说明为什么影响结果。
```

这个对应 Anthropic 官方的 adversarial review / writer-reviewer 模式，也符合 GitHub 把 agent 产物放进 PR review 的思路。

### 模板六：并行方案比较

适用：你不确定架构方向。

```text
请提出 3 个实现方案，不要改代码。

问题：

每个方案请说明：
1. 核心思路。
2. 会改哪些文件。
3. 优点。
4. 风险。
5. 如何验证。
6. 适合一次性实现还是分阶段实现。

最后给出你的推荐，但要明确推荐成立的前提。
```

如果要并行，可以让不同 agent 各自回答同一个问题，再由你综合。

## 对你当前工作流的直接启发

你现在的工作流是：

```text
AGENTS.md 写个人约定
docs/ 做渐进式上下文
TDD 做开发
新项目先配置 harness
```

这和高手经验高度一致，但可以再补强五点。

### 1. 明确 review 是流程的一等公民

你现在的流程已经有 `scripts/check` 和 TDD，但还可以更明确：

```text
实现完成 != 任务完成。
实现完成 + 测试通过 + diff review + 状态文档更新 = 任务完成。
```

建议在新项目的 `docs/testing.md` 或 `docs/development.md` 增加：

- reviewer prompt。
- 什么问题必须修。
- 什么问题算 optional。
- 什么时候需要新会话 review。

### 2. 给并行 agent 加 gate

不要默认并行。可以写成：

```text
允许并行：
- research
- POC
- 低风险维护
- 独立 worktree

不允许默认并行：
- 同一目录多个大功能
- 多个 agent 同时改公共接口
- 没有测试保护的大重构
```

这能吸收 Simon / Jesse 的好处，同时避开 Denny 提醒的 review 债。

### 3. 把“Scout 模式”加入流程

很多时候你不是要 agent 直接做，而是要它先帮你探路。建议增加一个固定入口：

```text
先当 scout，不要保留改动。
```

这对陌生代码、复杂 bug、方案调研尤其有用。

### 4. 把 blast radius 写进计划模板

每个计划都要求 agent 回答：

```text
这次改动预计影响几个文件？
是否会改公共接口？
是否需要迁移？
是否影响权限、数据、网络、部署？
如何回滚？
```

这样可以把 Peter 的高强度经验转成更稳的个人流程。

### 5. 给 `AGENTS.md` 做减法

Augment 的实验提醒我们：规则有用，但不是越多越好。你的 `AGENTS.md` 应该保留：

- 项目是什么。
- 目录入口。
- 必须读哪些文档。
- 必须跑哪些命令。
- 硬边界和禁止事项。

其他细节进入 `docs/`、scripts 或 skills。

## 反模式清单

| 反模式 | 为什么危险 | 替代做法 |
|---|---|---|
| “帮我做这个功能”然后直接开写 | 需求、范围、验收都不清楚 | 先 spec / plan |
| 一次让 agent 改很大一坨 | 难 review，难回滚 | 拆成 2-5 个小任务 |
| 没有失败测试就修 bug | 很容易修错原因 | 先复现，再回归测试 |
| 只看 agent 总结，不看 diff | agent 可能遗漏关键副作用 | 人类审查 diff |
| 把所有规则塞进 `AGENTS.md` | 上下文噪声高，规则会互相稀释 | 入口在 `AGENTS.md`，细节进 docs |
| 并行多个大改 | review 债爆炸 | research 并行，代码隔离 |
| reviewer 反馈直接全让 agent 改 | reviewer 可能误判 | 先让 agent 评估哪些反馈该采纳 |
| 默认 YOLO 跑外部命令 | 可能泄露、删除、写错外部系统 | 分层权限和隔离环境 |
| 过度迷信 benchmark / 模型排名 | 真实效果受 harness 和上下文影响 | 用自己的任务集评估 |
| 长会话里混杂很多任务 | 上下文污染 | 任务结束 `/clear` 或新会话 |

## 一页纸实践清单

每次使用 coding agent 前：

```text
1. 我是否说清了目标和非目标？
2. 我是否给了相关上下文入口，而不是一股脑贴材料？
3. 这个任务是否需要先写计划？
4. 是否需要 scout 先探路？
5. 是否能先写失败测试？
6. 预计 blast radius 多大？
7. 完成标准是什么命令或证据？
8. 是否需要新会话 review？
9. 是否有权限、secret、外网、删除、部署风险？
10. 这次失败后，应该回流到文档、测试、脚本还是 AGENTS.md？
```

对个人项目，最推荐的默认循环是：

```text
读 AGENTS.md / docs 入口
-> 澄清目标和非目标
-> 写短 plan
-> 先写失败测试
-> 最小实现
-> 局部测试
-> 重构
-> scripts/check
-> 新上下文 review 关键 diff
-> 更新 docs/project-status.md / troubleshooting
```

## 最后判断

确实有很多编程高手在分享如何用 coding agent 编程，但他们的共同点不是“发现了某个万能 prompt”，而是把 agent 放回软件工程的基本纪律里。

如果要用一句话指导你的实践：

> 把 agent 当成速度很快、上下文有限、需要测试和 review 约束的协作者。你越能把目标、上下文、测试和完成标准工程化，它越像高手；你越让它自由发挥，它越像一个手速极快但会跑偏的新人。

## 参考资料

访问日期均为 2026-05-31。

| 标题 | 作者或机构 | 发布日期 | 类型 | 链接 |
|---|---|---:|---|---|
| How OpenAI uses Codex | OpenAI | 2026，PDF 未标注精确日期 | technical report | https://cdn.openai.com/pdf/6a2631dc-783e-479b-b1a4-af0cfbd38630/how-openai-uses-codex.pdf |
| Codex best practices | OpenAI | 未标注 | docs | https://developers.openai.com/codex/learn/best-practices |
| Thoughts on coding agents | Denny Britz | 2026-02-22 | blog | https://dennybritz.com/posts/coding-agents |
| My LLM coding workflow going into 2026 | Addy Osmani | 2025-12-28 | blog | https://addyosmani.com/blog/ai-coding-workflow |
| Embracing the parallel coding agent lifestyle | Simon Willison | 2025-10-05 | blog | https://simonwillison.net/2025/Oct/5/parallel-coding-agents |
| How I'm using coding agents in September, 2025 | Jesse Vincent | 2025-10-05 | blog | https://blog.fsck.com/2025/10/05/how-im-using-coding-agents-in-september-2025 |
| Superpowers: How I'm using coding agents in October 2025 | Jesse Vincent | 2025-10-09 | blog | https://blog.fsck.com/2025/10/09/superpowers |
| Just Talk To It - the no-bs Way of Agentic Engineering | Peter Steinberger | 2026-03-14 | blog | https://steipete.me/posts/just-talk-to-it |
| The 7 Prompting Habits of Highly Effective Engineers | Josh Bleecher Snyder / Sketch | 2025-05-20 | blog | https://sketch.dev/blog/seven-prompting-habits |
| Claude Code best practices | Anthropic | 2025-04-18 | docs / blog | https://www.anthropic.com/engineering/claude-code-best-practices |
| Claude Code power user tips | Anthropic | 未标注 | docs | https://support.claude.com/en/articles/14554000-claude-code-power-user-tips |
| How and when to use subagents in Claude Code | Anthropic | 未标注 | blog | https://claude.com/blog/how-and-when-to-use-subagents-in-claude-code |
| Pick your agent: Use Claude and Codex on Agent HQ | GitHub Blog | 2026-02-04 | blog | https://github.blog/news-insights/company-news/pick-your-agent-use-claude-and-codex-on-agent-hq |
| Karpathy skills on OpenClaw: agents don't write better code. But they do it more efficiently. | Augment Code | 未标注 | technical report / blog | https://www.augmentcode.com/blog/karpathy-skills-on-openclaw-agents-don-t-write-better-code-but-they-do-it-more-efficiently |
| Claude.md, rules, hooks, agents, commands, skills... | r/ClaudeCode 社区 | 2026-05-09 | community | https://www.reddit.com/r/ClaudeCode/comments/1pxou18/claudemd_rules_hooks_agents_commands_skills/ |
| Do you actually use hooks in Claude Code? | r/ClaudeCode 社区 | 2026-04-17 | community | https://www.reddit.com/r/ClaudeCode/comments/1tkvg6t/do_you_actually_use_hooks_in_claude_code/ |
