# Codex 上下文压缩原理详解

调研日期：2026-05-29

## 这篇文档回答什么

这里的 **Codex 上下文压缩** 指 Codex / Codex CLI 在长时间编码任务中，为了避免上下文窗口被历史对话、工具调用、命令输出、文件内容和推理状态填满，而对会话状态进行 **compaction** 的机制。

本文回答：

1. 为什么 coding agent 会需要上下文压缩？
2. Codex 的上下文是怎么增长的？
3. 早期“总结式压缩”和现在的 Responses API 原生 compaction 有什么区别？
4. `/responses/compact` 返回的 `type=compaction` / `encrypted_content` 到底是什么？
5. compaction、prompt caching、conversation state、docs 外部记忆有什么区别？
6. 上下文压缩会丢什么、保留什么、有什么风险？
7. 作为使用 Codex 的开发者，应该怎样设计 `AGENTS.md` 和 `docs/` 来配合它？

核心结论：

> Codex 的上下文压缩不是把聊天记录做一个普通摘要，也不是把原文像 zip 一样无损压缩。它更像给长任务创建一个模型可继续使用的“工作记忆检查点”：把已经发生的长会话转成更小的上下文窗口，其中包括若干保留的高价值 item，以及一个不透明的 `type=compaction` 加密状态项。这样 agent 可以跨越上下文窗口边界继续工作，但人类不应该把它当成可靠、完整、可审计的事实来源。长期事实仍然应该写进代码、测试、`AGENTS.md`、`docs/`、计划和状态文件。

## 来源说明

本文主要依据官方公开资料和本地可观察配置：

| 来源 | 类型 | 用途 |
|---|---|---|
| OpenAI: Unrolling the Codex agent loop | 官方工程文章 | 解释 Codex agent loop、输入增长、早期 `/compact` 和现在 `/responses/compact` |
| OpenAI API: Compaction guide | 官方 API 文档 | 解释 server-side compaction、standalone compact endpoint、compaction item |
| OpenAI API: Responses compact reference | 官方 API 参考 | 解释 `/v1/responses/compact` 请求和返回结构 |
| OpenAI API: Conversation state guide | 官方 API 文档 | 解释 context window、input/output/reasoning tokens 和 conversation state |
| OpenAI: Responses API computer environment | 官方工程文章 | 解释 agent loop 中 shell/tool 输出如何填满上下文，以及原生 compaction 的动机 |
| 本机 `~/.codex/config.toml` | 本地观察 | 可见 `model_auto_compact_token_limit`、`model_context_window` 等字段，但具体语义以当前 Codex 版本为准 |

需要区分两类内容：

- **官方事实**：文档明确说明的行为，例如 `/responses/compact`、`type=compaction`、`encrypted_content`、`context_management`、`compact_threshold`。
- **本文推断**：从 agent harness 设计角度推断 Codex 可能保留哪些任务信息、丢弃哪些低价值信息。推断会明确标注，不当成官方内部实现。

## 先说结论：上下文压缩是什么

可以把一次长 Codex 会话想成一个不断增长的数组：

```text
input items =
  system/developer instructions
  + tool definitions
  + user messages
  + assistant messages
  + tool calls
  + tool outputs
  + file snippets
  + command logs
  + reasoning summaries
  + screenshots / images / structured artifacts
```

每次模型调用都要把必要上下文发给模型。任务越长，这个数组越大。大到一定程度后，会遇到三个问题：

1. **上下文窗口放不下**：模型有固定 context window。
2. **成本和延迟上升**：每轮都带大量历史，输入 token 变多。
3. **注意力被稀释**：模型虽然“看见”很多内容，但不一定能用好。

上下文压缩做的事是：

```text
长上下文窗口 W0
-> compaction pass
-> 更小的上下文窗口 W1
-> 后续对话用 W1 继续，而不是继续携带完整 W0
```

在当前 Responses API 里，W1 通常不只是一个人类可读摘要，而是：

```text
保留的高价值 messages/items
+ type=compaction 的 opaque encrypted_content
```

也就是说，Codex 后续不是靠“读一段中文/英文摘要”继续，而是靠一个模型/API 能使用的压缩状态继续。

## 为什么 coding agent 特别需要压缩

普通聊天也会长，但 coding agent 的上下文增长更快。

因为 coding agent 不只是对话，它还会：

