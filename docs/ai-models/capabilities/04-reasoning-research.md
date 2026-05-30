# Reasoning 调研

调研日期：2026-05-30

## 这篇文档回答什么

这里的 `Reasoning` 不是一个单一技术，而是一组围绕“让模型在回答前、调用工具中、执行任务时投入更多计算和搜索”的方法集合。

它至少包含四层含义：

```text
Reasoning =
  模型训练出来的多步推理能力
  + 推理时可调的 test-time compute
  + 外部工具、搜索、代码执行和环境反馈
  + 用 benchmark / eval / trace 判断推理是否真的有用
```

核心结论：

> Reasoning 的工程价值不在于模型会不会展示一段漂亮的思维链，而在于它能不能在复杂、模糊、多步骤、高风险任务中更稳定地做出正确决策，并且让成本、延迟、可验证性和安全边界仍然可控。

所以评价 reasoning，要少问：

```text
模型有没有“想”？
```

多问：

```text
它多花的 token、时间、工具调用和人类等待，是否换来了可复现的正确率、鲁棒性和任务完成率提升？
```

## 来源说明

本文覆盖论文、官方文档、技术报告、benchmark、开源项目、开发者社区、工程案例、招聘市场、事故复盘和历史类比。对可能变化的信息，按 2026-05-30 可查资料整理。

社区讨论只作为观察开发者关注点的入口，不单独作为事实依据。模型榜单和发布分数只作为当时产品状态，不写成长期结论。

