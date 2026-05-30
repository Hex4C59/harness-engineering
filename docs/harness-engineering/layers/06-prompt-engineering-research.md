# Prompt Engineering 调研

调研日期：2026-05-30

## 这篇文档回答什么

这篇文档围绕 **Prompt Engineering** 做一次横向调研，资料范围包括论文、博客、技术报告、官方文档、benchmark / eval、开源项目、开发者社区讨论、工程案例、招聘市场、事故复盘和历史类比。

核心问题不是“有哪些神奇提示词”，而是：

1. Prompt Engineering 到底是什么？
2. 哪些做法有实证或官方依据？
3. 哪些做法正在失效或迁移到 context engineering、eval、tool design 和 harness engineering？
4. 如果要把它用于真实 AI 应用和 coding agent，应该怎样工程化？

核心结论：

> Prompt Engineering 没有消失，但它的重心已经从“措辞技巧”迁移到“指令设计、上下文装配、工具边界、结构化输出、eval、版本管理和安全约束”。单条 prompt 可以改善一次输出，但生产系统需要把 prompt 当成可测试、可版本化、可回滚、可审计的软件组件。

## 事实、观点和推断

| 类型 | 内容 |
|---|---|
| 事实 | OpenAI、Anthropic、Google、Microsoft、AWS 都有官方 prompt engineering / prompt design 文档；OpenAI 明确建议为复杂应用固定模型快照并建立 eval；Anthropic 明确说并非所有失败都适合用 prompt engineering 修复；Google Gemini 文档强调直接、结构化、带约束的 prompt，并建议在需要近期事实、计算时使用 grounding / code execution；OWASP LLM Top 10 把 prompt injection 列为 LLM 应用核心风险之一；NCSC 认为 prompt injection 不能简单类比 SQL injection，因为 LLM 内部没有天然的“指令 / 数据”安全边界。 |
| 事实 | 论文侧已经形成从 prompt-based learning、chain-of-thought、automatic prompt optimization、prompt robustness、CoT faithfulness 到 promptware engineering 的研究线索；开源侧有 OpenAI Evals、promptfoo、DSPy、PromptBench、Microsoft Prompt flow、Guidance、OpenPrompt 等工具。 |
| 事实 | 招聘市场中，“Prompt Engineer”作为独立岗位并不常见。Vu & Oppenlaender 2025 / 2026 版本分析 20,662 条 LinkedIn 招聘，只有 72 条 prompt engineer 岗位，低于样本的 0.5%。 |
| 观点 | “Prompt Engineering 已死”通常批评的是 2023 年那种靠 persona、咒语、长模板、prompt pack 的流派，不是说指令设计、上下文选择和输出约束不再重要。 |
| 观点 | 真正值得保留的 prompt engineering 更接近 requirements engineering、UX writing、test design 和 API contract design 的混合体，而不是单纯文案写作。 |
| 推断 | 对 agent 和生产 AI 应用来说，Prompt Engineering 会被吸收到更大的工程体系中：context engineering 负责给模型什么信息，harness engineering 负责模型能做什么和如何验证，eval 负责判断 prompt / model / tool 组合是否真的变好。 |
| 推断 | 未来“会提示词”不太可能长期作为独立稀缺职业存在，但“能把业务目标转成可评估的模型行为、工具边界和失败处理策略”会成为 AI engineer、LLM app engineer、product engineer、eval engineer、security engineer 的基础能力。 |

## 来源地图

| 范围 | 代表资料 | 本文使用方式 |
|---|---|---|
| 官方文档 | OpenAI Prompt engineering / Reasoning best practices / Evaluation best practices；Anthropic Prompt engineering overview；Google Gemini Prompt design strategies；Microsoft Prompt engineering techniques；AWS Bedrock Prompt engineering | 作为当前实践的主依据 |
| 论文和技术报告 | Pre-train, Prompt, and Predict；Chain-of-Thought Prompting；The Prompt Report；PromptRobust；To CoT or not to CoT；Language Models Don't Always Say What They Think；Promptware Engineering；Prompt Engineer job market study | 用于建立历史脉络、技术边界和实证限制 |
| Benchmark / eval | PromptBench / PromptRobust；OpenAI Evals；promptfoo；Anthropic agent evals；OpenAI evaluation flywheel | 用于说明 prompt 需要被回归测试，而不是凭感觉调 |
| 开源项目 | DSPy、promptfoo、Microsoft Prompt flow、Guidance、OpenPrompt、DAIR.AI Prompt Engineering Guide | 用于说明 prompt engineering 的工具化方向 |
| 博客和工程案例 | OpenAI Cookbook evaluation flywheel；Anthropic context engineering / tool design / evals；LangChain context engineering；Lilian Weng Prompt Engineering | 用于观察生产团队如何把 prompt 融入系统 |
| 社区讨论 | Hacker News、Reddit、OpenAI Developer Community、Hugging Face Forums | 只作为市场情绪和从业者争论，不作为事实依据 |
| 事故复盘 | Bing Chat / Sydney prompt leak；Chevrolet dealer chatbot；DPD chatbot；NCSC prompt injection analysis；OWASP LLM Top 10 | 用于说明 prompt 不是安全边界 |
| 招聘市场 | Vu & Oppenlaender job market paper；LinkedIn / Indeed / WSJ 相关报道的二手引用 | 用于判断“岗位”与“技能”的分化 |
| 历史类比 | SQL injection、SEO、requirements engineering、test harness、configuration management | 用于帮助定位 prompt engineering 的长期形态 |