- 读 `AGENTS.md`、README、源码、配置和测试。
- 搜索代码库。
- 调用 shell。
- 跑 formatter、lint、typecheck、test。
- 读取失败日志。
- 修改文件。
- 看 diff。
- 使用浏览器、截图或 MCP。
- 产生计划、TODO、审查和总结。

一个典型长任务可能长这样：

```text
用户需求
-> 读 AGENTS.md
-> 读 README 和 docs
-> rg 搜索代码
-> 读 20 个文件片段
-> 写测试
-> 跑测试，失败输出 300 行
-> 修改实现
-> 跑 lint，失败输出 200 行
-> 修改实现
-> 跑测试，通过
-> 看 diff
-> 用户追加要求
-> 继续下一轮
```

这些 item 如果全都一直留在上下文里，窗口很快就会被填满。

OpenAI 的 Responses API computer environment 文章也指出，长任务里 agent 会调用 skills、加入工具结果和 reasoning summaries，context window 会很快被占满，因此需要保留关键细节、移除多余内容。

## Codex agent loop 如何让上下文增长

官方的 Codex agent loop 文章解释了 Codex harness 的核心逻辑：它负责组织用户、模型和工具之间的循环。

一个简化版 agent loop：

```text
User asks
-> Codex sends input items to model
-> model emits tool call
-> Codex runs tool
-> Codex appends tool output
-> model sees output and emits next tool call
-> ...
-> model emits final answer
-> user replies
-> Codex appends assistant answer and new user message
-> next turn starts
```

关键是：如果使用 stateless input-array chaining，下一次请求要把历史 input / output items 继续带上。官方文章里明确提到，因为继续对话时要追加上一轮 assistant message 和新的 user message，所以发送给 Responses API 的 `input` 会持续增长。

这也是为什么长任务不做压缩会越来越重。

## 为什么不只用 `previous_response_id`

Responses API 支持 `previous_response_id` 这种方式来延续前一次 response，不必每次都把完整 input array 重新发上来。

但 Codex 官方文章说，Codex 当时没有使用 `previous_response_id`，主要为了保持请求 stateless，并支持 Zero Data Retention（ZDR）配置。换句话说，Codex 更倾向于让每次请求都携带所需状态，而不是依赖服务端保存完整前文。

这带来一个后果：

```text
每次请求都更自包含
但 input 会不断增长
所以需要 compaction
```

这也是 Codex 上下文压缩的工程背景。

## 早期方案：手动 `/compact` + 普通总结

Codex 早期的 compaction 实现比较接近普通总结：

```text
用户手动调用 /compact
-> Codex 用现有 conversation + summarization instructions 调用 Responses API
-> 模型生成一条 assistant summary message
-> 后续 conversation 把这个 summary 当作新的 input
```

这个方案直观，但有明显限制：

1. **依赖用户手动触发**：太早压缩会浪费，太晚可能已经接近窗口上限。
2. **依赖总结 prompt**：总结质量受 prompt 和模型当时状态影响。
3. **只得到人类可读 summary**：它是自然语言摘要，不一定保留模型继续工作需要的隐藏状态。
4. **容易丢细节**：特别是文件路径、未完成 TODO、失败命令、约束和边界。
5. **不可持续维护**：不同 agent loop 都要自己设计 summarization system。

这种方式更像：

```text
把长会话缩成一段项目进度摘要
```

它能工作，但不是最适合长时间工具调用型 agent 的状态表示。

## 当前方案：Responses API 原生 compaction

现在 OpenAI 在 Responses API 里提供了原生 compaction。

官方资料里有两种使用方式：

1. **Server-side compaction**：在 `/responses` 请求里配置 `context_management` 和 `compact_threshold`，由服务端在超过阈值时自动触发。
2. **Standalone compact endpoint**：显式调用 `/responses/compact`，把当前完整上下文窗口发给 endpoint，得到一个新的 compacted window。

Codex 官方文章说明：Codex 现在会在超过 `auto_compact_limit` 时自动使用 `/responses/compact` 来压缩 conversation。

压缩后的结果不是一条普通 summary message，而是一组 output items，其中包含一个特殊 item：

```json
{
  "type": "compaction",
  "encrypted_content": "gAAAAABpM0Yj-...="
}
```

这个 item 的特点是：

- `type` 固定为 `compaction`。
- `encrypted_content` 是加密内容。
- 对人类和客户端来说是 opaque，不应该尝试解释。
- 用来把先前上下文中的关键状态和 reasoning 以更省 token 的方式带到下一轮。