| 来源 | 类型 | 主要价值 |
|---|---|---|
| [Chain-of-Thought Prompting Elicits Reasoning in Large Language Models](https://arxiv.org/abs/2201.11903) | 论文 | CoT 作为 prompt 技术的起点之一，说明中间步骤能显著改善大模型多步任务 |
| [Self-Consistency Improves Chain of Thought Reasoning in Language Models](https://arxiv.org/abs/2203.11171) | 论文 | 用多条推理路径采样和投票提升答案稳定性 |
| [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629) | 论文 | 把 reasoning trace 和 action / observation 交错，用于工具使用和环境交互 |
| [Tree of Thoughts](https://arxiv.org/abs/2305.10601) | 论文 | 把推理变成搜索问题，让模型生成、评估和回溯多个候选思路 |
| [Reflexion](https://arxiv.org/abs/2303.11366) | 论文 | 通过 verbal feedback 和自我反思改善后续尝试 |
| [STaR](https://arxiv.org/abs/2203.14465) | 论文 | 用模型自己生成的 rationale 迭代训练推理能力 |
| [OpenAI: Learning to reason with LLMs](https://openai.com/index/learning-to-reason-with-llms/) | 官方技术博客 | o1 把 reasoning 明确产品化，并强调 RL 与 test-time thinking |
| [OpenAI Reasoning models](https://platform.openai.com/docs/guides/reasoning) | 官方文档 | reasoning tokens、reasoning effort、reasoning summaries、上下文和成本控制 |
| [OpenAI Reasoning best practices](https://platform.openai.com/docs/guides/reasoning-best-practices) | 官方文档 | 区分 reasoning model 和 GPT model，给出 planner / doer、prompt 和成本建议 |
| [OpenAI Chain-of-thought monitorability](https://openai.com/index/chain-of-thought-monitoring/) | 官方研究 | 说明 CoT 可监控性价值，也提醒不能简单监督或暴露原始 CoT |
| [Anthropic Extended thinking](https://docs.anthropic.com/en/docs/build-with-claude/extended-thinking) | 官方文档 | Claude 的 extended thinking、thinking budget、工具间思考和加密思考块 |
| [Anthropic Extended thinking tips](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/extended-thinking-tips) | 官方文档 | 对 Claude extended thinking 的提示词、预算和工具使用建议 |
| [Google Gemini Thinking](https://ai.google.dev/gemini-api/docs/thinking) | 官方文档 | Gemini 的 thinking budget、include thoughts 和模型思考控制 |
| [Gemini 2.5 Pro](https://blog.google/technology/google-deepmind/gemini-model-thinking-updates-march-2025/) | 官方博客 | Google 把 Gemini 2.5 称为 thinking model，并把 reasoning 与 coding / math / science 绑定 |
| [DeepSeek-R1](https://github.com/deepseek-ai/DeepSeek-R1) / [论文](https://arxiv.org/abs/2501.12948) | 开源项目 / 技术报告 | 展示 RL 强化 reasoning、可蒸馏到小模型、开放权重推动社区复现 |
| [Hugging Face Open-R1](https://github.com/huggingface/open-r1) | 开源项目 | 社区复现 DeepSeek-R1 pipeline，推动 reasoning 训练开源化 |
| [GPQA](https://arxiv.org/abs/2311.12022) | Benchmark | 研究生级、Google-proof 科学问答，常用于推理模型发布 |
| [FrontierMath](https://epoch.ai/frontiermath) | Benchmark | 高难数学题，关注强模型在专家数学上的边界 |
| [Humanity's Last Exam](https://lastexam.ai/) | Benchmark | 高难多学科封闭题，用于衡量强模型在专家知识与推理上的剩余差距 |
| [ARC-AGI](https://arcprize.org/arc-agi) | Benchmark | 抽象与程序归纳类推理，强调低先验和泛化 |
| [SWE-bench](https://www.swebench.com/) | Benchmark | 真实 GitHub issue 修复，把 reasoning 放进代码仓库、测试和 agent harness |
| [LiveCodeBench](https://arxiv.org/abs/2403.07974) | Benchmark | 动态更新代码题，降低训练污染，覆盖执行、自修复和测试输出预测 |
| [The Illusion of Thinking](https://machinelearning.apple.com/research/illusion-of-thinking) | 论文 / 批判 | 用可控 puzzle 分析 reasoning model 的能力边界和规模泛化问题 |
| [Anthropic Claude Code quality postmortem](https://www.anthropic.com/engineering/april-23-postmortem?pubDate=20260425) | 事故复盘 | Claude Code reasoning effort、thinking history 和 prompt 变更说明推理预算是产品可靠性风险点 |
| [OpenAI Codex Core Agent 岗位](https://openai.com/careers/applied-ai-engineer-codex-core-agent-san-francisco/) | 招聘市场 | 岗位要求 eval、failure modes、tool-use、context construction 和 agent robustness |
| Hacker News / Reddit / LocalLLaMA 等讨论 | 开发者社区 | 观察真实用户对成本、延迟、开源复现、CoT 可见性和“过度思考”的关注 |
| AlphaGo / AlphaZero / MCTS | 历史类比 | test-time search 与 learned policy/value 结合，是理解 reasoning model 的有用类比 |

## 事实、观点和推断

| 类型 | 本文用法 |
|---|---|
| 事实 | 来源中明确出现的方法、API、benchmark 设计、事故描述、岗位要求、项目机制 |
| 观点 | 研究者、厂商、工程团队和开发者社区对 reasoning 的解释、经验和争议 |
| 推断 | 基于多类来源共同指向的趋势，整理出的工程判断框架 |

本文最重要的边界：

```text
Reasoning 能提升复杂任务表现，但不是正确性的同义词。
Chain-of-thought 能改善部分任务，但不是可审计证明。
Thinking budget 能买来更多搜索，但也会买来成本、延迟和新的失败模式。
```

## 事实一：Reasoning 从 prompt trick 变成了模型产品能力

早期 LLM reasoning 的核心入口是 prompt。CoT 论文展示了“给出中间推理步骤”可以改善算术、常识和符号推理。Self-consistency 进一步说明，不只生成一条推理链，而是采样多条链并投票，可以提升答案稳定性。Tree of Thoughts 把这个方向推进为搜索：模型不只线性生成答案，而是生成多个候选 thought，评估、回溯、继续展开。

这条脉络的共同点是：

- 推理不是一次 token 预测就结束。
- 多步中间状态能帮助模型处理复杂问题。
- 多路径采样、投票、搜索和回溯可以把 test-time compute 换成更高成功率。

到了 o1、Claude extended thinking、Gemini thinking、DeepSeek-R1 之后，reasoning 不再只是“请一步一步想”的 prompt 技巧，而变成模型和 API 层能力：

- OpenAI 暴露 `reasoning.effort`，并在 usage 中记录 `reasoning_tokens`。
- Anthropic 暴露 extended thinking 和 thinking budget，并支持工具使用过程中的 thinking block。
- Gemini API 支持 thinking budget，有些模型默认开启 thinking，也可按任务关闭或调整。
- DeepSeek-R1 把长链推理和 RL 训练作为技术报告主线，并开放权重和蒸馏模型。

**事实结论：** Reasoning 已经从“提示词写法”上升为“训练方法 + 模型族 + API 参数 + 计费和延迟模型”。

## 事实二：Reasoning 的主要收益来自 test-time compute

OpenAI 文档明确说明，reasoning model 会先使用内部 reasoning tokens，再生成最终回答；这些 token 不通过 API 暴露原文，但会占上下文窗口，并作为输出 token 计费。官方建议在实验 reasoning model 时预留足够 token 空间，避免模型在推理阶段耗尽 `max_output_tokens`，导致付费但没有可见输出。

Anthropic 和 Gemini 的文档也类似：`thinking budget` 或 `reasoning.effort` 本质上是让模型在回答前或工具调用之间使用更多计算。

工程上可以把它理解成：

```text
普通回答：prompt -> answer
reasoning 回答：prompt -> hidden search / deliberation -> answer
agentic reasoning：prompt -> think -> tool -> observe -> think -> tool -> ... -> answer / artifact
```

这和 AlphaGo / AlphaZero 的历史类比很接近：模型参数提供直觉，test-time search 提供更强局面评估和决策。区别是 LLM 的搜索空间是语言、工具调用、代码执行和环境状态，而不是棋盘合法走法。

**事实结论：** Reasoning 的成本不是抽象的，它会具体表现为更多 output token、更多延迟、更多上下文占用、更多工具调用和更复杂状态管理。

## 事实三：Reasoning 与 agent 工具使用已经绑在一起

ReAct 论文很早就把 reasoning 和 acting 放在同一个循环里：模型生成推理片段，然后执行动作，观察环境，再继续推理。今天的 coding agent、research agent、data agent 基本都沿着这个结构演化。

OpenAI reasoning 文档建议在函数调用场景中保留 reasoning items，特别是在连续工具调用时，让模型可以延续推理状态，而不是每轮重新开始。Anthropic 的 extended thinking 文档也把工具使用前后的 thinking block、加密 thinking 和 token budget 作为重要机制。

这说明 reasoning 的工程对象不是一段答案，而是一条运行轨迹：

```text
user goal
-> model thought / plan
-> tool call
-> observation
-> revised thought
-> more tool calls
-> final answer or patch
```

在 coding agent 场景里，SWE-bench、Terminal-Bench、LiveCodeBench 等 benchmark 都在把推理能力放进真实执行环境中评估。模型需要读仓库、定位文件、解释失败、修改代码、运行测试、根据错误继续迭代。

**事实结论：** 高价值 reasoning 越来越不是“纸面解题”，而是“在环境反馈中持续修正决策”。

## 事实四：Benchmark 正在追着 reasoning model 变难

Reasoning model 发布材料常用的 benchmark 可以粗分为几类：

| 类别 | 代表 benchmark | 主要测什么 |
|---|---|---|
| 学科专家推理 | GPQA Diamond、MMLU-Pro、HLE | 跨学科知识、科学推理、专家题 |
| 数学推理 | AIME、FrontierMath、OlympiadBench | 多步数学、证明感、搜索和反例意识 |
| 抽象泛化 | ARC-AGI | 从少量示例归纳规则并泛化 |
| 代码推理 | LiveCodeBench、SWE-bench、Aider、Terminal-Bench | 代码生成、调试、仓库理解、环境操作 |
| 工具和 agent | τ-bench、BFCL、SWE-bench、Terminal-Bench | 工具选择、多轮状态、业务规则和任务完成 |
| 长上下文推理 | MRCR、Needle / multi-needle、长文档 QA | 在大量信息中筛选、合成和判断 |

这些 benchmark 的共同趋势是：

- 从短答案题转向多步骤任务。
- 从静态题库转向动态更新或私有 holdout。
- 从裸模型回答转向带工具、带 agent harness 的完整运行。
- 从单一 accuracy 转向 resolved rate、pass@1、cost、latency、trace 和 failure mode。

**事实结论：** Reasoning 的评测对象已经从“模型会不会说出正确答案”扩展为“model + harness + tool + budget + evaluator 的整体表现”。

## 事实五：开源 reasoning 改变了竞争结构

DeepSeek-R1 是 reasoning 发展中的重要节点。它把 RL 强化 reasoning、长链推理、蒸馏到较小模型和开放权重放在同一个项目里。更关键的是，它让社区不再只能等待闭源厂商发布 reasoning 模型，而是可以：

- 研究 reasoning 训练数据和 RL pipeline。
- 蒸馏、微调和部署自己的 reasoning 模型。
- 对比不同 prompt、reward、verifier 和 test-time compute 策略。
- 用 Open-R1 这类项目复现和改造训练流程。

开源 reasoning 也带来一个现实变化：许多团队会用闭源强模型做高价值 planning / judging，用开源或小模型做低成本执行、验证或本地私有任务。

**事实结论：** Reasoning 不再只是 frontier lab 的黑盒能力，正在变成开源模型、推理框架、RL 训练和本地部署生态的一部分。

## 事实六：Reasoning 会引入新的产品事故面

Anthropic 在 2026-04-23 的 Claude Code quality report 中把近期质量问题归因于三类产品层变更：默认 reasoning effort 从 `high` 调到 `medium`、闲置会话恢复时错误地持续清理旧 thinking history、以及降低 verbosity 的 system prompt 变更。Anthropic 明确说 API 和 inference layer 没有受影响，问题出在 Claude Code、Claude Agent SDK 和 Claude Cowork 的产品 / harness 层。

这件事很有代表性：用户感受到的是“模型变笨了”，但根因可能是 reasoning budget、上下文保留、prompt 和产品默认值的组合变化，而不是 base model 能力突然下降。

OpenAI 文档也提醒：如果 `max_output_tokens` 或上下文空间不足，模型可能在 reasoning 阶段耗尽预算，返回 `incomplete`，甚至没有可见输出。这在用户体验、成本控制和任务可靠性上都会变成具体问题。

常见事故面包括：

- reasoning effort 配置错误。
- 模型在简单任务上过度思考，成本和延迟异常。
- 复杂任务 thinking budget 不够，质量显著下降。
- 工具调用之间没有保留必要 reasoning state，导致重复思考或断链。
- UI 把中间 preamble 当 final answer。
- 用户误以为 reasoning summary 等于完整真实思维链。

**事实结论：** Reasoning 参数需要像 timeout、retry、rate limit、memory limit 一样纳入工程治理和回归测试。

## 事实七：招聘市场已经把 reasoning 放进 eval、post-training 和 agent reliability

OpenAI Codex Core Agent 相关岗位强调真实 coding task 上的 agent behavior、eval、regression、failure modes、tool-use strategy、context construction 和 robustness。Anthropic、Google DeepMind、OpenAI 等前沿团队的 post-training、eval、agent、alignment 岗位也经常要求候选人理解模型评估、RL、工具使用、复杂任务失败分析和产品化实验。

这说明 reasoning 不是只属于研究员的问题。工程团队需要的人才画像正在变成：

```text
会定义任务
+ 会设计 eval
+ 会读 agent trace
+ 会定位 reasoning / tool / context failure
+ 会把 failure mode 转成训练、prompt、工具或 harness 改进
```

**事实结论：** 市场正在为“让模型更会想、并知道什么时候想错了”的能力付钱。

## 事实八：工程案例显示 reasoning 更适合作为关键判断层

OpenAI reasoning best practices 收集的客户案例集中在几个方向：

- 法律和金融文档：Hebbia、Endex、Blue J、BlueFlame AI 等案例强调，在密集合同、税务研究、金融条款和复杂估值问题中，reasoning model 更擅长跨文档综合和处理隐含关系。
- Agent planning：Argon AI、Lindy 等案例把 o1 用作 planner 或多步骤 agent 决策层，再把局部执行交给其他模型或工具。
- 视觉与合规：SafetyKit 案例强调 reasoning model 在困难图像分类和风险判断中的提升。
- 代码审查与 eval：CodeRabbit、Braintrust 等案例把 reasoning model 用于代码 review 或 LLM-as-judge，因为这些任务不一定低延迟，但需要更细的判断。

这些案例都是供应商材料里的客户叙述，不能当作独立实验结论。但它们共同说明一个实践方向：

```text
Reasoning 最适合放在“复杂判断、规划、审查、仲裁”这些高价值节点，
而不是替代所有普通生成和格式化步骤。
```

**事实结论：** 真实工程系统更倾向把 reasoning model 当关键判断层，而不是全链路默认模型。

## 观点层：围绕 Reasoning 的几类争议

### 观点一：CoT 是能力放大器，但不等于解释

许多论文和实践都证明，显式或隐式中间推理能改善复杂任务表现。但另一个研究方向也提醒：模型展示的 chain-of-thought 不一定忠实反映真实因果过程。模型可能先受到偏置、格式、答案线索影响，再生成看起来合理的解释。

OpenAI 的 chain-of-thought monitorability 研究把 CoT 看成一个有价值的监控窗口，但也强调不能简单用强监督把原始 CoT 压成讨好人类的解释，否则可能损害可监控性。

实践含义：

- 可以把 reasoning trace 当 debug 线索。
- 不要把可见 CoT 当数学证明或审计证据。
- 对高风险任务，最终仍要依赖外部验证：测试、证明器、规则引擎、人类 review、可复现实验。

### 观点二：Reasoning model 更像 planner，不一定适合所有 token

OpenAI best practices 把 reasoning model 描述为更适合复杂规划、决策和模糊任务的 planner，而低延迟 GPT model 更适合明确执行任务的 workhorse。这个分工在实际系统里很重要。

典型架构是：

```text
reasoning model：拆任务、定策略、审查风险、做最终判断
fast model：改写、抽取、分类、格式化、执行局部明确步骤
工具 / 程序：搜索、计算、运行测试、校验 schema、操作环境
```

观点结论：

> 不是每一步都应该用最强 reasoning model。好的系统会把 reasoning 花在最值得花的地方。

### 观点三：Reasoning 对 coding agent 很重要，但 harness 决定上限

Coding agent 需要 reasoning，因为它要理解需求、读仓库、定位 bug、规划修改、解释测试失败。但同一个模型在不同 agent scaffold 下表现差异很大。SWE-agent、OpenHands、Codex、Claude Code、Jules 等系统都说明：工具接口、上下文选择、编辑器、shell、测试、权限和状态文件会显著影响最终结果。

观点结论：

```text
Reasoning 是 agent 的认知引擎。
Harness 是它能否把认知变成可靠交付的工程系统。
```

### 观点四：开发者社区对 reasoning 的关注很工程化

Hacker News、Reddit、LocalLLaMA 等社区围绕 o1、DeepSeek-R1、Gemini thinking 和 Apple reasoning 批判论文的讨论，反复出现几个主题：

- 推理成本和延迟是否值得。
- 开源模型能否追上闭源 reasoning model。
- 原始 CoT 不可见是否影响调试和信任。
- reasoning 是否只是更长输出，还是确实更强搜索。
- 模型在简单任务上“过度思考”会不会降低产品体验。
- benchmark 是否被污染、是否过拟合、是否能代表真实工作。

这些讨论有噪声，但它们说明真实用户关心的不是哲学上的“会不会推理”，而是：

```text
它是不是更准？
慢多少？
贵多少？
能不能调？
错的时候我能不能知道为什么？
能不能接进我的工具链？
```

### 观点五：批判研究提醒我们不要把 reasoning 神秘化

Apple 的 The Illusion of Thinking 使用可控 puzzle 分析 reasoning model，指出这些模型在某些复杂度区间表现提升，但在更高复杂度下仍会崩溃，而且不一定表现出稳定的算法泛化。类似批判的价值不是证明 reasoning 没用，而是提醒：

- 题目规模、分布和复杂度会暴露不同失败模式。
- 更长 thinking 不等于真正学会算法。
- 强 benchmark 分数不等于稳健泛化。
- 需要可控实验，而不只是发布页分数。

观点结论：

> Reasoning model 是更强的统计和搜索系统，不应被直接等同于人类式理解或形式化算法能力。

## 推断：Reasoning 会重塑六个工程层面

### 1. 模型选型从“最强模型”变成“预算分配”

未来系统更像一个推理预算调度器：

- 简单分类、抽取、格式转换：低 reasoning 或非 reasoning model。
- 复杂规划、代码审查、法律/金融/科研判断：中高 reasoning。
- 异步研究、复杂 debugging、安全审计：高 reasoning，配合工具和人类 review。
- 可计算问题：尽量调用程序、求解器、数据库、测试，而不是让模型空想。

关键指标不是单次最高正确率，而是：

```text
成功率 / 成本 / 延迟 / 可解释调试性 / 人类等待时间 / 失败损失
```

### 2. Prompt 重点从“教它怎么想”变成“定义成功边界”

对现代 reasoning model，继续写“think step by step”通常不是最重要的。更有价值的是：

- 明确目标和非目标。
- 给出约束、输入、输出契约。
- 定义完成条件。
- 要求使用工具验证。
- 说明何时停止、何时提问、何时升级给人。

这和 OpenAI reasoning best practices 一致：给清晰目标和约束，不要过度规定中间步骤。

### 3. Eval 必须记录 reasoning budget

比较 reasoning model 时，至少要记录：

- 模型版本。
- reasoning effort / thinking budget。
- 是否允许工具。
- 最大 token / 最大时间。
- 是否多次采样、投票、rerank。
- 是否使用 agent scaffold。
- 成本、延迟和失败类型。
- prompt、上下文和随机种子或系统指纹。

否则两个结果可能不可比。一个模型看似更强，可能只是用了更多 test-time compute、更多样本、更多工具或更宽松评分器。

### 4. Trace 会变成 reasoning 工程的核心资产

Reasoning 失败很少只发生在最终答案。它可能发生在：

- 错读目标。
- 选错工具。
- 忽略约束。
- 过早停止。
- 反复搜索同一方向。
- 测试失败后错误归因。
- 把观察结果纳入上下文时丢失关键信息。

因此 trace 必须记录：

- 输入和上下文选择。
- reasoning effort / thinking budget。
- 每次模型调用。
- 工具调用、参数和返回。
- 文件 diff、命令输出、测试结果。
- 中间状态、preamble、summary。
- 人类审批和修改。
- 最终结果与评分。

没有 trace，就无法区分是模型不会推理、上下文缺失、工具设计差、预算不足，还是 eval 不合理。

### 5. Reasoning 会放大安全问题

Reasoning model 更会规划，也更会使用工具。这会提升能力，也会提升风险：

- 更强的 prompt injection 利用能力。
- 更复杂的越权行动规划。
- 更隐蔽的工具误用。
- 更高成本的 runaway loop。
- 更难发现的错误合理化。

所以 reasoning agent 需要更严格的：

- 权限分层。
- 工具 allowlist。
- 沙箱和网络限制。
- 高风险动作审批。
- cost / token / time budget。
- trace 审计。
- adversarial eval。

### 6. Reasoning 会把软件开发进一步推向 harness engineering

如果模型能更好地“想”，人类工程师的工作不会消失，而是更多变成：

- 设计任务边界。
- 组织上下文。
- 设计工具。
- 设计 eval。
- 设计预算策略。
- 读 trace 和 failure mode。
- 把成功路径产品化。
- 把失败样本回流成测试、文档、prompt、tool 或训练数据。

这正是 harness engineering 的范围。

## 工程落地清单

### 什么时候该用 reasoning model

优先使用：

- 需求模糊、信息不完整，需要澄清和规划。
- 多文档综合、法律金融科研类判断。
- 复杂代码审查、架构设计、debugging。
- 需要多轮工具调用和状态管理的 agent task。
- 高价值任务，延迟和成本可以接受。
- 需要 LLM-as-judge 处理复杂标准。

谨慎使用：

- 简单分类、改写、摘要、格式转换。
- 低延迟聊天、语音交互。
- 明确可用程序计算的问题。
- 大批量低价值任务。
- 没有外部验证的高风险决策。

### 怎么配置 reasoning

建议从低到高调，而不是默认拉满：

| 任务 | 建议 |
|---|---|
| 简单检索、分类、格式化 | `none` / `minimal` / 非 reasoning model |
| 常规编码、数据分析、客服判断 | `low` |
| agentic coding、研究、长文档分析 | `medium` |
| 复杂 debug、代码审计、深度研究 | `high` |
| 异步高价值任务、安全审计、复杂 eval | 只在 eval 证明有收益时使用最高档 |

同时设置：

- `max_output_tokens`。
- 总任务超时。
- 最大工具调用次数。
- 最大成本预算。
- 失败和 incomplete 处理。
- reasoning token / thinking budget 监控。

### 怎么写 prompt

更推荐：

```text
目标是什么
输入是什么
输出契约是什么
硬约束是什么
什么算完成
必须如何验证
遇到哪些情况要停下来问人
```

少依赖：

```text
请一步一步思考
请展示完整思维链
请自己保证正确
```

对 agent 任务，最好明确：

- 先读哪些文件或资料。
- 能用哪些工具。
- 哪些动作需要审批。
- 修改后必须运行哪些验证。
- 最终交付什么 artifact。

### 怎么做 eval

Reasoning eval 至少分三层：

1. **答案层**：最终答案、patch、决策是否正确。
2. **过程层**：工具调用、搜索路径、测试反馈、预算使用是否合理。
3. **系统层**：成本、延迟、失败率、安全事件、人类接管率是否可接受。

记录表应至少包含：

| 字段 | 说明 |
|---|---|
| model | 模型和 snapshot |
| reasoning_budget | effort / thinking budget |
| tools | 是否允许工具和工具版本 |
| harness | agent scaffold、prompt、权限、状态管理 |
| dataset | 任务集和污染控制说明 |
| metric | accuracy、resolved、pass@1、cost、latency |
| failure_mode | 错误分类 |
| trace_link | 可回放 trace |

## 历史类比

### 类比一：从快思考到慢思考

Kahneman 的 System 1 / System 2 常被拿来类比 LLM 和 reasoning model。这个类比有启发，但不能照搬。

有用之处：

- 普通模型像快速直觉，适合常规模式匹配。
- reasoning model 像慢速审议，适合复杂、多步和高风险任务。

危险之处：

- LLM 没有人类心理学意义上的意识和意图。
- 更长输出不等于更深理解。
- 模型的“解释”不一定忠实。

### 类比二：从神经网络直觉到搜索

AlphaGo / AlphaZero 的启发更工程化：

```text
策略网络给候选方向。
搜索在推理时展开可能路径。
价值评估帮助选择。
环境反馈校正判断。
```

LLM reasoning 的对应物是：

```text
模型先验 -> 生成候选方案
test-time thinking -> 搜索和比较
工具调用 -> 获取环境反馈
verifier / tests / judge -> 评价候选
```

这解释了为什么 reasoning 与工具、测试、verifier、trace 会越来越紧密。

### 类比三：数据库查询优化器

Reasoning model 也像查询优化器：它不只是执行一步操作，而是在多条可能执行计划中选择成本和收益更合适的一条。

这个类比提醒我们：

- 需要 cost model。
- 需要执行计划可观测。
- 需要回归测试。
- 需要针对真实 workload 调优。
- 优化器选错计划时，系统会又慢又错。

## 常见误区

### 误区一：Reasoning 越多越好

不是。过高预算可能让简单任务变慢、变贵，还可能引入过度分析和额外工具调用。

### 误区二：能展示推理链就是可信

不是。推理链可能不忠实，最终仍要靠外部验证。

### 误区三：Reasoning model 可以替代 eval

不是。Reasoning model 可以做 judge，但 judge 本身也要校准、抽检和回归。

### 误区四：Benchmark 高就代表真实工作稳

不是。真实工作还依赖上下文、工具、权限、测试、网络、数据、团队流程和失败接管。

### 误区五：开源 reasoning 只意味着更便宜

不只是便宜。开源的更大价值是可检查、可蒸馏、可部署到私有环境、可针对领域任务训练和评估。

## 给本项目的整理框架

后续研究 reasoning，可以按下面结构继续积累：

```text
docs/ai-models/
  reasoning-methods.md        CoT、self-consistency、ToT、ReAct、Reflexion、verifier
  reasoning-models.md         o-series、GPT-5.x、Claude thinking、Gemini thinking、R1、QwQ、Kimi
  reasoning-evals.md          GPQA、AIME、FrontierMath、HLE、ARC、SWE-bench
  reasoning-engineering.md    budget、trace、tool use、prompt、failure mode、cost
  reasoning-safety.md         CoT 监控、prompt injection、越权工具、审计
```

最重要的是不要把 reasoning 写成玄学能力，而是保持下面这个问题链：

```text
任务是什么？
为什么需要 reasoning？
预算是多少？
怎么验证收益？
失败时如何归因？
如何接入 harness？
```

## 参考资料

- [Chain-of-Thought Prompting Elicits Reasoning in Large Language Models](https://arxiv.org/abs/2201.11903)
- [Self-Consistency Improves Chain of Thought Reasoning in Language Models](https://arxiv.org/abs/2203.11171)
- [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)
- [Tree of Thoughts: Deliberate Problem Solving with Large Language Models](https://arxiv.org/abs/2305.10601)
- [Reflexion: Language Agents with Verbal Reinforcement Learning](https://arxiv.org/abs/2303.11366)
- [STaR: Bootstrapping Reasoning With Reasoning](https://arxiv.org/abs/2203.14465)
- [OpenAI: Learning to reason with LLMs](https://openai.com/index/learning-to-reason-with-llms/)
- [OpenAI Reasoning models](https://platform.openai.com/docs/guides/reasoning)
- [OpenAI Reasoning best practices](https://platform.openai.com/docs/guides/reasoning-best-practices)
- [OpenAI: Chain-of-thought monitorability](https://openai.com/index/chain-of-thought-monitoring/)
- [Anthropic: Extended thinking](https://docs.anthropic.com/en/docs/build-with-claude/extended-thinking)
- [Anthropic: Extended thinking tips](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/extended-thinking-tips)
- [Google Gemini API: Thinking](https://ai.google.dev/gemini-api/docs/thinking)
- [Gemini 2.5 Pro: Google's most intelligent AI model](https://blog.google/technology/google-deepmind/gemini-model-thinking-updates-march-2025/)
- [DeepSeek-R1 GitHub](https://github.com/deepseek-ai/DeepSeek-R1)
- [DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning](https://arxiv.org/abs/2501.12948)
- [Hugging Face Open-R1](https://github.com/huggingface/open-r1)
- [GPQA: A Graduate-Level Google-Proof Q&A Benchmark](https://arxiv.org/abs/2311.12022)
- [FrontierMath](https://epoch.ai/frontiermath)
- [Humanity's Last Exam](https://lastexam.ai/)
- [ARC-AGI](https://arcprize.org/arc-agi)
- [SWE-bench](https://www.swebench.com/)
- [LiveCodeBench](https://arxiv.org/abs/2403.07974)
- [The Illusion of Thinking](https://machinelearning.apple.com/research/illusion-of-thinking)
- [Anthropic: An update on recent Claude Code quality reports](https://www.anthropic.com/engineering/april-23-postmortem?pubDate=20260425)
- [OpenAI Codex Core Agent 岗位](https://openai.com/careers/applied-ai-engineer-codex-core-agent-san-francisco/)
