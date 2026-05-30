# 什么时候该新开 Codex 窗口

调研日期：2026-05-29

## 这篇文档回答什么

你问的是一个很实际的问题：

> 当前窗口支持 258k 上下文，我需不需要每完成一个任务就重新开窗口？还是可以一直接着做？长上下文会不会影响模型注意力？

本文把这个问题拆成三层：

1. 长上下文模型是否真的能稳定利用整段上下文。
2. Coding agent 的上下文为什么比普通聊天更容易污染。
3. 在 Codex / Claude Code / Cursor 这类 coding agent 里，什么时候继续当前窗口，什么时候 compact，什么时候新开窗口。

核心结论：

> 不需要每完成一个很小的步骤就新开窗口，但应该把“一个窗口”当成一个相对独立的工作包，而不是整个项目的长期记忆。对同一个目标、同一份计划、同一个验证闭环，可以继续当前窗口；任务边界已经结束、需求切换、上下文里充满失败尝试或无关日志时，最好新开窗口，并用 `AGENTS.md`、docs、计划、测试和 handoff 文件恢复上下文。

如果你看到的 258k 是 **上下文窗口上限**，它意味着这个窗口容量大，不意味着已经快满。可以继续做同一批相关任务。

如果你看到的 258k 是 **当前已占用 token 接近上限**，那就应该先让 agent 总结当前状态、把关键事实写入文件或 final handoff，再开新窗口。

## 先给操作答案

最实用的规则是：

```text
一个窗口 = 一个连续工作包
一个新窗口 = 一个新的任务边界
docs / tests / git diff = 跨窗口记忆
```

### 可以继续当前窗口

这些情况可以继续，不必强迫自己重开：

| 场景 | 为什么可以继续 |
|---|---|
| 还在同一个功能、同一个 bug、同一篇文档里 | 最近上下文仍然是高价值信息 |
| 刚跑完测试或调研，马上要根据结果修改 | 工具输出和中间判断仍然有用 |
| 用户追加的是当前任务的小修正 | 继续比重新解释更省心 |
| agent 对任务状态仍然清楚 | 上下文没有明显污染 |
| 下一步依赖刚刚形成的计划或权衡 | 计划还在短期工作记忆里 |

### 最好新开窗口

这些情况建议重开：

| 场景 | 为什么重开更稳 |
|---|---|
| 一个完整任务已经结束，验证也完成 | 后续任务不需要旧日志和旧探索 |
| 要切换到完全不同主题 | 旧上下文会变成噪声 |
| 你已经纠正 agent 两次以上同一类错误 | 上下文里积累了失败路线和反复纠正 |
| 旧窗口里有大量失败日志、无关文件、调研材料 | 模型可见内容变多，但有效信号比例下降 |
| 任务目标发生了明显变化 | 旧目标会继续影响模型判断 |
| 要做安全、权限、删除、迁移、支付等高风险操作 | 新窗口更容易建立干净边界 |
| 需要 review 上一个任务的结果 | 新窗口视角更独立，不容易被旧解释带偏 |

### 可以用 compact，但不要迷信 compact

Compact 适合：

- 一个长任务还没有完成。
- 当前上下文快满。
- 旧上下文大体是有用的，只是太长。
- 你需要保留连续性，而不是完全切换任务。

Compact 不适合：

- 任务已经完成。
- 旧上下文有大量错误尝试。
- 新任务和旧任务无关。
- 你需要一次独立 review。

一句话：

```text
继续当前窗口保留连续性。
compact 延长当前任务寿命。
新开窗口减少上下文污染。
```

## 为什么 258k 不等于“放心一直聊”

长上下文窗口解决的是“能不能放进去”，不是“模型能不能等质量地使用每个 token”。

对 coding agent 来说尤其明显。一个窗口里会堆进：

- 系统和开发者指令。
- `AGENTS.md`、README、docs。
- 用户需求。
- 文件片段。
- `rg`、`sed`、`git diff`、测试输出。
- 失败日志。
- 中间计划。
- 被否定的方案。
- 最后总结。

这些内容不都是同等价值。旧日志、失败路线、过时目标和无关文件会继续占据模型注意力。窗口越长，越需要管理“什么值得继续留在模型面前”。