官方 API 文档明确说：compaction item 不 intended to be human-interpretable。也就是说，它不是给你读的，而是给后续模型/API 继续工作的。

## 压缩前后发生了什么

简化流程如下：

```text
1. 当前窗口 W0 已经很长

W0 = [
  user request,
  AGENTS.md context,
  read file outputs,
  shell command outputs,
  assistant tool calls,
  test failures,
  code edits,
  current plan,
  ...
]

2. 触发 compaction

W1 = responses.compact(W0)

3. 得到更小窗口

W1 = [
  selected high-value items,
  { type: "compaction", encrypted_content: "..." }
]

4. 下一轮继续

next_input = [
  ...W1,
  new user message
]
```

Standalone compact endpoint 的文档强调：返回的 compacted window 是下一轮的 canonical context window，应该原样传给下一次 `/responses` 调用，不要再手动删它。

Server-side compaction 的文档则说明：如果你用 stateless input-array chaining，追加 output items 后，可以丢弃最近一次 compaction item 之前的旧 items，以减小请求和延迟；最新 compaction item 承载继续所需的上下文。

## Server-side compaction

Server-side compaction 是把压缩逻辑交给 Responses API：

```python
response = client.responses.create(
    model="gpt-5.3-codex",
    input=conversation,
    store=False,
    context_management=[
        {"type": "compaction", "compact_threshold": 200000}
    ],
)
```

它的工作方式：

1. 客户端正常调用 `/responses`。
2. 请求里包含 `context_management`。
3. 当 rendered token count 超过 `compact_threshold`，服务端触发 compaction。
4. response stream 里会出现 encrypted compaction item。
5. 后续 loop 继续携带这个 compaction item。

这个模式的优点：

- 客户端不需要自己判断什么时候压缩。
- 不需要额外调用 `/responses/compact`。
- 更适合长时间 agent loop。
- 文档说在 `store=false` 时对 ZDR 友好。

可以把它理解成：

```text
自动 checkpoint
```

## Standalone compact endpoint

Standalone 方式更显式：

```python
compacted = client.responses.compact(
    model="gpt-5.5",
    input=long_input_items_array,
)

next_input = [
    *compacted.output,
    {
        "type": "message",
        "role": "user",
        "content": user_input_message(),
    },
]
```

它的工作方式：

1. 你先积累一段长 conversation window。
2. 在它还放得进模型 context window 时，调用 `/responses/compact`。
3. endpoint 返回 compacted output。
4. 下一轮把 compacted output 作为新的上下文基础。

这个模式适合你想自己控制 compaction timing 的 agent harness。

需要注意：你发给 `/responses/compact` 的窗口本身仍然必须放得进模型上下文。如果已经超过窗口再想压缩，可能已经太晚。

可以把它理解成：

```text
手动 checkpoint
```

## `encrypted_content` 是什么

这是最容易误解的部分。

`encrypted_content` 不是：

- 可读 Markdown 摘要。
- 可解压的 zip 文件。
- 可让用户还原原文的压缩包。
- 可以复制到其他任意模型或工具里的通用记忆格式。

它更接近：

```text
OpenAI Responses API / 模型可使用的加密状态表示
```

官方文章说，这个 opaque encrypted item 用来保留模型对原始 conversation 的 latent understanding。另一篇 Responses API computer environment 文章说明，最新模型被训练为分析先前会话状态，并产出一个 token-efficient 的 encrypted representation，用于跨窗口继续工作。

用直觉类比：

```text
普通总结：
  给人看的项目摘要。

compaction item：
  给模型/API 下次继续工作的加密工作记忆。
```

这个设计有两个重要效果：

1. **更省 token**：不再把大量原始历史逐字放回上下文。
2. **更贴近模型训练**：模型被训练去生成和使用这种压缩状态，而不是只依赖通用自然语言总结。

但它也带来一个限制：

```text
人类不能审计 compaction item 里到底保留了什么。
```

所以不能把它当成项目事实来源。

## 它保留什么，丢弃什么

官方不会逐项告诉我们具体保留策略，因为 compaction 逻辑可能随模型演进而变化。下面是从 agent harness 角度的合理推断。

它应该尽量保留：

- 当前用户目标。
- 明确约束和禁止事项。
- 当前计划和未完成 TODO。
- 重要文件路径。
- 已做出的关键决策。
- 最近的工具结果。
- 关键测试失败或通过状态。
- 当前错误根因或调试假设。
- 安全、权限、sandbox、approval 相关上下文。
- 继续执行下一步所需的 reasoning state。