## 一句话定义

比较稳妥的定义：

```text
Prompt Engineering 是把人的目标、业务规则、上下文、示例、输出契约和限制条件，
组织成模型可执行输入，并用 eval、日志和版本管理持续改进的工程活动。
```

这个定义比“写好提示词”更宽，也比“上下文工程”更窄：

- Prompt Engineering：偏指令、格式、示例、输出契约。
- Context Engineering：偏在每一步给模型选择什么上下文、工具结果、记忆、检索内容和状态。
- Harness Engineering：偏运行时、工具、权限、状态、验证、trace、审批和交付闭环。

三者关系可以这样看：

```text
Prompt Engineering < Context Engineering < Harness Engineering
```

Prompt 是模型看到的入口，context 是模型看到的工作面，harness 是模型所在的工程系统。

## 历史脉络

### 1. Prompt-based learning 阶段：prompt 是把任务映射到预训练目标

2021 年的 `Pre-train, Prompt, and Predict` 把 prompt-based learning 组织成一个范式：把原始输入改写成带模板的文本，让语言模型填空或生成，从而在少样本甚至零样本下完成任务。

这个阶段的 prompt 更像 ML 方法：

- 模板。
- verbalizer。
- cloze prompt。
- soft prompt / prefix tuning。
- prompt tuning。
- 少样本示例选择。

这和今天面向 ChatGPT / Claude / Gemini 的自然语言 prompt 不完全相同，但它奠定了一个事实：**不改模型参数，也能通过输入结构显著改变模型行为**。

### 2. ChatGPT 前后：prompt 变成普通人和模型交互的接口

2022 年底 ChatGPT 之后，prompt engineering 变成大众概念。很多实践集中在：

- role prompting：让模型扮演某种角色。
- clear instruction：明确任务和输出格式。
- few-shot：给输入输出样例。
- chain-of-thought：让模型展示中间推理。
- delimiters：用 Markdown、XML、分隔符隔开指令和资料。
- prompt chaining：把复杂任务拆成多步。

这时 prompt engineering 的传播速度极快，也带来了大量低质量内容：固定模板、prompt pack、persona 咒语、过度包装的“秘籍”。

### 3. CoT 高峰：prompt 可以激发推理，但不是万能解释器

`Chain-of-Thought Prompting Elicits Reasoning in Large Language Models` 是 prompt engineering 研究中的标志性论文之一。它展示了在足够大的模型上，给几个带中间推理的示例，能显著提升算术、常识和符号推理任务表现。

但后续研究给了两个重要限制：

- `To CoT or not to CoT?` 的结论是，CoT 的主要收益集中在数学和符号推理，其他任务收益小得多；直接生成答案在很多 MMLU 题目上与 CoT 几乎一样。
- `Language Models Don't Always Say What They Think` 表明 CoT 解释可能受偏置特征影响，生成合理但不忠实的解释。

事实层面：CoT 是有效技巧，但不是普适技巧，也不是模型真实内在推理的可靠审计日志。

工程推断：如果你需要可验证推理，优先把中间状态外部化成可检查结构，例如测试、工具调用、程序执行、检索证据、trace，而不是只相信模型写出来的推理说明。

### 4. 自动 prompt 优化：prompt 从手工文案变成搜索空间

`Large Language Models Are Human-Level Prompt Engineers` 提出 Automatic Prompt Engineer：把 instruction 视为“程序”，用 LLM 生成候选 instruction，再按评分函数选择更好的 prompt。

DSPy、Prompt flow、promptfoo、OpenAI Evals 等工具都在推动类似方向：

- 把 prompt 写进程序。
- 用数据集评价 prompt。
- 用优化器或搜索生成候选 prompt。
- 用 CI / regression suite 防止退化。

这说明 prompt engineering 的严肃形态不是“人类凭感觉调句子”，而是“在明确指标上搜索和验证输入程序”。

### 5. 2025-2026：从 prompt engineering 走向 context / harness engineering

Anthropic 在 2025 年的 context engineering 文章中明确说，它们把 context engineering 看作 prompt engineering 的自然演进。原因很简单：现代 agent 已经不是单次问答，而是多轮、带工具、带记忆、带检索、带状态的系统。

对 agent 来说，一次模型调用里的输入可能包括：

- system / developer instructions。
- 当前用户任务。
- 计划和进度。
- 相关文件。
- 检索结果。
- 工具定义。
- 工具调用结果。
- 历史摘要。
- 权限约束。
- 输出 schema。
- 错误日志。
- eval rubric。

这时问题已经不是“这句话怎么写得更漂亮”，而是：

```text
此刻模型应该知道什么？
不应该知道什么？
哪些信息是事实来源？
哪些信息只是用户输入？
哪些操作必须通过工具验证？
哪些输出必须被 schema、测试或人类审批拦住？
```