所以 258k 更像一个很大的工作台：

```text
工作台大 -> 可以同时摊开更多资料
资料太多 -> 仍然会找错重点
旧资料乱 -> 会干扰当前动作
```

## 论文和技术报告怎么说

### 1. Lost in the Middle：相关信息在中间更容易被忽略

`Lost in the Middle: How Language Models Use Long Contexts` 研究了多文档问答和 key-value retrieval。结论很有名：模型通常更擅长使用输入开头和结尾的信息，相关信息落在长上下文中间时，表现会明显下降。

这对 coding agent 的启发是：

```text
不要假设“我之前说过”就等于 agent 会稳定使用。
关键约束最好放在当前任务 prompt、AGENTS.md、计划或最近的上下文里。
```

### 2. RULER：针找得好，不代表长上下文真的会用

`RULER: What's the Real Context Size of Your Long-Context Language Models?` 指出，很多模型在传统 needle-in-a-haystack 上接近满分，但在多针、聚合、多跳追踪、问答等任务上，随着上下文增长会明显退化。

这说明：

```text
长上下文检索能力 != 长上下文工作能力
能找到某一句话 != 能在大量上下文里稳定做工程判断
```

### 3. NoLiMa：没有字面匹配时，长上下文更难

`NoLiMa: Long-Context Evaluation Beyond Literal Matching` 把问题设计成“问题和答案所在事实之间没有明显字面重合”，要求模型做 latent association。论文报告说，许多宣称支持 128k 以上上下文的模型，在 32k 这类长度时就会出现明显性能下降。

这和代码任务很像。代码任务经常不是简单搜索同一个字符串，而是要把这些东西关联起来：

- 需求里的业务词。
- 测试里的断言。
- 实现里的抽象。
- 文档里的架构约束。
- 错误日志里的症状。

所以，越是需要关联推理，越不应该把无关材料无限塞进同一个窗口。

### 4. Context Rot：更多 token 可能降低可靠性

Chroma 的 `Context Rot` 技术报告把问题说得更直接：现代模型的长上下文表现并不均匀，输入越长不一定越好；内容怎么组织、相关信息放在哪里、无关内容是什么类型，都会影响表现。

这对 coding agent 的操作启发是：

```text
上下文工程不是“把所有东西都给模型”。
上下文工程是“让模型在当前步骤看到最有用的东西”。
```

### 5. 长上下文技术报告仍然支持“能做很多事”

也要避免走向另一个极端。Gemini 1.5、Claude 3、GPT-4.1 等技术报告和产品文章都展示了强大的长上下文能力，尤其是在长文档、代码库、多模态输入和 retrieval 任务上。

所以正确判断不是：

```text
长上下文没用
```

而是：

```text
长上下文很有用，但不是免费的长期记忆，也不是无损注意力。
```

## Coding agent 为什么更容易上下文污染

普通聊天的上下文主要是对话。Coding agent 的上下文是“对话 + 工具轨迹 + 文件系统观测 + 测试反馈 + 中间决策”。

OpenAI 在 Codex agent loop 文章里解释过，Codex 会把用户消息、assistant 输出、工具调用和工具结果组成不断增长的 input item 列表。长任务中，为了避免耗尽上下文窗口，Codex 会触发 compaction，把旧输入替换成更小的上下文窗口，并使用 Responses API 的 `type=compaction` item 继续任务。

Anthropic 在 Claude Code best practices 里给出的建议更像实战经验：

- 在不相关任务之间使用 `/clear`。
- 自动 compact 可以保留重要代码和决策，但长会话里的无关文件、命令和对话会降低表现。
- 如果同一个问题纠正超过两次，最好清空上下文，用吸收了教训的新 prompt 重新开始。
- 调研类任务可以交给 subagent，避免污染主窗口。

Cursor 的 `Dynamic context discovery` 文章也提到，长 shell 输出和 MCP 结果如果直接截断会丢信息。Cursor 的做法是把完整输出写到文件，让 agent 按需读取，从而减少不必要 summarization。

这些实践背后的共同点是：

```text
让大信息留在文件系统。
让当前窗口只保留当前步骤最需要的信息。
```