它通常会倾向丢弃或压缩：

- 过长的 shell 输出。
- 重复搜索结果。
- 已经过时的失败尝试。
- 大量文件原文。
- 不再相关的中间推理。
- 多轮重复确认。
- 低价值日志。
- 很旧的对话措辞细节。

注意，这里说的是“倾向”，不是官方保证。

## 它是有损还是无损

从用户可理解角度看，compaction 是 **有损的**。

它不会保证：

- 原始对话逐字可恢复。
- 所有命令输出都完整存在。
- 每个文件片段都可重新展开。
- 每个旧约束都不会被弱化。
- 每个失败路径都被记住。

但从 agent 工作角度看，它的目标不是无损存档，而是保留足够状态，让后续任务可以继续。

更准确地说：

```text
compaction 优化的是“继续工作所需的信息密度”，不是“历史记录完整性”。
```

这也是为什么长期项目知识要写进仓库文件，而不是只相信会话记忆。

## compaction 和普通总结的区别

| 维度 | 普通总结 | Codex / Responses compaction |
|---|---|---|
| 输出形态 | assistant message / text summary | compacted output items + `type=compaction` |
| 是否可读 | 可读 | `encrypted_content` 不可读 |
| 是否可审计 | 可以人工检查摘要 | 只能检查保留的可见 items，不能审计 encrypted content |
| 是否模型训练对齐 | 依赖普通总结能力 | 官方称原生 compaction 随模型训练演进 |
| 状态承载 | 自然语言 | 加密 token-efficient representation |
| 使用方式 | 把 summary 作为新 input | 把 compacted window 作为后续 input |
| 典型风险 | 摘要漏细节 | 状态不可见、仍可能漏细节 |

普通总结适合给人接手。

原生 compaction 适合给模型继续跑。

两者不是互相替代。好的 agent harness 往往同时需要：

```text
人类可读 project-status / handoff summary
+ 模型可用 compaction item
```

## compaction 和 prompt caching 的区别

这两个概念经常一起出现，但完全不同。

| 概念 | 解决什么 | 怎么解决 |
|---|---|---|
| Prompt caching | 重复前缀带来的计算成本 | 复用相同 prompt prefix 的计算 |
| Compaction | 上下文窗口太大 | 把旧上下文转成更小的 compacted window |

Prompt caching 不会缩短上下文。它只是当下一次请求和上一次请求有相同前缀时，让系统复用计算。

Codex 官方文章里提到，精确 prefix match 对缓存很重要，因此静态内容应放在 prompt 前部，变量内容放后面。文章还指出，改变可用 tools、改变 model、改变 sandbox 配置、approval mode 或 current working directory，都可能导致 cache miss。

Compaction 则是另一件事：它会改变上下文窗口本身，把旧 items 替换成 compacted state 和少量保留 items。

可以这样记：

```text
prompt caching = 同样内容，算得更便宜。
compaction = 内容太多，换成更小状态。
```

## compaction 和 conversation state 的区别

Conversation state 是“怎么保存或延续对话”。

Responses API 有几种方式：

- 客户端自己维护 input array，每轮把需要的 items 发上去。
- 使用 `previous_response_id`。
- 使用 Conversations API 创建持久 conversation object。

Compaction 是“当状态太大时，如何缩小它”。

二者关系：

```text
conversation state 决定状态怎么被携带。
compaction 决定状态太大时怎么被压缩。
```

Codex 官方文章强调它倾向 stateless request，以支持 ZDR 场景。这种选择让 compaction 更重要。

## compaction 和 `AGENTS.md` / `docs/` 的区别

这是对你最有用的一点。

Compaction 是会话内的模型工作记忆。`AGENTS.md` 和 `docs/` 是项目里的长期事实来源。

| 信息 | 应该放哪里 |
|---|---|
| 当前会话临时推理 | compaction / conversation state |
| 项目长期规则 | `AGENTS.md` |
| 开发命令和验证方式 | `docs/development.md` / `docs/testing.md` |
| 当前目标和进度 | `docs/project-status.md` |
| 架构边界 | `docs/architecture/` |
| 重要技术决策 | `docs/decisions/` |
| 实施计划 | `docs/plans/` |
| 失败经验 | `docs/troubleshooting.md` |

不要把 compaction 当成“项目记忆”。它是自动机制，不是你可审计、可编辑、可版本化的文档。