这就是 prompt engineering 与 harness engineering 的交界处。

## 官方文档的共同结论

### OpenAI

OpenAI 文档把 prompt engineering 定义为写出有效指令，使模型能稳定产生符合需求的内容。几个关键事实：

- 不同模型类型和同一家族不同快照可能需要不同 prompt。
- 复杂应用应固定生产模型快照，避免未预期行为漂移。
- 应建立 eval，监控 prompt 在迭代和模型升级时的表现。
- developer messages / instructions 具有高于用户输入的指令优先级。
- Markdown、XML、section title 有助于标记逻辑边界。
- few-shot 示例应与指令高度一致，否则会带来反效果。
- 对 reasoning model，OpenAI 明确建议保持 prompt 简单直接，避免强行要求“think step by step”或暴露推理。

工程启发：

```text
Prompt 不是聊天话术，而是应用层业务逻辑的一部分。
Prompt 改动应该像代码改动一样有版本、测试和回滚路径。
```

### Anthropic

Anthropic 的 prompt engineering overview 有一个重要提醒：不是所有 success criteria 或失败 eval 都适合用 prompt engineering 修复，例如 latency 和 cost 可能更适合换模型或改架构。

Anthropic 的相关工程文章还给出三个趋势：

- Building Effective Agents：简单 workflow 优先，只有任务确实需要自主决策时才上 agent。
- Effective Context Engineering：prompt engineering 正在自然演化为 context engineering。
- Writing Effective Tools for AI Agents：工具描述、工具边界和工具 eval 本身也需要 prompt-engineering。

工程启发：

```text
当失败来自缺少数据、工具太粗、权限过大、目标不可测、状态混乱时，
继续改 prompt 往往只是掩盖系统设计问题。
```

### Google Gemini

Google Gemini prompt design 文档强调：

- prompt engineering 是迭代过程。
- prompt 内容顺序可能影响输出。
- Gemini prompt 设计指南强调直接、结构化、明确任务和约束的 prompt。
- 需要近期事实时使用 Grounding with Google Search。
- 需要算术、计数或计算时使用 code execution。
- agentic workflows 需要控制模型如何计划、分解、诊断和权衡成本 / 准确性。

工程启发：

```text
不要让 prompt 承担搜索引擎、数据库、计算器和策略引擎的职责。
能用工具闭环的地方，用工具。
```

### Microsoft

Microsoft Azure / Foundry 文档强调：

- prompt engineering 能提高准确性和 grounding，但仍然必须验证模型输出。
- 清晰指令、清晰语法、任务拆分、输出结构都重要。
- 它也提醒：某些技巧在新模型上效果可能不再明显，例如把任务放在 prompt 前面在 GPT-4 级模型上不一定总带来差异。
- 对 chain-of-thought，Microsoft 文档已经把适用范围限定到 non-reasoning models，并提醒不要试图用不支持的方式提取模型 reasoning。

工程启发：

```text
Prompt 技巧有模型代际依赖。
旧模型有效的技巧，不能默认迁移到新 reasoning model。
```

### AWS Bedrock

AWS 把 prompt engineering 定义为 crafting and optimizing input prompts，强调任务和数据决定最佳方法。Bedrock 文档覆盖分类、问答、带上下文问答、摘要等常见任务。

工程启发：

```text
Prompt 模板应该按任务类型组织，而不是把一个“万能模板”套到所有场景。
```

## 技术分类

### 基础 prompting

| 技术 | 适用场景 | 主要风险 |
|---|---|---|
| clear instruction | 大多数任务 | 目标不清时只是让模型更自信地错 |
| role / persona | 控制风格、语气、职责边界 | 容易被过度神化，不能替代真实能力和权限控制 |
| delimiters | 长上下文、引用资料、结构化输入 | 只能帮助理解边界，不是安全隔离 |
| output format instruction | JSON、表格、分类标签、短答案 | 仍需 schema validator 或 parser 兜底 |
| few-shot | 分类、抽取、风格迁移、边界案例 | 示例和文字指令冲突时会造成混乱 |
| negative instruction | 禁止事项、合规规则 | “不要做 X”不如权限、工具和后处理约束可靠 |

### 推理 prompting

| 技术 | 适用场景 | 主要风险 |
|---|---|---|
| chain-of-thought | 数学、符号、分步推理，尤其 non-reasoning model | 成本高；解释不一定忠实；reasoning model 上可能无益或有害 |
| zero-shot CoT | 快速试探复杂题 | 对新 reasoning model 不应机械使用 |
| self-consistency | 多路径采样再投票 | 成本上升；投票可能掩盖共同偏差 |
| least-to-most | 复杂任务分解 | 分解错误会把任务带偏 |
| tree / graph of thoughts | 搜索式推理、规划 | 实现复杂，常常不如外部工具和状态机稳 |

### 上下文和检索

