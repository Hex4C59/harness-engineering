# Vibe Coding 的 Review 困境与学习债

调研日期：2026-05-30

## 这篇文档回答什么

这篇文档围绕一个非常具体的困境：

```text
我原本只熟悉 Python。
现在有了 coding agent，我可以让 AI 帮我写更难的 Rust 项目。
但我不懂 Rust，所以我无法真正 review 代码。
我只能检查 AI 描述的实现过程是否符合我的想法，而这个描述本身也可能是假的。
项目也许能跑，但我没有真正学会 Rust，也没有学会用 Rust 开发项目。
```

这不是个小问题。它暴露了 Vibe Coding 的核心张力：

> AI 降低了“产出代码”的门槛，但没有自动降低“判断代码是否正确、可维护、安全、符合语言惯用法”的门槛。相反，当人类进入不熟悉领域时，review 责任、学习债和维护风险会被推迟，而不是消失。

本文把这个问题拆成三层：

1. **Review 困境**：你无法判断 AI 生成的 Rust 是否真的好，只能判断它讲述的故事是否顺耳。
2. **学习债**：你得到一个可运行项目，但没有获得足够可迁移的语言模型、调试经验和工程判断。
3. **责任错位**：AI 可以生成和解释，但最终维护、部署、事故、数据和用户后果仍然落在人身上。

一句话结论：

```text
Vibe Coding 可以跨过陌生语言的起步门槛，
但不能跳过 review、debug、测试、维护和学习。
如果没有专门设计的 harness，它更像“外包实现”，不是“掌握技能”。
```

## 来源说明

本文使用论文、技术报告、官方文档、工程博客、社区讨论和工程事故材料。涉及模型、产品和调查数据的内容以 2026-05-30 可查公开资料为准。