最稳的做法是：

```text
让 compaction 帮 Codex 跨窗口继续工作；
让 docs 帮人和新会话准确恢复事实。
```

## 与上下文压缩相关的另一件事：工具输出截断

上下文压缩不是唯一的 context management。

OpenAI 的 computer environment 文章还提到，shell 输出可能很大，所以模型可以指定每个命令的 output cap，Responses API 会强制这个 cap，并返回有界结果，保留开头和结尾，同时标记中间省略。

这和 compaction 是两层控制：

| 层级 | 作用 |
|---|---|
| Tool output bounding | 单次工具输出不要撑爆上下文 |
| Compaction | 整个长会话不要撑爆上下文 |

对使用 Codex 来说，这意味着：

- 不要让命令无意义地输出海量日志。
- 需要完整日志时，把日志写到文件，再让 Codex 针对性读取。
- 使用 `rg`、`sed -n`、测试过滤器等缩小输出。
- 让重要结论进入 docs，而不是依赖终端输出长期留在上下文里。

## 一个具体例子

假设你让 Codex 做一个 TDD 功能：

```text
任务：实现用户注册
```

压缩前上下文可能包含：

```text
1. 用户需求
2. AGENTS.md
3. docs/testing.md
4. 读取的路由文件
5. 读取的模型文件
6. 读取的数据库迁移
7. 搜索输出
8. 新增失败测试
9. pytest 失败日志
10. 修改实现
11. pytest 通过日志
12. lint 失败日志
13. 修复 lint
14. git diff
15. 用户追加“加邮箱唯一性校验”
```

压缩后，下一轮可能只需要：

```text
1. 当前任务目标：用户注册 + 邮箱唯一性校验
2. 关键约定：TDD、完成前跑 scripts/check
3. 关键路径：src/auth/..., tests/auth/...
4. 当前状态：注册测试已通过，lint 已修复
5. 未完成事项：补唯一性测试和实现
6. type=compaction encrypted_content
7. 新用户消息
```

原始 pytest 输出的每一行不再重要。重要的是：

```text
测试曾因什么失败，现在是否通过，下一步是什么。
```

这就是 compaction 想保留的工作状态。

## 为什么压缩后有时会“失忆”或“跑偏”

即使原生 compaction 比普通总结更适合 agent，它也不是魔法。

可能出现的问题：

### 1. 细节没有被认为重要

某个边界条件、文件路径或用户口头偏好如果没有显著进入任务主线，可能被弱化。

解决：

- 把长期偏好写入 `AGENTS.md`。
- 把当前计划写入 `docs/plans/`。
- 把当前状态写入 `docs/project-status.md`。

### 2. 错误假设被压缩进状态

如果压缩前 agent 已经形成错误理解，compaction 可能把这个错误理解延续下去。

解决：

- 在关键节点让 Codex 复述当前理解。
- 发现方向错时明确纠正，并更新计划文件。

### 3. 原始证据丢失

压缩后可能不再保留完整命令输出或日志。

解决：

- 重要日志写到文件。
- 在 `project-status.md` 记录验证命令和结果。
- 对 bug 修复写回归测试，而不是靠日志记忆。

### 4. 人类无法审计 encrypted content

你看不到 compaction item 里到底有什么。

解决：

- 人类可读状态必须写进 docs。
- 完成前必须重新运行验证。
- 不用 compaction 作为验收证据。

### 5. 压缩触发太晚

Standalone compaction 要求发给 `/responses/compact` 的窗口仍然放得进模型上下文。如果太晚，可能已经无法压缩。

解决：

- 使用 server-side compaction threshold。
- 在自建 harness 里提前触发。
- 控制工具输出，避免一次性爆掉窗口。

## 对你使用 Codex 的实践建议

### 1. `AGENTS.md` 保持短

`AGENTS.md` 应该是入口地图，不是大百科。

推荐只放：

- 必读顺序。
- 硬性开发约定。
- TDD / 验证要求。
- 禁止事项。
- docs 写入规则。

原因：

```text
AGENTS.md 越短、越稳定，越适合每次进入上下文；
长期知识越结构化，越不依赖 compaction 是否记住。
```

### 2. 把长期事实写入 `docs/`

如果某件事下次还需要知道，就不要只留在会话里。

写入位置：