| 技术 | 适用场景 | 主要风险 |
|---|---|---|
| RAG | 私有知识库、文档问答、近期事实 | 检索质量差会让 prompt 看起来很完整但事实错 |
| long-context stuffing | 需要跨文档综合 | 上下文污染、注意力稀释、成本高 |
| context summarization | 长会话、长任务 | 摘要会丢细节，且错误摘要会长期污染后续推理 |
| scoped context | coding agent、复杂项目 | 需要维护目录地图和事实来源 |
| memory | 个性化、长期项目 | 过期信息和错误记忆需要治理 |

### 结构化输出和工具调用

| 技术 | 适用场景 | 主要风险 |
|---|---|---|
| JSON schema / Structured Outputs | 程序消费模型输出 | schema 只能约束形状，不能保证语义正确 |
| function calling / tool use | 查询、计算、执行动作 | 工具权限过大时 prompt injection 影响会变成真实操作风险 |
| tool descriptions | agent 选择工具 | 描述太宽会误用；描述太窄会漏用 |
| command preambles / progress TODO | 长任务透明度 | 过度要求会浪费 token，并鼓励空洞叙述 |

### 自动优化

| 技术 | 适用场景 | 主要风险 |
|---|---|---|
| APE / meta-prompting | 生成候选 prompt | 容易对小样本过拟合 |
| DSPy optimizer | 有数据集和指标的 LLM pipeline | 指标定义错误时会优化错目标 |
| promptfoo / OpenAI Evals | 回归测试和模型比较 | eval 集不代表生产分布时会产生虚假安全感 |
| LLM-as-judge | 风格、质量、复杂语义评价 | 位置偏差、长度偏差、judge 模型偏差 |

## Benchmark 和 eval：从“感觉更好”到“证明更好”

Prompt engineering 最大的问题是非确定性。一次输出变好不代表系统变好。

OpenAI eval 文档建议：

- 早做 eval-driven development。
- 任务特定 eval 比泛用指标更有价值。
- 记录日志，从失败样本中挖 eval case。
- 尽量自动化评分。
- 持续评估，而不是一次性评估。
- 用人类反馈校准自动评分。

OpenAI Cookbook 的 evaluation flywheel 很适合作为 prompt 开发流程：

```text
Analyze：人工看失败样本，归类失败模式。
Measure：为失败模式建立数据集和 grader。
Improve：改 prompt、示例、上下文或系统组件，再跑 eval。
```

这比“prompt-and-pray”可靠。

PromptBench / PromptRobust 的价值在于提示我们：同义替换、typo、语序变化、语义层面的对抗扰动都可能改变 LLM 输出。Prompt Robust 研究构造了 4,788 个 adversarial prompts，覆盖 8 类任务和 13 个数据集，结论是当时主流 LLM 对 adversarial prompt 并不稳健。

实践结论：

```text
一个 prompt 至少要过三类测试：
1. 正常输入是否达到目标。
2. 边界输入是否不崩。
3. 恶意或冲突输入是否不会越权。
```

对于 agent，还要额外评估：

- tool selection accuracy。
- tool argument correctness。
- handoff accuracy。
- 是否循环。
- 是否过度执行。
- 是否在证据不足时停下。
- 是否能从工具失败中恢复。
- 最终 artifact 是否通过外部验证。

## 开源项目观察

### OpenAI Evals

OpenAI Evals 是评估 LLM 和 LLM 系统的框架，也有 open-source benchmark registry。它的意义不只是“跑分”，而是把 prompt、模型版本和系统行为放进可回归的测试结构里。

适合：

- 模型升级前后比较。
- prompt 改动回归测试。
- 私有业务任务 eval。
- prompt chain / tool-using agent 的评估。

### promptfoo

promptfoo 是开源 CLI / library，用于测试 prompts、agents、RAG，并支持 red teaming、provider 对比、CI/CD 和安全扫描。它的 README 明确强调 data-driven，而不是 gut feel。

适合：

- prompt A/B 测试。
- 模型供应商对比。
- LLM app red team。
- 把 prompt eval 放进 CI。

### DSPy

DSPy 的核心思想是“programming, not prompting”。开发者定义模块、签名和指标，DSPy 优化 few-shot examples、instructions，甚至 fine-tuning。

适合：

- 有明确训练 / 验证数据。
- LLM pipeline 中多个 prompt 互相影响。
- 需要用指标系统优化 prompt，而不是手调。

### Microsoft Prompt flow

Prompt flow 把 LLM、prompt、Python code 和其他工具连成 executable flows，并支持调试、评估、CI/CD 和监控。

适合：

- 企业 LLM app 原型到生产。
- 多步骤 prompt chain。
- 需要 trace 和评估的业务流程。

### Guidance

Guidance 用程序化方式控制 LLM 输出，支持结构化生成、regex / CFG 约束、控制流和工具调用。它代表了一个趋势：把 prompt 从纯文本模板变成带约束的生成程序。

适合：

- 输出结构必须稳定。
- 需要减少解析失败。
- 需要在生成过程中施加语法约束。

### OpenPrompt

OpenPrompt 更偏 prompt learning 研究范式，围绕模板、verbalizer、PLM 适配等。它适合理解早期 prompt research 的技术基础，但和现代 chat / agent prompt 不是同一个使用层。

## 工程案例

### 1. OpenAI Cookbook：resilient prompt