## 新开窗口和 compact 的本质区别

| 策略 | 本质 | 优点 | 风险 |
|---|---|---|---|
| 继续当前窗口 | 保留完整近期上下文 | 连续性最好，适合同一任务 | 噪声持续累积 |
| compact | 把旧上下文压成工作状态 | 能跨越长任务窗口限制 | 摘要有损，可能漏掉微妙约束 |
| 新开窗口 | 放弃旧会话工作记忆，从文件恢复 | 干净、独立、适合新任务和 review | 如果没有 handoff，会丢中间状态 |

一个成熟工作流不应该只靠其中一种。

更好的组合是：

```text
同一任务短期推进 -> 继续当前窗口
同一任务很长 -> compact 或写 handoff 后继续
任务完成或切换 -> 新开窗口
跨窗口事实 -> 写入 AGENTS.md / docs / tests / git diff
```

## 对你当前 258k 窗口的建议

如果你现在正在围绕同一个主题连续工作，比如：

- 整理 Codex 上下文压缩。
- 写 coding agent playbook。
- 补 harness engineering 文档。
- 调研 agent TDD 工作流。

可以继续当前窗口，因为这些主题彼此相关，旧上下文仍然有价值。

如果你完成一篇文档后，下一步要切到完全不同事情，比如：

- 从写文档切到实现一个新 Web app。
- 从调研 Codex 切到修某个生产项目 bug。
- 从 coding agent 主题切到不相关生活/配置问题。

建议新开窗口。不是因为 258k 不够，而是因为旧上下文会继续影响模型的注意力和判断。

我会用这条个人规则：

```text
小任务：同一窗口连续做。
中任务：一个窗口做完一个 feature / doc / bug。
大任务：用 docs/plans 分 slice，每个 slice 或每 1-2 个 slice 可以新开窗口。
review：尽量新窗口。
主题切换：新窗口。
```

## 推荐的窗口生命周期

### 1. 开始一个任务

让 agent 只读必要入口，不要把所有资料一次性塞满：

```text
请先读取 AGENTS.md、README.md 和当前任务相关 README。
再按需读取 docs/ 下和任务直接相关的文件。
不要一次性读取无关目录。
```

### 2. 任务中

让 agent 把状态外部化：

```text
如果任务会超过一个窗口，请把计划、已完成项、未完成项、验证命令和风险写到 docs/plans/ 或 handoff 文档。
```

对长日志和大文件，优先让 agent 搜索和截取：

```text
先用 rg 找相关位置。
只读取相关片段。
长命令输出先保存或截取关键部分。
```

### 3. 完成一个任务

完成时保留三个东西：

```text
改了什么
验证了什么
下一步是什么
```

如果要马上开始同一主题的小后续，可以继续。

如果要切换任务，开新窗口。

### 4. 新窗口接手

新窗口 prompt 可以这样写：

```text
这是一个从旧窗口切换过来的新会话。请不要依赖旧聊天记录。

请先读取：
1. AGENTS.md
2. README.md
3. docs/coding-agents/README.md
4. 和本任务相关的具体文档

当前任务：

旧窗口已经完成：

还没完成：

关键文件：

验证命令：

请先复述你理解的目标、非目标、预计修改文件和验证方式，然后再开始。
```

## 什么时候应该写 handoff

如果满足任一条件，建议写 handoff：

- 任务还没完成，但你准备换窗口。
- 已经有多个方案被否掉。
- 中间发现了关键坑点。
- 失败日志里有以后还会用到的信息。
- 任务跨越多个文件或多个模块。
- 你希望明天继续做。

handoff 最小模板：

```markdown
# Handoff: <task>

日期：

## 目标

## 已完成

## 未完成

## 关键文件

## 关键决策

## 已否定方案

## 验证命令和结果

## 下一步建议
```

不要把完整聊天粘进去。handoff 要写“状态”，不是写“历史”。

## 判断清单

每次想“要不要新开窗口”时，问这 7 个问题：