- 当前进度：`docs/project-status.md`
- 开发命令：`docs/development.md`
- 测试策略：`docs/testing.md`
- 架构边界：`docs/architecture/`
- 重要决策：`docs/decisions/`
- 计划：`docs/plans/`
- 失败经验：`docs/troubleshooting.md`

这能抵抗 compaction 丢细节。

### 3. 让 Codex 经常更新状态文件

长任务中可以要求：

```text
完成一个阶段后，更新 docs/project-status.md；
如果计划变化，更新 docs/plans/...；
如果遇到新失败，更新 docs/troubleshooting.md。
```

这相当于人为创建可审计 checkpoint。

### 4. 控制工具输出

避免：

```bash
cat huge.log
npm test -- --verbose
grep -R ...
find . -type f
```

优先：

```bash
rg "pattern" path
sed -n '1,160p' file
pytest tests/auth/test_register.py -q
npm test -- --runInBand specific.test.ts
```

如果输出很大，先写入文件，然后让 Codex 读取关键片段。

### 5. 关键阶段让 Codex 复述状态

在长任务中，尤其是压缩后，可以让 Codex 做一次状态校准：

```text
请先根据当前上下文和 docs/project-status.md，复述：
1. 当前目标
2. 已完成事项
3. 未完成事项
4. 下一步验证命令
不要改代码。
```

这能及时发现压缩后的理解偏差。

### 6. 完成声明必须靠新验证

不要相信“记忆中测试通过”。

完成前必须重新运行：

```bash
./scripts/test
./scripts/check
```

或者当前项目对应的最小验证命令。

压缩后保留的是工作状态，不是最终证据。

## 对自建 agent harness 的启发

如果你以后自己写 agent harness，可以把状态分成四层：

| 层级 | 例子 | 作用 |
|---|---|---|
| L1 模型工作记忆 | compaction item | 让模型跨窗口继续 |
| L2 人类可读状态 | project-status、plan、handoff | 让人和新会话接手 |
| L3 可验证事实 | tests、lint、CI、diff | 证明任务是否完成 |
| L4 外部资源 | 文件系统、数据库、日志、artifacts | 存放大信息，不塞 prompt |

成熟 harness 不应该只依赖 L1。

更好的结构是：

```text
compaction 负责“模型不断片”
docs 负责“人类可审计”
tests/CI 负责“完成可证明”
filesystem 负责“大上下文按需读取”
```

这和你前面整理的 harness engineering 思想完全一致：上下文不是越多越好，而是要分层、可控、可验证。

## 常见误解

### 误解 1：上下文压缩就是把聊天记录总结一下

早期实现接近这个，但当前 Responses API 原生 compaction 不只是普通文本总结。它会产生 `type=compaction` 的加密不透明状态项。

### 误解 2：压缩是无损的

不是。它的目标是继续任务，而不是还原历史。

### 误解 3：有了大 context window 就不需要压缩

大窗口仍然会被长任务填满，而且成本、延迟和注意力稀释仍然存在。长任务需要 context management，不只是更大窗口。

### 误解 4：compaction item 可以给人类检查

不能。`encrypted_content` 是 opaque，不 intended to be human-interpretable。

### 误解 5：compaction 可以替代 `docs/`

不能。compaction 是会话工作记忆，`docs/` 是长期事实来源。

### 误解 6：prompt caching 和 compaction 是同一件事

不是。prompt caching 省计算，compaction 省上下文。

## 一句话模型

可以用这个心智模型理解 Codex 上下文压缩：

```text
Codex 长任务 = 不断增长的工具化会话
context window = 有限工作台
tool output bounding = 控制每次工具别把工作台堆满
prompt caching = 同样前缀少算一点
compaction = 把旧工作台整理成模型可继续用的检查点
docs/tests/git = 人类可审计、可验证、可恢复的长期事实来源
```

所以，真正可靠的做法不是问“Codex 会不会记住”，而是：

```text
该让模型临时记住的，交给 compaction。
该让项目长期记住的，写进文件。
该证明真的完成的，交给测试和验证命令。
```

## 参考资料

- OpenAI: [Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop/)
- OpenAI API: [Compaction](https://developers.openai.com/api/docs/guides/compaction)
- OpenAI API Reference: [Compact a response](https://developers.openai.com/api/reference/resources/responses/methods/compact)
- OpenAI API: [Conversation state](https://developers.openai.com/api/docs/guides/conversation-state)
- OpenAI: [From model to agent: Equipping the Responses API with a computer environment](https://openai.com/index/equip-responses-api-computer-environment/)