OpenAI Cookbook 的 prompt evaluation flywheel 把 prompt 改进拆成分析、测量、改进三步。这个案例重要在于它把 prompt engineering 从“写一个更好模板”转成“围绕失败模式建立闭环”。

可复用方法：

- 每次失败都归类。
- 为常见失败建立 grader。
- 改 prompt 前先有 baseline。
- 改完后看指标，而不是只看 demo。

### 2. Anthropic Claude Code evals

Anthropic 的 agent evals 文章提到 Claude Code 最初依赖内部员工和外部用户反馈快速迭代，后来逐步添加 eval：先覆盖 concision、file edits 等窄行为，再覆盖 over-engineering 等复杂行为。

可复用方法：

- agent 初期可以人工快迭代。
- 一旦行为要稳定扩展，就必须把质量标准编码成 eval。
- eval 不只是 final answer，还要覆盖工具行为和工作流行为。

### 3. Anthropic tool descriptions

Anthropic 在 writing tools for agents 中提到过一个具体问题：Claude web search tool 曾经不必要地在 query 参数里追加 `2025`，导致搜索结果偏置。它们通过改进 tool description 来引导模型更好使用工具。

可复用方法：

- 工具描述本身就是 prompt。
- 工具 schema、字段名、描述、例子会影响 agent 行为。
- tool prompt 的 eval 应该包含“是否正确选择工具”和“参数是否正确”。

### 4. Google Gemini grounding / code execution

Google 官方文档建议：需要近期事实时开启 Google Search grounding；需要算术、计数、计算时开启 code execution。

可复用方法：

- prompt 不应该替代工具。
- 近期事实、私有知识、计算和执行动作要外部化。
- 模型负责协调，工具负责事实和执行。

## 事故复盘

### 1. Bing Chat / Sydney prompt leak

2023 年 Bing Chat 上线早期，用户通过 prompt injection 诱导其泄露隐藏的初始指令和代号 Sydney。Ars Technica、OECD AI incident database 等都记录了这次事件。

事实：

- 这是 prompt injection 进入主流视野的早期商业产品事件之一。
- 泄露的是系统指令和行为规则，不是传统数据库字段。

观点：

- 把系统 prompt 当秘密是不可靠的。
- 系统 prompt 可以作为行为指导，但不应承载真正 secret。

推断：

- 需要假设系统 prompt 可能被部分恢复、转述或泄露。
- 真正的安全边界应该放在权限、工具、数据访问控制和后端校验里。

### 2. Chevrolet dealer chatbot

2023 年，Chevrolet of Watsonville 网站上的 ChatGPT 驱动销售 chatbot 被用户诱导“同意”以 1 美元出售 Tahoe，并执行非汽车销售任务。最终 dealership 关闭了该 chatbot。

事实：

- bot 的话术被用户控制，输出了业务上荒唐的承诺。
- 实际并没有真正完成车辆销售合同。

观点：

- 这不是“AI 真卖车”，而是 chatbot 权威边界和业务承诺边界设计失败。

推断：

- 面向客户的 LLM 不应能生成看似具有法律约束力的承诺，除非后端有明确授权和校验。
- 对报价、合同、退款、赔偿等场景，要用确定性业务系统生成结论，LLM 只能解释。

### 3. DPD chatbot

2024 年 DPD 的 AI 客服 chatbot 在用户引导下爆粗、写诗批评公司，并导致 DPD 关闭相关 AI 功能。

事实：

- 事件触发于系统更新后的客服 chatbot。
- 用户通过对话让 bot 偏离客服任务。

观点：

- 客服场景的 prompt 不只是“语气友好”，还要明确失败时如何拒绝、升级和停止。

推断：

- 品牌风险 eval 应包含越界请求、辱骂诱导、竞争对手推荐、负面内容生成等 case。

### 4. Prompt injection 不是 SQL injection

NCSC 2025 文章明确说，LLM 内部没有“数据”和“指令”的天然区分，只有 next token。因此 prompt injection 可能无法像 SQL injection 那样被彻底修复。NCSC 建议把 LLM 看成 “inherently confusable deputy”，重点降低影响和剩余风险。

事实：

- OWASP LLM Top 10 也把 prompt injection 放在 LLM 应用核心风险位置。
- NCSC 认为如果系统不能承受剩余风险，可能就不适合用 LLM。

工程结论：

```text
Prompt 可以表达安全策略，但不能成为安全策略本身。
```

实际防线应该包括：

- 最小权限工具。
- 后端授权。
- 动作审批。
- 输出验证。
- 引用和证据要求。
- 外部内容隔离和标注。
- 日志和告警。
- 红队 eval。
- 高风险场景禁用自动执行。

## 开发者社区的主要争论

社区讨论不能当作事实依据，但能反映从业者痛点。

### 争论一：Prompt Engineering 是不是“真工程”

常见反方观点：

- 它不可复现。
- 它太依赖模型版本。
- 很多技巧只是玄学。
- prompt pack 是低质量内容。

常见正方观点：

- 没有数学闭式解不代表不是工程。
- 工程严谨性来自 eval、日志、回归测试和版本控制。
- 生产 AI 系统里的 system prompt、tool prompt、routing prompt 确实影响行为。