| 来源 | 类型 | 主要价值 |
|---|---|---|
| [The Impact of AI on Developer Productivity: Evidence from GitHub Copilot](https://arxiv.org/abs/2302.06590) | 实验 / 论文 | 受控任务中 Copilot 组完成速度更快，说明代码生成能降低某些实现成本 |
| [Measuring the Impact of Early-2025 AI on Experienced Open-Source Developer Productivity](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/) | RCT / 技术报告 | 早 2025 AI 工具让熟悉大型仓库的资深开发者在研究任务中平均变慢，提醒不能把 demo 等同生产力 |
| [Stack Overflow Developer Survey 2025: AI](https://survey.stackoverflow.co/2025/ai) | 开发者调查 | 开发者广泛使用 AI，但信任、准确性和调试 AI 生成代码仍是主要顾虑 |
| [DORA 2025 State of AI-assisted Software Development](https://dora.dev/research/2025/dora-report/) | 行业报告 | AI 是组织能力放大器，效果依赖软件交付系统、平台和文化 |
| [How Novices Use LLM-Based Code Generators to Solve CS1 Coding Tasks](https://arxiv.org/abs/2309.14049) | 编程教育研究 | 初学者能用 LLM 完成任务，但 interaction pattern 和学习效果并不等同于真正掌握 |
| [Do Users Write More Insecure Code with AI Assistants?](https://arxiv.org/abs/2211.03622) | 安全研究 | AI 助手可能让用户产出更多不安全代码，且会影响用户对安全性的信心判断 |
| [AI-Assisted Programming Decreases the Productivity of Experienced Developers by Increasing the Technical Debt and Maintenance Burden](https://arxiv.org/abs/2510.10165) | 实证研究 / 争议材料 | 从代码库历史指标观察 AI 后技术债、维护负担和重复倾向，适合作为风险信号而非最终定论 |
| [GitHub Copilot code review responsible use](https://docs.github.com/en/copilot/responsible-use/code-review) | 官方文档 | 明确 AI code review 是补充，不应替代人类 review，并列出漏报、误报和错误建议风险 |
| [OpenAI Codex best practices](https://developers.openai.com/codex/learn/best-practices) | 官方文档 | 强调测试、检查、diff review、`AGENTS.md`、明确“good”是什么 |
| [Anthropic Claude Code best practices](https://www.anthropic.com/engineering/claude-code-best-practices) | 官方工程博客 | 强调先探索、计划、小步、测试、工具权限和人工判断 |
| [The Rust Programming Language: Ownership](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html) | 官方文档 | Rust 核心概念是 ownership、borrowing、slice 等，不能只靠语法表层理解 |
| [The Rustonomicon](https://doc.rust-lang.org/nomicon/) | 官方文档 | 说明 unsafe Rust 的风险边界和需要人工理解的不变量 |
| [Ironies of Automation](https://doi.org/10.1016/0005-1098%2883%2990046-8) | 人因工程经典论文 | 自动化会把人从操作中移开，但在异常时仍要求人接管，形成技能退化和责任错位 |
| [Karpathy 的 `vibe coding` 公开表述及社区整理](https://simonwillison.net/2025/Mar/19/vibe-coding/) | 社区概念源头 | “少看代码、靠感觉推进”的说法准确命名了这种体验，但也暴露了 review 缺口 |
| Replit AI agent 删除生产数据库事件 | 工程事故 / 社区案例 | 展示 agent 能执行危险动作时，计划、权限、环境隔离和恢复机制不是可选项 |

## 事实、观点和推断

本文刻意区分三类说法。

| 类型 | 本文用法 |
|---|---|
| 事实 | 论文、官方文档、调查、产品文档或事故报道中可以直接确认的机制、结果或限制 |
| 观点 | 研究者、工程师、厂商和社区对事实的解释，例如“AI review 不应替代人类 review” |
| 推断 | 本文基于多个来源整理出的判断框架，例如“Vibe Coding 会制造学习债” |

一个重要前提：

```text
能跑不是事实终点，只是验证起点。
```

在陌生领域里，`cargo test` 通过、demo 可运行、AI 解释流畅，都不能直接证明代码是 idiomatic Rust、边界安全、维护成本合理、长期架构正确。

## 概念边界：Vibe Coding 到底是什么

Karpathy 在 2025 年把 `vibe coding` 作为一个社区说法带火，大意是：开发者几乎不看代码，更多是描述意图、运行、看结果、继续让 AI 修改。这个词有点玩笑感，但命名很准确：它把人的工作从“逐行构造代码”转成“用感觉和反馈驾驭生成过程”。

这里不把 Vibe Coding 当贬义词。它至少有三个真实价值：

- 快速原型。
- 降低陌生 API / 框架 / 语言的启动成本。
- 让非专家把想法推进到可运行状态。

问题在于，Vibe Coding 容易把三件事混在一起：

| 看起来像 | 实际上可能只是 |
|---|---|
| 我会写 Rust 了 | 我会让 AI 生成 Rust 代码 |
| 我理解这个项目了 | 我理解 AI 对项目的解释 |
| 代码通过测试了 | 当前测试没有暴露问题 |
| AI review 说没问题 | 另一个模型没有在当前上下文里发现问题 |
| 项目能跑 | 项目在 happy path 上能跑 |

这个区别很残酷，但很重要。

## 事实一：AI 确实能降低实现成本，但收益不是稳定常数

GitHub Copilot 早期研究在受控 JavaScript HTTP server 任务中发现，使用 Copilot 的开发者完成任务明显更快。这个结果说明 AI 能降低某些编码任务的实现成本，尤其是任务边界清楚、技术栈常见、反馈简单的时候。

但 METR 在 2025 年 7 月发布的随机对照研究中观察到另一面：在熟悉大型开源仓库的资深开发者、真实 issue 和早 2025 AI 工具组合下，允许使用 AI 的开发者平均变慢。这个研究不能推出“AI 一定降低生产力”，因为工具变化很快、样本也有范围限制；但它足以反驳“AI 写代码必然提速”的简单叙事。

DORA 2025 的组织视角更稳定：AI 是能力放大器。它会放大已有工程系统的优点，也会放大混乱流程、低质量测试、薄弱 review 和不清楚需求。

**事实结论：**

```text
AI 对实现速度有真实帮助，但收益依赖任务、代码库、测试、review、工具熟练度和组织系统。
```

## 事实二：开发者已经在用 AI，但并没有完全信任它

Stack Overflow 2025 调查显示，AI 工具使用已经很普遍，但开发者对准确性和信任仍然有明显保留。社区里常见抱怨是：AI 生成的代码“差一点点正确”，调试这种差一点点的代码反而耗时。

这和你描述的困境一致：

```text
你可以判断项目是否看起来符合你的想法，
但你未必能判断实现是否真的正确。
```

当你不懂 Rust 时，AI 输出中最危险的不是语法错误。语法错误通常会被编译器抓住。更危险的是：

- 抽象边界不符合 Rust 惯用法。
- 生命周期、所有权和 clone 策略导致隐藏性能或设计问题。
- 错误处理表面上用了 `anyhow` / `thiserror`，但错误语义不清。
- 异步、并发、锁、channel 使用能编译但容易卡死或泄漏任务。
- `unsafe` 被包装成“性能优化”或“FFI 必需”，但真实不变量没人能审。
- 测试只覆盖 happy path，边界和失败路径没被验证。

**事实结论：**

```text
AI 生成代码的主要风险经常不是“完全不能跑”，而是“看起来合理，但非专家无法识别其长期风险”。
```

## 事实三：AI code review 被官方定位为辅助，而不是替代

GitHub Copilot code review 的 responsible use 文档明确把 AI review 定位为补充，而不是人类 review 的替代。官方文档也提醒，它可能漏报、误报，或者给出不准确、不安全的建议。

OpenAI Codex best practices 也没有说“让 Codex 写完就结束”。相反，它建议让 Codex 写或更新测试、运行相关检查、确认结果、review diff，并通过 `AGENTS.md` 或 prompt 告诉 Codex 什么叫 “good”。Codex 文档还强调：如果团队有 `code_review.md` 并从 `AGENTS.md` 引用，Codex 可以按照这些规则做 review。

这说明厂商自己的最佳实践已经承认：

```text
可靠性来自测试、规则、上下文、review、反馈和工程约束，
不是来自模型的一句“我检查过了”。
```

**事实结论：**

AI review 有价值，但不能解决“人类完全不懂目标领域却要承担最终判断”的问题。

## 事实四：初学者使用代码生成器，不等于自动学会编程

编程教育研究里，LLM code generator 的效果通常不是单向结论。`How Novices Use LLM-Based Code Generators to Solve CS1 Coding Tasks` 这类研究说明，初学者可以借助 LLM 完成更多任务，但他们如何提问、如何修改、是否阅读解释、是否调试、是否自己重写，会显著影响学习质量。

这个结论可以迁移到“Python 用户用 AI 写 Rust”：

| 使用方式 | 更可能得到 |
|---|---|
| 只描述需求，让 AI 写完整项目 | 一个可运行 artifact，但学习浅 |
| 让 AI 解释每个设计选择，并要求自己复述 | 更容易形成概念模型 |
| 先写测试和接口，再让 AI 实现 | 更容易学会边界和验收 |
| 遇到编译错误直接让 AI 修 | 错误消失，但你未必理解 borrow checker |
| 把每次修复总结进笔记和规则 | 学习会积累成可复用知识 |

**事实结论：**

AI 可以成为 tutor，也可以成为代工。差别不在模型，而在工作流。

## 事实五：安全研究说明“自信”可能比错误更危险

`Do Users Write More Insecure Code with AI Assistants?` 的安全研究显示，使用 AI 助手的参与者可能写出更多不安全代码，而且对自己代码安全性的判断会受到影响。不同研究设计、任务和工具会影响结论，但这个方向对 Vibe Coding 很关键：

```text
AI 不只是生成代码，它还生成一种“这大概没问题”的心理感受。
```

当用户不懂 Rust、数据库、加密、认证、并发或部署时，这种感受尤其危险。因为用户没有足够的内部模型去反驳 AI 的解释。

Replit AI agent 删除生产数据库事件也说明：一旦 agent 能执行真实环境动作，风险就不只是代码质量，而是权限、环境隔离、恢复、审批和审计。

**事实结论：**

Vibe Coding 在高风险领域必须有外部护栏。人的信心不能替代权限边界和可恢复系统。

## Rust 例子：为什么“不懂语言就无法 review”尤其真实

Rust 的难点不主要是语法，而是语义模型。Rust 官方书把 ownership 作为核心章节，因为它决定内存管理、借用、生命周期、可变性和数据竞争防护。Rust 的很多价值来自编译期约束，但这些约束也要求开发者理解代码为什么能编译、为什么不能编译。

一个 Python 开发者让 AI 写 Rust，常见错位是：

| Python 直觉 | Rust 里需要额外理解 |
|---|---|
| 对象可以到处传引用 | ownership、borrow、lifetime |
| 复制数据通常不是第一关注点 | `clone`、move、copy 的成本和语义 |
| 异常冒泡很自然 | `Result`、错误类型、`?`、错误上下文 |
| 动态结构灵活 | trait、generic、enum、pattern matching |
| 并发靠运行时约定 | `Send`、`Sync`、锁、channel、async runtime |
| 性能问题后面再看 | 分配、借用、zero-copy、迭代器、monomorphization |
| 库封装了底层危险 | `unsafe` 不变量、FFI、别名、生命周期延长 |

所以 Rust 的 review 不是问：

```text
这段代码有没有语法错误？
```

而是问：

```text
这个 ownership 边界是不是自然？
这里 clone 是必要的还是为了绕过 borrow checker？
错误类型是否表达了调用方需要处理的语义？
trait 抽象是否过早？
async task 是否会泄漏？
unsafe 块的不变量是否被完整说明和测试？
```

如果你不懂 Rust，这些问题很难靠“读 AI 的总结”解决。因为总结本身也需要 review。

## 观点层：Vibe Coding 的真正瓶颈不是写，而是判断

综合论文、官方文档、工程博客和社区经验，可以把当前共识整理成一句话：

```text
生成代码越来越便宜，判断代码越来越稀缺。
```

### 观点一：AI 可以替你打字，但不能替你拥有问题

AI 可以根据 prompt 实现功能，但它并不拥有你的真实目标、用户、约束、事故后果和维护责任。尤其是当你跨入陌生技术栈时，你和 AI 的关系很容易变成：

- 你负责愿望。
- AI 负责代码。
- 测试负责碰巧覆盖的部分。
- 未来的你负责还债。

这个结构短期很爽，长期很危险。

### 观点二：不会 review 的代码，就是外包代码

如果你完全无法判断某段 Rust 代码的设计是否合理，那么这段代码对你来说更像外包交付物，而不是你自己的工程资产。

外包不是坏事，但外包需要验收机制：

- 规格。
- 测试。
- 代码审查。
- 安全检查。
- 运行观测。
- 文档。
- 维护合同。

Vibe Coding 的问题是，它常常拥有外包的风险，却没有外包的验收流程。

### 观点三：AI 解释会制造“理解错觉”

AI 很擅长把一段实现解释得顺滑。顺滑解释会降低人的警惕，让人觉得自己理解了。真正的理解应该能通过这些测试：

- 你能不用 AI 重新写一个最小版本吗？
- 你能解释为什么这里不能简单 clone / unwrap / static lifetime 吗？
- 你能预测改错一行后哪个测试会失败吗？
- 你能在没有 AI 的情况下定位一个编译错误吗？
- 你能向另一个人解释这个模块的 invariants 吗？

如果不能，说明你得到的是“可接受叙述”，不是“可迁移能力”。

### 观点四：学习不是自动发生的，必须被设计进 workflow

用 AI 写 Rust 项目时，学习是否发生，取决于你有没有把学习目标变成任务约束。比如：

```text
不要直接给最终代码。
先解释本 slice 涉及的 Rust 概念。
给一个 30 行以内的最小例子。
让我先预测 borrow checker 会不会接受。
再实现项目代码。
最后给我 3 个检查问题，我回答后你再继续。
```

这会显著变慢，但这是学习本来需要的成本。AI 可以降低摩擦，不能取消认知负荷。

## 推断：Vibe Coding 会产生三种债

### 1. Review 债

你今天接受了自己看不懂的 diff，未来就要在 bug、重构、安全审计或性能问题里偿还。

Review 债的典型症状：

- diff 很大，但你只能看功能描述。
- 测试通过，但你不知道测试是否覆盖关键语义。
- AI 说“这是 idiomatic Rust”，但没有引用官方文档、常见 crate 约定或项目规范。
- 出问题时你只能继续问 AI，而不能独立缩小问题范围。

### 2. 学习债

你完成了项目，但没有形成可复用技能。下一次遇到类似问题，仍然只能让 AI 从头生成。

学习债的典型症状：

- 看懂最终代码困难。
- 离开 AI 无法写出同类模块。
- 不知道为什么编译器报错。
- 不知道怎么读 crate 文档。
- 不知道 Rust 项目如何组织测试、feature、workspace、release。

### 3. 维护债

项目跑起来后，后续修改比预期更难。因为架构不是你内化过的架构，而是模型在某个上下文里生成的结果。

维护债的典型症状：

- 新需求一来就牵一发动全身。
- 抽象层次和命名不符合你的思维模型。
- 错误处理和日志无法支持排查。
- 依赖选型没人能解释 trade-off。
- 版本升级、性能优化、安全修复都要重新请 AI 猜。

## 什么时候 Vibe Coding 是合理的

不是所有 Vibe Coding 都糟糕。合理场景包括：

| 场景 | 可接受原因 | 必要护栏 |
|---|---|---|
| 一次性脚本 | 生命周期短，风险低 | 限权、备份、dry-run |
| 原型验证 | 目标是学习需求，不是长期维护 | 明确 throwaway，不直接生产化 |
| 熟悉领域里的陌生 API | 你能 review 业务和架构 | 小 diff、官方文档、测试 |
| 低风险内部工具 | 影响范围有限 | 权限隔离、日志、回滚 |
| 学习项目 | 目标是形成理解 | tutor workflow、复述、重写、练习 |

不合理场景包括：

| 场景 | 风险 |
|---|---|
| 生产数据库、支付、权限、认证 | 错误后果高，不能只靠 AI 解释 |
| 加密、安全、unsafe Rust、FFI | 专业不变量难被非专家 review |
| 大规模重构 | diff 太大，review 债暴涨 |
| 长期维护的核心项目 | 学习债会变成维护债 |
| 没有测试和回滚的自动执行 | 事故不可控 |

## 如何把 Vibe Coding 改造成可学习工作流

如果目标是“借 AI 学 Rust 并做项目”，不要让 agent 直接冲到最终实现。更好的流程是：

```text
Concept -> Tiny Example -> Test -> Implementation Slice -> Review -> Reflection
```

### Step 1：先声明学习目标，不只声明功能目标

差的 prompt：

```text
用 Rust 写一个文件搜索 CLI。
```

更好的 prompt：

```text
用 Rust 写一个文件搜索 CLI，但我的目标是学习 Rust。
每一步只做一个小 slice。
每个 slice 先解释涉及的 Rust 概念，再给最小例子，再写测试，再实现。
不要一次生成超过 150 行 diff。
完成后问我 3 个 review 问题，确认我理解后再继续。
```

### Step 2：要求 AI 写“可反驳”的设计说明

不要只要“实现过程总结”。要让 AI 明确列出：

- 选择了哪些 crate，为什么。
- 没选择哪些方案，为什么。
- ownership 边界在哪里。
- 哪些地方用了 clone，为什么合理。
- 错误类型如何设计。
- 哪些行为由测试覆盖。
- 哪些风险没有覆盖。

### Step 3：把 diff 控制在能 review 的大小

对陌生语言，diff 要比熟悉语言更小。建议：

| 熟悉度 | 单次 diff 上限 |
|---|---:|
| 熟悉语言、熟悉项目 | 200-400 行 |
| 熟悉语言、陌生项目 | 100-200 行 |
| 陌生语言、熟悉领域 | 50-150 行 |
| 陌生语言、陌生领域 | 20-80 行 |

这不是形式主义。diff 越小，你越有机会真正读懂。

### Step 4：让 AI 生成测试，但你来解释测试

每个测试都问：

```text
如果实现错了，这个测试会不会失败？
它覆盖的是语法、happy path，还是关键语义？
还有哪个边界没有测？
```

对 Rust 项目，至少检查：

- 单元测试。
- 集成测试。
- 错误路径测试。
- CLI snapshot / golden output。
- property-based test，适合 parser、路径规则、状态机。
- `cargo fmt`、`cargo clippy`、`cargo test`。
- 如果有 unsafe，要求说明 safety invariant，并加 Miri / sanitizer 等额外检查，视项目而定。

### Step 5：用“独立 reviewer agent”，但不要让它代替你

可以开新会话让 reviewer 只读审查：

```text
请作为 Rust reviewer 审查这个 diff。
只输出 P0/P1/P2 问题。
每个问题必须包含文件位置、失败场景、为什么 Rust 语义上有风险、如何验证。
不要给纯风格建议。
如果没有合并前必须修的问题，明确说可以先合并。
```

然后你要做的不是盲信 reviewer，而是把它的评论变成学习材料：

- 让它解释涉及的 Rust 概念。
- 让它给最小反例。
- 让它指出官方文档或 crate 文档依据。
- 让它给测试证明。

### Step 6：保留学习日志

每个 slice 结束时写 5 行：

```text
今天学到的 Rust 概念：
我原来的 Python 直觉：
Rust 里的不同点：
本项目里对应代码：
下次 review 要检查什么：
```

这会把一次性的 AI 对话转成你自己的长期知识。

## 如何判断自己是不是在真正学习

可以用一个简单量表：

| 问题 | 能做到说明 |
|---|---|
| 我能不用 AI 解释本模块的数据流吗？ | 有基本理解 |
| 我能指出哪些地方发生 move / borrow / clone 吗？ | 开始理解 Rust 语义 |
| 我能预测删除某个错误处理分支会破坏哪个测试吗？ | 测试和行为关联起来了 |
| 我能不用 AI 修一个小编译错误吗？ | 开始获得调试能力 |
| 我能把同类功能重新写一个最小版本吗？ | 有迁移能力 |
| 我能拒绝 AI 的一个不合理建议吗？ | 有 review 能力 |

如果这些都做不到，就诚实地把项目标记为：

```text
AI-assisted prototype, not yet personally maintainable.
```

这不是失败。它只是准确标注资产状态。

## 给个人开发者的操作清单

### 如果目标是快速做出东西

- 明确这是 prototype 还是 production。
- 禁止 agent 直接操作生产资源。
- 小步提交，每步可回滚。
- 至少有 smoke test 和关键路径测试。
- 让 AI 生成风险清单，但你只接受能被测试或文档支持的说法。

### 如果目标是学习 Rust

- 每个功能都先做 tiny example。
- 每个新语法都要求解释 “Python 直觉 vs Rust 语义”。
- 每个 slice 控制在 20-80 行可读 diff。
- 每次让 AI 修 borrow checker 前，先让自己猜错误原因。
- 每天保留一页学习日志。
- 每周不用 AI 重写一个小模块。

### 如果目标是长期维护项目

- 写 `AGENTS.md`，明确 Rust 版本、crate 选型、测试命令、clippy 策略、unsafe 政策。
- 写 `docs/development.md`，记录架构、错误处理、模块边界。
- 高风险模块必须找懂 Rust 的人或独立 reviewer 深审。
- CI 至少跑 `cargo fmt --check`、`cargo clippy --all-targets -- -D warnings`、`cargo test`。
- 对 parser、状态机、文件系统操作、并发逻辑增加针对性测试。
- 不接受自己完全看不懂的核心抽象。

## 可直接复制的 prompt

### 学习优先

```text
我想用 Rust 实现这个功能，但我的目标是学习，不是只拿到代码。

请按以下流程：
1. 先说明这个 slice 涉及的 Rust 概念。
2. 给一个 30 行以内的最小例子。
3. 让我预测关键 ownership / borrowing 行为。
4. 再写项目代码。
5. 写测试并解释每个测试能抓住什么 bug。
6. 最后问我 3 个 review 问题。

限制：
- 单次 diff 不超过 80 行。
- 不要引入 unsafe。
- 不要用 clone 绕过 borrow checker，除非解释成本和理由。
- 如果有多个设计方案，先列 trade-off，不要直接实现。
```

### Review 优先

```text
请作为克制的 Rust reviewer 审查当前 diff。

只报告 P0/P1/P2：
- P0：正确性、安全、数据丢失、死锁、未定义行为、生产事故风险。
- P1：明显可维护性、错误处理、测试缺口、Rust 语义误用。
- P2：值得以后改善，但不阻塞合并。

每条必须包含：
- 文件和行号。
- 失败场景。
- 为什么这是 Rust 语义或工程风险。
- 如何用测试、clippy、Miri 或手工步骤验证。

不要输出纯风格建议。
如果没有 P0/P1，请明确说“可以先合并，但保留以下 P2”。
```

### 防止解释幻觉

```text
请不要只解释“代码做了什么”。
请列出你解释中哪些是：
- 从代码直接可见的事实。
- 基于 Rust 规则的推断。
- 你不确定、需要我验证的假设。

对于每个关键判断，请给出验证方式。
```

## 和 harness engineering 的关系

这个问题最后会回到本仓库的主线：harness engineering。

Vibe Coding 的裸形态是：

```text
human intent -> model generates code -> human runs it -> feels ok
```

更可靠的形态应该是：

```text
learning goal + product goal
-> plan
-> tiny example
-> reviewable slice
-> tests
-> deterministic checks
-> independent review
-> human explanation
-> reflection notes
-> next slice
```

也就是说，真正有价值的不是“让 AI 写 Rust”，而是建立一个系统，让你：

- 能安全得到 AI 的生产力。
- 能看见 AI 的不确定性。
- 能把代码产物变成学习材料。
- 能逐步获得 review 能力。
- 能知道什么时候必须找人类专家。

## 最终判断

你的困境不是个人问题，而是 Vibe Coding 的结构性问题。

```text
AI 可以把陌生领域的产出门槛降得很低，
但 review 门槛、维护门槛和学习门槛不会自动消失。
```

如果目标只是原型，Vibe Coding 很有价值。如果目标是长期维护或真正学会 Rust，就必须改变工作流：让 AI 少一点“一次性交付完整答案”，多一点“分步教学、可验证实现、独立 review、学习日志和人类复述”。

最重要的判断标准不是“这个项目能不能跑”，而是：

```text
当它坏掉时，我能不能理解它为什么坏；
当需求变化时，我能不能安全地改；
当 AI 给出建议时，我能不能判断哪些该接受、哪些该拒绝。
```

如果答案还是否定的，那项目可以继续做，但应该被标注为 `AI-assisted prototype`，而不是 `personally maintainable software`。

## 参考资料

- GitHub Next / arXiv: [The Impact of AI on Developer Productivity: Evidence from GitHub Copilot](https://arxiv.org/abs/2302.06590)
- METR: [Measuring the Impact of Early-2025 AI on Experienced Open-Source Developer Productivity](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/)
- Stack Overflow: [2025 Developer Survey - AI](https://survey.stackoverflow.co/2025/ai)
- DORA: [2025 State of AI-assisted Software Development](https://dora.dev/research/2025/dora-report/)
- arXiv: [How Novices Use LLM-Based Code Generators to Solve CS1 Coding Tasks in a Self-Paced Learning Environment](https://arxiv.org/abs/2309.14049)
- arXiv: [Do Users Write More Insecure Code with AI Assistants?](https://arxiv.org/abs/2211.03622)
- arXiv: [AI-Assisted Programming Decreases the Productivity of Experienced Developers by Increasing the Technical Debt and Maintenance Burden](https://arxiv.org/abs/2510.10165)
- GitHub Docs: [Responsible use of GitHub Copilot code review](https://docs.github.com/en/copilot/responsible-use/code-review)
- OpenAI Docs: [Codex best practices](https://developers.openai.com/codex/learn/best-practices)
- Anthropic Engineering: [Claude Code best practices](https://www.anthropic.com/engineering/claude-code-best-practices)
- Rust Book: [Understanding Ownership](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html)
- Rust: [The Rustonomicon](https://doc.rust-lang.org/nomicon/)
- Bainbridge: [Ironies of Automation](https://doi.org/10.1016/0005-1098%2883%2990046-8)
- Simon Willison: [Not all AI-assisted programming is vibe coding](https://simonwillison.net/2025/Mar/19/vibe-coding/)
- TechTarget: [Replit AI agent snafu shot across the bow for vibe coding](https://www.techtarget.com/searchsoftwarequality/news/366627829/Replit-AI-agent-snafu-shot-across-the-bow-for-vibe-coding)