| 问题 | 如果答案是 yes |
|---|---|
| 下一个任务和当前任务是否无关？ | 新开窗口 |
| 当前任务是否已经完整验证？ | 新开窗口或 `/clear` |
| 当前窗口里是否有大量失败日志和无关探索？ | 新开窗口 |
| agent 是否已经反复误解同一件事？ | 新开窗口，用更清楚 prompt |
| 下一个任务是否需要独立 review？ | 新开窗口 |
| 关键状态是否已经写入文件？ | 可以放心新开 |
| 新窗口能否通过 AGENTS.md + docs + tests 恢复？ | 可以放心新开 |

如果你还没把关键状态写到文件里，不要直接关闭旧窗口。先让 agent 做一次 handoff。

## 和 AGENTS.md / docs 的关系

窗口管理的目标不是让你记住“哪次聊天说过什么”，而是让项目本身变成可恢复系统。

建议分层：

| 信息 | 放哪里 |
|---|---|
| 永久协作规则 | `AGENTS.md` |
| 主题文档入口 | `README.md` / `docs/*/README.md` |
| 当前任务计划 | `docs/plans/` |
| 当前项目状态 | `docs/project-status.md` 或任务 handoff |
| 设计决策 | `docs/decisions/` |
| 可验证行为 | tests / scripts / CI |
| 临时推理、失败日志 | 当前窗口或临时材料 |

这也是为什么你的项目约定里强调：

```text
AGENTS.md 只放入口和硬约定。
docs/ 承载渐进式上下文。
脚本和测试承载验证闭环。
```

这样做以后，新开窗口的成本会越来越低。

## 常见误区

### 误区 1：窗口越长越好

不一定。长窗口提高容量，但也提高噪声管理难度。

### 误区 2：每个小任务都必须新开

不需要。同一工作包里频繁重开会浪费恢复成本。

### 误区 3：compact 是无损记忆

不是。compact 是有损状态压缩。它适合延续当前任务，不适合当长期事实来源。

### 误区 4：新窗口会丢掉所有能力

不会。只要 `AGENTS.md`、docs、tests、git diff 和 handoff 写得好，新窗口反而更清醒。

### 误区 5：我之前纠正过 agent，它应该一直记得

不一定。重要纠正要写入当前任务 prompt、docs、测试或规则文件。

## 一句话模型

可以这样记：

```text
上下文窗口是工作台，不是档案馆。
compact 是整理工作台，不是备份档案馆。
新窗口是换一张干净工作台。
AGENTS.md、docs、tests 和 git diff 才是跨窗口事实来源。
```

所以，对 258k 窗口的最终建议是：

> 不要因为窗口大就无限续聊，也不要机械地每个小步骤都重开。把窗口边界对齐到“任务边界”：同一任务继续，长任务 compact 或写 handoff，任务切换就新开。

## 参考资料

- Nelson F. Liu et al.: [Lost in the Middle: How Language Models Use Long Contexts](https://aclanthology.org/2024.tacl-1.9/)
- Cheng-Ping Hsieh et al.: [RULER: What's the Real Context Size of Your Long-Context Language Models?](https://arxiv.org/abs/2404.06654)
- Ali Modarressi et al.: [NoLiMa: Long-Context Evaluation Beyond Literal Matching](https://arxiv.org/abs/2502.05167)
- Kelly Hong, Anton Troynikov, Jeff Huber: [Context Rot: How Increasing Input Tokens Impacts LLM Performance](https://www.trychroma.com/research/context-rot)
- Google DeepMind: [Gemini 1.5: Unlocking multimodal understanding across millions of tokens of context](https://arxiv.org/abs/2403.05530)
- OpenAI: [Introducing GPT-4.1 in the API](https://openai.com/index/gpt-4-1/)
- OpenAI: [Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop/)
- OpenAI API: [Compaction](https://developers.openai.com/api/docs/guides/compaction)
- OpenAI API: [Conversation state](https://developers.openai.com/api/docs/guides/conversation-state)
- Anthropic: [Best practices for Claude Code](https://www.anthropic.com/engineering/claude-code-best-practices)
- Anthropic: [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- Anthropic: [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- Cursor: [Dynamic context discovery](https://cursor.com/blog/dynamic-context-discovery)
- HumanLayer: [12 Factor Agents](https://www.humanlayer.dev/blog/12-factor-agents)