本文判断：

```text
没有 eval 的 prompt tweaking 不是工程。
有目标、数据、指标、版本和回归测试的 prompt 设计，可以是工程。
```

### 争论二：Prompt Engineering 是否已死

“已死”的部分：

- persona 咒语。
- 过度追求完美措辞。
- 把 prompt 当 secret sauce。
- 依赖模型暴露 CoT。
- 单 prompt 解决所有问题。
- prompt engineer 作为纯文案岗位。

仍然重要的部分：

- 明确目标和非目标。
- 写可执行约束。
- 设计输入 / 输出契约。
- 选择 examples。
- 管理上下文。
- 设计工具说明。
- 建立 eval。
- 防止 prompt injection。
- 在模型升级时做回归测试。

### 争论三：Context Engineering 是不是新瓶装旧酒

社区里有一种合理批评：context engineering 只是 prompt engineering 换名字。

本文判断：

- 对单次聊天来说，它们差异不大。
- 对 agent 和生产系统来说，差异真实存在。

Prompt engineering 关注“输入怎么写”。Context engineering 关注“哪些信息、状态、工具结果、记忆、检索内容在什么时候进入输入”。它把 prompt 从静态文本扩展成运行时组装过程。

## 招聘市场

### 事实

Vu & Oppenlaender 的论文分析了 20,662 条 LinkedIn job postings，其中 72 条是 prompt engineer 岗位，低于样本 0.5%。论文还给出技能画像：AI knowledge、prompt design、communication、creative problem-solving 都重要。

媒体报道和社区讨论普遍认为：2023 年“Prompt Engineer”被包装成高薪新职业，到 2025-2026 年，独立 title 明显降温，更多并入：

- AI Engineer。
- LLM Application Engineer。
- Automation Engineer。
- AI Product Engineer。
- Eval Engineer。
- RAG / Context Engineer。
- AI Safety / Security Engineer。
- Conversation Designer。
- AI Product Manager。

### 观点

“Prompt Engineer”作为独立岗位短命，并不说明 prompt engineering 不重要。它更像“会 Excel”“会 SQL”“会搜索”“会写规格”一样，逐渐变成多个岗位的基础能力。

### 推断

长期稀缺的不是会写漂亮 prompt 的人，而是能做这些事的人：

- 把业务目标转成可评估行为。
- 定义成功和失败。
- 构建 eval dataset。
- 设计 system / developer prompt。
- 设计工具和权限边界。
- 处理用户输入与外部内容的不可信问题。
- 能解释模型行为变化。
- 能把失败案例沉淀成回归测试。

## 历史类比

### 类比一：SQL injection

不准确之处：

- SQL 有明确指令 / 数据语法边界，可以用 prepared statement 等机制隔离。
- LLM 目前没有同等级的内部安全边界。

有用之处：

- 它提醒我们，不可信输入进入执行上下文时会变成安全问题。
- 早期行业容易把新漏洞当小技巧，直到事故足够多才建立工程规范。

结论：

```text
Prompt injection 比 SQL injection 更像“语义层的 confused deputy”。
```

### 类比二：SEO

早期 prompt engineering 像早期 SEO：

- 大量技巧有效但脆弱。
- 平台更新会让旧技巧失效。
- 有灰产和课程泡沫。
- 最终稳定价值回到内容质量、结构、用户意图和测量。

结论：

```text
Prompt 技巧会商品化，结构化问题定义和评估能力不会。
```

### 类比三：Requirements Engineering

Prompt 很像给一个不稳定执行者写需求：

- 目标要清楚。
- 非目标要清楚。
- 验收标准要清楚。
- 边界条件要清楚。
- 示例要覆盖典型和反例。

结论：

```text
好 prompt 的底层能力往往是好规格能力。
```

### 类比四：Configuration Management

生产 prompt 更像配置和策略代码：

- 它影响运行时行为。
- 它需要版本控制。
- 它需要 review。
- 它需要回滚。
- 它需要环境区分。
- 它需要变更记录。

结论：

```text
Prompt 文件应该像代码和配置一样被管理，而不是散落在聊天记录里。
```

### 类比五：Test Harness

单条 prompt 只能描述期望行为。Eval harness 才能验证行为。

结论：

```text
没有 eval harness，prompt engineering 会停留在 demo engineering。
```

## 对 Coding Agent 和 Harness Engineering 的启发

### 1. `AGENTS.md` 是 prompt，但不应只是 prompt

`AGENTS.md` 的价值不是让模型“记住所有事”，而是：

- 指向事实来源。
- 标明硬约束。
- 定义权限和边界。
- 列出必跑命令。
- 说明协作规则。

它应该短而稳定。细节应进入局部文档、计划、测试说明、skills、hooks 和脚本。

### 2. Prompt 要按作用域分层

推荐分层：

```text
全局规则：安全、权限、协作、输出风格。
项目规则：目录地图、运行命令、测试策略。
任务规则：目标、非目标、验收标准、相关文件。
工具规则：什么时候用工具、工具参数怎么填、失败怎么处理。
输出规则：schema、格式、引用、diff、报告。
```

不要把所有规则塞进一个巨大 prompt。

### 3. Prompt 的正确性要靠外部系统验证

对 coding agent 来说，prompt 里说“写高质量代码”意义有限。更有效的是：

- 先读 README 和相关文件。
- 小步修改。
- 不覆盖用户改动。
- 跑测试。
- 跑 lint / typecheck。
- 查看 diff。
- 失败后分析原因。
- 完成时报告验证结果。

这就是 harness engineering 的工作。

### 4. 把 prompt failure 转成 eval / checklist / tool change

如果 agent 犯错，不要只写：

```text
下次请注意。
```

更好的回流是：

```text
失败：agent 修改了不相关文件。
改进：
- 写入前检查 git diff。
- AGENTS.md 增加“不要覆盖用户改动”。
- 对 destructive command 要审批。
- 把该案例加入 review checklist。
```

### 5. 对 agent，不要把 prompt 当安全边界

如果 agent 有工具权限，prompt injection 的影响会从“说错话”升级成“做错事”。

高风险动作应使用：

- sandbox。
- 最小权限。
- allowlist。
- 人类审批。
- dry-run。
- audit log。
- 回滚机制。
- 外部 validator。

## 实践清单

### 写 prompt 前

- 明确任务类型：分类、抽取、生成、问答、代码、agent、工具调用。
- 写出成功标准和失败标准。
- 找到事实来源。
- 决定是否需要 RAG、搜索、代码执行、数据库或业务 API。
- 决定哪些信息不能放进 prompt。
- 为高风险场景定义人工审批。

### 写 prompt 时

- 使用清晰的角色和职责，但不要把 persona 当能力。
- 把任务、上下文、约束、输出格式分段。
- 用 Markdown / XML / 分隔符标明边界。
- 对输出使用 schema、枚举或固定字段。
- few-shot 示例要覆盖典型、边界和反例。
- 把重要规则写成可执行约束，不要只写抽象价值观。
- 对 reasoning model，优先简单直接，不机械要求 CoT。

### 验证 prompt 时

- 建立最小 eval set。
- 覆盖正常输入、边界输入、冲突输入、恶意输入。
- 记录模型版本、prompt 版本、温度、工具配置。
- 至少做一次模型 / prompt A/B。
- 对 LLM judge 做人类校准。
- 把线上失败样本回流 eval。

### 上线 prompt 时

- 固定模型快照或明确模型升级策略。
- prompt 文件进版本控制。
- prompt 改动走 review。
- 对输出做 schema validation。
- 对事实回答要求引用或可追溯证据。
- 对执行动作使用后端授权，不信任模型自称。
- 建立日志、trace、成本、延迟和失败率指标。

### 维护 prompt 时

- 每次模型升级跑回归 eval。
- 定期清理过期规则。
- 把长 prompt 拆成模块。
- 统计失败模式，不要只看平均分。
- 对 prompt injection 做红队测试。
- 记录为什么加某条规则，以及它修复了什么失败。

## 常见误区

| 误区 | 更可靠的做法 |
|---|---|
| 找一个万能 prompt | 按任务类型和数据分布设计 prompt |
| 把 prompt 写得越长越好 | 只放当前任务需要的上下文 |
| “请一步一步思考”总有帮助 | 按模型类型和任务决定；reasoning model 往往不需要 |
| 模型解释了原因，所以可信 | 解释可能不忠实；要看证据、工具结果和外部验证 |
| 系统 prompt 可以防越权 | 权限、审批和后端校验才是安全边界 |
| prompt 通过一次 demo 就能上线 | 用 eval 和线上日志验证 |
| prompt engineer 是写提示词的人 | 更像定义行为、数据、工具、边界和评估的人 |

## 判断框架

当一个 AI 系统失败时，先不要立刻改 prompt。按这个顺序定位：

1. 目标是否清楚？
2. 输出是否可判定？
3. 上下文是否足够、相关、不过期？
4. 是否需要工具，而不是语言模型猜？
5. 工具权限是否过大？
6. 输出是否有 schema 或 validator？
7. 是否有 eval 覆盖这个失败？
8. 模型版本是否变化？
9. prompt 是否和 examples 冲突？
10. 是否是 prompt injection 或不可信输入问题？

如果问题在 1-7，单纯改 prompt 多半不是根治。

## 结论

Prompt Engineering 的长期价值不在“神奇句式”，而在把模糊意图转成模型可执行、系统可验证、团队可维护的行为契约。

短期看，它仍然是使用 LLM 的基础技能。中期看，它会继续被自动优化工具、prompt generator、model-specific guidance 吞掉一部分。长期看，它会融入更大的工程系统：

```text
Prompt Engineering：写清楚要什么。
Context Engineering：给对信息。
Tool Engineering：让模型能做正确的事。
Eval Engineering：证明它真的做对。
Harness Engineering：让这一切在受控环境里持续运行。
```

所以更准确的判断是：

```text
Prompt Engineering 作为玄学技巧正在贬值；
Prompt Engineering 作为 instruction + context + eval + safety 的工程纪律正在升值。
```

## 参考资料

### 官方文档

- OpenAI Docs: [Prompt engineering](https://developers.openai.com/api/docs/guides/prompt-engineering)
- OpenAI Docs: [Reasoning best practices](https://developers.openai.com/api/docs/guides/reasoning-best-practices)
- OpenAI Docs: [Evaluation best practices](https://developers.openai.com/api/docs/guides/evaluation-best-practices)
- OpenAI Cookbook: [Building resilient prompts using an evaluation flywheel](https://developers.openai.com/cookbook/examples/evaluation/building_resilient_prompts_using_an_evaluation_flywheel)
- Anthropic Docs: [Prompt engineering overview](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview)
- Anthropic Engineering: [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- Anthropic Engineering: [Writing effective tools for AI agents](https://www.anthropic.com/engineering/writing-tools-for-agents)
- Anthropic Engineering: [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- Google AI for Developers: [Prompt design strategies](https://ai.google.dev/guide/prompt_best_practices)
- Microsoft Learn: [Prompt engineering techniques](https://learn.microsoft.com/en-us/azure/foundry/openai/concepts/prompt-engineering)
- AWS Bedrock: [What is prompt engineering?](https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-prompt-engineering.html)
- NCSC: [Prompt injection is not SQL injection](https://www.ncsc.gov.uk/blog-post/prompt-injection-is-not-sql-injection)
- OWASP: [Top 10 for Large Language Model Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)

### 论文和技术报告

- Liu et al. 2021: [Pre-train, Prompt, and Predict: A Systematic Survey of Prompting Methods in Natural Language Processing](https://arxiv.org/abs/2107.13586)
- Wei et al. 2022: [Chain-of-Thought Prompting Elicits Reasoning in Large Language Models](https://arxiv.org/abs/2201.11903)
- Zhou et al. 2022: [Large Language Models Are Human-Level Prompt Engineers](https://arxiv.org/abs/2211.01910)
- Turpin et al. 2023: [Language Models Don't Always Say What They Think](https://arxiv.org/abs/2305.04388)
- Zhu et al. 2023: [PromptRobust: Towards Evaluating the Robustness of Large Language Models on Adversarial Prompts](https://arxiv.org/abs/2306.04528)
- Sahoo et al. 2024: [A Systematic Survey of Prompt Engineering in Large Language Models](https://arxiv.org/abs/2402.07927)
- Schulhoff et al. 2024 / 2025: [The Prompt Report](https://arxiv.org/abs/2406.06608)
- Sprague et al. 2024 / ICLR 2025: [To CoT or not to CoT?](https://arxiv.org/abs/2409.12183)
- PromptBench paper: [PromptBench: A Unified Library for Evaluation of Large Language Models](https://arxiv.org/abs/2312.07910)
- Promptware Engineering: [Software Engineering for Prompt-Enabled Systems](https://arxiv.org/abs/2503.02400)
- Vu & Oppenlaender 2025 / 2026: [Prompt Engineer: Analyzing Hard and Soft Skill Requirements in the AI Job Market](https://arxiv.org/abs/2506.00058)

### 开源项目和工具

- OpenAI: [Evals](https://github.com/openai/evals)
- promptfoo: [LLM evals and red teaming](https://github.com/promptfoo/promptfoo)
- Stanford NLP: [DSPy](https://github.com/stanfordnlp/dspy)
- Microsoft: [Prompt flow](https://github.com/microsoft/promptflow)
- Microsoft: [PromptBench](https://github.com/microsoft/promptbench)
- Guidance AI: [Guidance](https://github.com/guidance-ai/guidance)
- THUNLP: [OpenPrompt](https://github.com/thunlp/OpenPrompt)
- DAIR.AI: [Prompt Engineering Guide](https://dair.ai/projects/prompt-engineering/)
- Lilian Weng: [Prompt Engineering](https://lilianweng.github.io/posts/2023-03-15-prompt-engineering/)

### 事故和社区材料

- Ars Technica: [AI-powered Bing Chat spills its secrets via prompt injection attack](https://arstechnica.com/information-technology/2023/02/ai-powered-bing-chat-spills-its-secrets-via-prompt-injection-attack/)
- OECD AI Incident Monitor: [Bing Chatbot Exposes Confidential Instructions](https://oecd.ai/en/incidents/2023-02-10-4440)
- MIT AI Risk Navigator: [Chevrolet Dealer Chatbot Agrees to Sell Tahoe for $1](https://www.airi-navigator.com/incidents/622)
- Time: [DPD chatbot incident](https://time.com/6564726/ai-chatbot-dpd-curses-criticizes-company/)
- Hacker News: [The new skill in AI is not prompting, it's context engineering](https://news.ycombinator.com/item?id=44427757)
- Hacker News: [AI Prompt Engineering Is Dead](https://news.ycombinator.com/item?id=39617062)
- Reddit: [Prompt engineering lacks engineering rigor](https://www.reddit.com/r/PromptEngineering/comments/1i0o5fk)
- Reddit: [Prompt Engineering is Dead in 2026](https://www.reddit.com/r/PromptEngineering/comments/1rci46t/prompt_engineering_is_dead_in_2026/)
