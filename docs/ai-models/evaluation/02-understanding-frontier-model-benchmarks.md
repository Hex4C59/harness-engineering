# 如何读懂 Claude、GPT、Gemini 发布时的 Benchmark

## 核心结论

模型发布页里的 benchmark 不是“智商考试排行榜”，而是一组不同形状的评测：

```text
Benchmark = 任务集 + 输入格式 + 运行协议 + 评分器 + 模型配置
```

同一个模型在不同 benchmark 上的分数，回答的是不同问题：

- `MMLU / MMLU-Pro`：会不会做多学科选择题。
- `GPQA Diamond`：会不会做研究生级科学推理题。
- `AIME`：会不会做竞赛数学。
- `HumanEval / MBPP / LiveCodeBench`：会不会生成可运行代码。
- `SWE-bench / Terminal-Bench / Aider Polyglot`：会不会像 coding agent 一样修真实任务。
- `MMMU / VideoMME / CharXiv`：会不会看图、看视频、读图表并推理。
- `IFEval / MultiChallenge`：会不会严格遵守复杂指令。
- `τ-bench / τ²-bench / BFCL`：会不会使用工具、遵守业务规则、和用户多轮协作。
- `MRCR / LongBench / LOFT`：长上下文是不是真的能用。
- `SimpleQA / TruthfulQA`：回答事实问题时会不会胡说。
- `LMArena / Chatbot Arena`：人类更喜欢哪个模型的回答。

读 benchmark 的关键不是记住谁第一，而是看清：

> 这个分数到底来自什么任务、什么数据、什么工具、什么提示词、什么采样次数、什么评分器？

截至 2026-05-29，Claude、GPT、Gemini 的官方发布材料越来越多地把 benchmark 分成：推理、数学、代码、多模态、长上下文、指令遵循、工具使用、安全和人类偏好。真正有用的是按能力维度读，而不是把所有分数混成一个“模型强弱”。

## 先搞清楚 Benchmark 是什么

一个 benchmark 通常包含五件东西：

| 部分 | 例子 |
|---|---|
| 任务集 | 500 个 GitHub issue、198 道 GPQA Diamond 题、225 个 Aider Polyglot 代码编辑题 |
| 输入格式 | 选择题、自然语言题目、代码仓库 + issue、图片 + 问题、长文档 + 查询 |
| 运行协议 | 是否允许工具、是否多次采样、是否有 thinking budget、是否联网、是否用 agent scaffold |
| 评分器 | 标准答案、单元测试、隐藏测试、LLM judge、人类偏好、Elo |
| 报告指标 | accuracy、pass@1、pass@k、resolved rate、Elo、cost per task、latency |

所以看到一个表格时，要先问：

1. 这是裸模型还是完整 agent？
2. 有没有使用工具，例如 Python、浏览器、shell、代码编辑器？
3. 是单次回答，还是多次采样后挑最好？
4. 是公开测试集，还是私有 holdout？
5. 是自动评分，还是 LLM judge 或人类投票？
6. 分数有没有报告成本、时间、token、失败率？

如果这些没写清楚，不同模型之间就不一定可比。

## 常见分数是什么意思

| 指标 | 含义 | 常见场景 |
|---|---|---|
| `accuracy` | 答对比例 | MMLU、GPQA、MMMU、AIME |
| `pass@1` | 单次生成就通过的比例 | HumanEval、MBPP、LiveCodeBench |
| `pass@k` | 生成 k 次，只要有一次通过就算过 | 代码生成、数学采样 |
| `% resolved` | 成功解决任务比例 | SWE-bench、Terminal-Bench |
| `Elo` | 基于对战或偏好比较的相对评分 | Codeforces、LMArena |
| `win rate` | 人类或 judge 更偏好某模型的比例 | 聊天、写作、arena |
| `exact match` | 输出必须和标准答案匹配 | 短答案、检索、事实问答 |
| `F1` | 部分匹配得分 | 抽取、问答 |
| `cost / success` | 每完成一个成功任务花多少钱 | agent benchmark |
| `latency` | 响应或完成任务耗时 | 产品可用性 |

`pass@k` 特别容易误读。`pass@1` 更像“日常一次就能不能成”，`pass@100` 更像“如果我愿意让模型试很多次，它有没有可能找到答案”。模型发布页如果只写高 k 分数，就不能直接理解成日常稳定性。

## 三大厂发布时通常比较什么

### OpenAI / GPT 系列

OpenAI 近年的发布材料经常突出：

- 推理和数学：`AIME`、`GPQA Diamond`、`HLE`。
- 代码：`SWE-bench Verified`、`Aider Polyglot`、有时也会提 `LiveCodeBench`。
- 多模态：`MMMU`、`CharXiv`、视频或图像理解评测。
- 长上下文：`MRCR`、needle-in-haystack 风格任务、内部长上下文 eval。
- 指令遵循：`IFEval`、`MultiChallenge`、内部 instruction-following eval。
- 事实性：`SimpleQA`。
- 安全和风险：system card 里的 cyber、bio、autonomy、jailbreak 等安全 eval。
- 垂直领域：例如 GPT-5 发布材料里的 `HealthBench`。

例如 GPT-4.1 发布页强调 coding、instruction following 和 long context；GPT-5 发布页强调 AIME、SWE-bench、Aider Polyglot、MMMU、HealthBench、GPQA 等。

### Anthropic / Claude 系列

Anthropic 的系统卡和发布材料常见维度：

- 代码和 agentic coding：`SWE-bench Verified`、`Terminal-Bench`、有时也包括 Aider 或内部 coding eval。
- 推理和知识：`MMLU`、`MMMLU`、`GPQA Diamond`、`AIME`。
- 多模态：`MMMU` 等视觉理解指标。
- 工具和计算机使用：`τ-bench`、computer-use、tool-use、browser/desktop 操作类内部和外部 eval。
- 长任务能力：METR 风格的 task horizon、内部研究/工程使用调查。
- 安全：ASL、jailbreak、cyber、bio、agent autonomy 等系统卡评估。

Claude 的发布叙事尤其常把“真实工作流”和“agent 长任务”放在前面，因为 Claude Code 和企业 agent 场景是它的强卖点。

Claude 4 发布材料还清楚标注了哪些 benchmark 使用 extended thinking，哪些没有使用。比如 SWE-bench Verified、Terminal-Bench 通常按 no extended thinking 报告，而 GPQA、MMMU、AIME、TAU-bench 等可能使用 extended thinking。这类脚注非常关键。

### Google / Gemini 系列

Gemini 发布材料经常突出：

- 多模态：`MMMU`、`MMMU-Pro`、`VideoMME`、图像/视频/音频任务。
- 长上下文：needle-in-haystack、`MRCR`、`LOFT`、百万 token 级上下文测试。
- 推理和数学：`AIME`、`GPQA Diamond`、`HLE`。
- 代码：`LiveCodeBench`、`SWE-bench Verified`、有时也会提 Aider 或 agentic coding。
- 人类偏好：LMArena / Chatbot Arena 排名。
- 产品维度：速度、成本、context window、thinking budget。

Gemini 的特色是多模态和长上下文经常被放在很突出的位置。Gemini 1.5 时代重点是百万 token context，Gemini 2.5 以后则更强调 thinking、AIME/GPQA、SWE-bench 和 VideoMME。

Gemini 2.5 Pro 发布材料里还会明确说某些结果没有使用 majority voting 这类 test-time 技巧；SWE-bench 则会说明是 custom agent setup。读 Gemini 表格时也要把 `thinking budget`、`with tools`、`custom agent` 这些条件单独看。

## General Knowledge / Academic Reasoning

### MMLU

全称是 **Massive Multitask Language Understanding**。它是最经典的通用知识选择题 benchmark。

数据长这样：

```text
Question: Which theorem best explains ...
A. ...
B. ...
C. ...
D. ...
Answer: B
```

它覆盖 57 个学科，包括：

- 数学。
- 物理。
- 法律。
- 医学。
- 历史。
- 经济。
- 计算机。
- 道德推理。

评分通常是 accuracy。答对就是 1，答错就是 0。

MMLU 适合回答：

> 模型是否有广泛的学科知识和基本推理能力？

局限：

- 已经很老，强模型分数接近饱和。
- 是选择题，不代表真实工作能力。
- 部分题目有错误或争议。
- 很可能进入训练语料，需要关注污染。

### MMLU-Pro

MMLU-Pro 是更难的版本。它把很多题从 4 个选项扩展到 10 个选项，并增加更需要推理的问题。

它想解决的问题是：

> 原始 MMLU 对强模型太容易，区分度不够。

MMLU-Pro 更适合看强模型之间的差距，但它仍然是闭卷选择题，不等于真实 agent 能力。

### MMMLU

MMMLU 是 multilingual MMLU，测多语言学科问答。Anthropic、Google 等发布材料有时会报告 MMMLU，用来说明模型在非英语学科问题上的能力。

它适合回答：

> 模型的学科知识和推理能力是否能迁移到多语言环境？

局限：

- 仍然是选择题。
- 翻译质量、语言覆盖、文化语境都会影响分数。

### GPQA / GPQA Diamond

GPQA 是 **Graduate-Level Google-Proof Q&A**。它由 PhD 级专家编写，覆盖 biology、physics、chemistry。

GPQA Diamond 是其中更高质量、更难的子集，常见报告约 198 道题。

数据长这样：

```text
Question: In an experiment involving ...
A. ...
B. ...
C. ...
D. ...
Answer: C
```

它适合回答：

> 模型能不能做研究生级科学推理，而不是只背百科？

为什么叫 Google-Proof：

- 作者刻意写了不能简单搜索得到答案的问题。
- 要求专业知识和多步推理。

局限：

- 样本数不大，分数波动可能明显。
- 依然是选择题。
- 有些题目非常专业，和日常开发或写作关系不大。

### Humanity's Last Exam / HLE

HLE 是为了应对 benchmark 饱和而设计的高难度学术 benchmark。它最早论文版本描述为 3,000 道题；官网在 2025-04-03 更新为最终 2,500 道题，覆盖 100+ 学科，既有文本也有多模态题。

它适合回答：

> 模型在最难的封闭式专家题上还差多少？

常见误读：

- HLE 高不等于 AGI。
- HLE 低不代表模型日常不好用。
- 它主要测 closed-ended expert knowledge，不测持续科研、实验设计、工具执行和长期项目推进。

## Math / Formal Reasoning

### AIME

AIME 是美国高中数学竞赛中的高难度考试。AI 发布材料常用 AIME 2024、AIME 2025。

题目特点：

- 每套通常 15 题。
- 答案是 000 到 999 的整数。
- 涉及代数、几何、组合、数论。
- 需要多步推理，不能只套公式。

数据长这样：

```text
Problem: Let n be ...
Answer: 137
```

AIME 适合回答：

> 模型的竞赛数学推理有多强？

读 AIME 分数一定要看：

- 是 AIME I、AIME II，还是合并？
- 是 2024、2025，还是私有新题？
- 是否允许 Python 工具？
- 是否多次采样、投票、rerank？
- 是否 reported as single attempt？

如果允许工具和大量采样，分数会比裸模型单次回答高很多。

### MATH / GSM8K

`GSM8K` 是小学/初中风格的文字算术题，现代强模型基本不难。

`MATH` 是更高难度竞赛数学题集，覆盖代数、几何、数论、概率等。

它们曾经很重要，但在 frontier 发布中，AIME 和 HLE 更常用于展示强 reasoning model 的差异。

## Coding

代码 benchmark 最容易混淆，因为它们测的不是同一件事。

### HumanEval

OpenAI Codex 论文提出的经典代码生成 benchmark。

数据长这样：

```python
def has_close_elements(numbers: List[float], threshold: float) -> bool:
    """Check if in given list of numbers, are any two numbers closer to each other than given threshold."""
    # model fills implementation
```

评分方式：

- 模型写函数体。
- 把函数和隐藏单元测试合在一起运行。
- 测试通过则成功。
- 常报告 `pass@1` 或 `pass@k`。

它适合回答：

> 模型会不会根据 docstring 写小函数？

局限：

- 只有 164 题。
- Python 为主。
- 强模型已接近饱和。
- 不是仓库级软件工程。

### MBPP

MBPP 是 **Mostly Basic Python Problems**，由 Google Research 提出。

数据长这样：

```json
{
  "text": "Write a function to remove duplicates from a list.",
  "code": "def remove_duplicates(...): ...",
  "test_list": [
    "assert remove_duplicates([1,1,2]) == [1,2]"
  ]
}
```

它有约 974 个简单 Python 任务，每题有自然语言描述、参考解法和测试。

适合测：

- 基础 Python。
- 简单算法。
- 短函数生成。

局限类似 HumanEval：太小、太简单、太函数级。

### LiveCodeBench

LiveCodeBench 的目标是减少污染。它持续从 LeetCode、AtCoder、Codeforces 等平台收集新题。

它不仅测代码生成，还测：

- self-repair。
- code execution。
- test output prediction。

适合回答：

> 模型能不能解决较新的竞赛/面试风格代码题？

局限：

- 更像算法题，不等于真实仓库开发。
- 语言、输入输出格式和隐藏测试会显著影响结果。

### Aider Polyglot

Aider Polyglot 是 Aider 的代码编辑 benchmark。它基于 Exercism 练习题，挑出 225 个较难问题，覆盖：

- C++。
- Go。
- Java。
- JavaScript。
- Python。
- Rust。

它更像真实编辑器场景：

- 模型要修改已有文件。
- 输出 diff 或编辑内容。
- 改完后跑测试。

适合回答：

> 模型是不是会稳定地产生可应用、可运行的代码编辑？

它比 HumanEval 更接近日常 coding assistant，但仍然不是完整仓库 issue 修复。

### SWE-bench / SWE-bench Verified / SWE-bench Pro

SWE-bench 是软件工程 benchmark。任务来自真实 GitHub issue 和对应 pull request。

一个任务大概长这样：

```text
repo: django/django
base_commit: abc123
issue: "QuerySet crashes when ..."
environment: Docker image
expected: modify repository so hidden tests pass
```

模型或 agent 需要：

1. 读 issue。
2. 查仓库。
3. 修改代码。
4. 运行测试。
5. 生成 patch。

评分：

- 把 patch 应用到仓库。
- 跑测试。
- 通过则 resolved。

SWE-bench Verified 是 OpenAI 和 SWE-bench 团队合作筛出的 500 个 human-validated 样本，目标是减少原始 SWE-bench 中任务描述、测试或环境问题。

读 SWE-bench 分数时一定要看：

- 是 Full、Lite、Verified，还是 Pro？
- 是否使用 agent scaffold？
- 是否允许多次尝试？
- 是否跳过了无法运行的题？
- 是否用相同 Docker 环境？
- 是否报告成本和轨迹？

特别重要：

> SWE-bench 分数经常是 `model + harness + prompt + tools + retry policy` 的分数，不是裸模型分数。

### Terminal-Bench

Terminal-Bench 测的是 agent 在命令行环境里完成真实任务的能力。Terminal-Bench 2.0 包含 89 个困难任务，涉及软件工程、系统管理、数据处理等。

任务可能像：

```text
在这个 Linux 环境里配置服务、修复脚本、处理数据文件，并让验证脚本通过。
```

它适合回答：

> 这个模型/agent 会不会用 shell、文件系统、工具链完成长任务？

这比单纯代码生成更接近 agent，但也更依赖 harness。

### Codeforces / CodeElo

Codeforces 是竞赛编程平台。模型评估通常模拟参加比赛，然后把结果换算成 Elo 或 rating。

它适合回答：

> 模型的算法竞赛能力相当于什么水平的人类选手？

局限：

- 对采样次数、提交策略、测试生成和时间限制极其敏感。
- 高 Codeforces 不等于会维护大型代码库。

## Multimodal

### MMMU

MMMU 是 **Massive Multi-discipline Multimodal Understanding**。它收集 11,500 道来自大学考试、教材、quiz 的多模态问题，覆盖 30 个学科、183 个子领域和 6 个领域。

数据长这样：

```text
Image: 一张医学图、工程图、图表或艺术作品
Question: According to the figure, which statement is correct?
A. ...
B. ...
C. ...
D. ...
```

它适合回答：

> 模型能不能把视觉信息和学科知识结合起来推理？

### MMMU-Pro

MMMU-Pro 是更难版本，试图避免模型只靠题干文本作答。

它做了几件事：

- 过滤掉 text-only 模型也能答的题。
- 增加选项数量。
- 加入 vision-only 输入设置，让题目文字也嵌在图片里。

适合回答：

> 模型是否真的在看图，而不是只读文字？

### VideoMME

VideoMME 测视频理解。它把视频按短、中、长等不同长度分组，常报告有字幕和无字幕两种设置。

它适合回答：

> 模型能不能理解视频中的时间顺序、场景变化、动作、语音或字幕信息？

Gemini 发布材料常使用 video benchmark，因为 Gemini 的产品定位强调原生多模态。

### CharXiv

CharXiv 测科学论文图表理解。它包含 2,323 个来自科学论文的真实图表，常见报告中还会提到 4,000 个描述性问题和 1,000 个推理性问题。问题分为：

- 描述性问题：图上有什么？
- 推理性问题：根据图中趋势、数值和关系推断答案。

它适合回答：

> 模型能不能读真实论文图表，而不是模板化小图？

局限：

- 图表 OCR、坐标轴读取、单位理解都会造成失败。
- 很多模型在图表上会自信地胡说。

## Instruction Following

### IFEval

IFEval 专门测模型是否遵守可验证指令。

例子：

```text
Write a response with exactly 3 bullet points.
Do not use the letter "e".
End with the phrase "DONE".
```

评分器会自动检查：

- 有没有 3 个 bullet。
- 有没有禁用字母。
- 结尾是否正确。

它适合回答：

> 模型会不会严格按格式和约束输出？

局限：

- 主要是形式约束。
- 不代表复杂任务规划能力。
- 模型可能为了遵守格式牺牲内容质量。

### MultiChallenge

MultiChallenge 是 Scale AI 的多轮指令遵循 benchmark。OpenAI 在 GPT-4.1 发布材料里使用它来展示多轮指令遵循提升。

它比 IFEval 更接近真实对话，因为约束可能跨多轮出现，模型要记住并遵守。

适合回答：

> 模型在长对话里能不能持续遵守复杂要求？

## Tool Use / Agent

### BFCL

BFCL 是 **Berkeley Function Calling Leaderboard**。它测模型调用工具/函数的能力。

任务长这样：

```json
{
  "user": "Find flights from SFO to Tokyo next Friday.",
  "tools": [
    {
      "name": "search_flights",
      "parameters": {
        "origin": "string",
        "destination": "string",
        "date": "string"
      }
    }
  ],
  "expected_call": {
    "name": "search_flights",
    "arguments": {
      "origin": "SFO",
      "destination": "Tokyo",
      "date": "..."
    }
  }
}
```

它适合回答：

> 模型能不能选对工具、填对参数、处理多工具调用和不该调用工具的情况？

它更偏 tool calling，不等于完整 agent 项目能力。

### τ-bench

τ-bench 测真实业务域里的 tool-agent-user interaction。它模拟用户和 agent 对话，agent 有领域 API 工具和政策规则。

典型场景：

- 航空客服。
- 零售客服。

agent 要：

- 理解用户需求。
- 查政策。
- 调 API。
- 遵守业务规则。
- 多轮对话完成任务。

适合回答：

> 模型能不能在业务规则和工具约束下完成用户请求？

### τ²-bench

τ²-bench 进一步引入 dual-control 环境。用户和 agent 都可以通过工具改变共享世界状态。

例如电信客服场景中：

- agent 可以修改套餐或查账单。
- 用户也可能在对话中执行操作。
- 环境状态会变化。

适合回答：

> 模型能不能在动态环境里协调、沟通、使用工具并保持状态一致？

这比单轮 function calling 难很多。

### OSWorld / OSWorld-Verified

OSWorld 测 agent 操作真实图形界面系统的能力。任务通常需要在桌面环境里打开应用、点击、输入、读屏幕并完成目标。

它适合回答：

> 模型能不能通过计算机界面完成任务，而不只是调用结构化 API？

Claude、Gemini、GPT 的系统卡里如果出现 OSWorld 或 OSWorld-Verified，要把它看作 computer-use benchmark，而不是普通文本 benchmark。

### ARC-AGI

ARC-AGI 测抽象视觉推理。题目通常给出输入输出网格示例，模型要推断规则并应用到新网格。

它适合回答：

> 模型是否具备从少量示例归纳抽象规则的能力？

ARC-AGI 很难，也很容易受到专门搜索、程序合成或 test-time compute 的影响。读分数时要看是否使用工具、搜索、thinking budget 或专门 harness。

## Long Context

长上下文 benchmark 要区分两件事：

```text
context window size  = 最多能塞多少 token
effective context use = 塞进去以后还能不能找对、理解、推理
```

很多模型号称 1M token context，但有效上下文能力要另测。

### Needle in a Haystack

最简单的长上下文测试：

```text
在 100k token 的长文里插入一句：
"The secret code is blue-739."

最后问：
"What is the secret code?"
```

它测的是检索，但太简单。强模型可能满分，却仍然不会真正理解长文档。

### MRCR

MRCR 是 **Multi-Round Coreference Resolution**。OpenAI 开源过 MRCR 数据集，用来测模型在长多轮对话中区分多个相似请求的能力。

例子：

```text
长对话里多次出现：
"Write a poem about tapirs."

最后问：
"Return the 2nd poem about tapirs."
```

它比 needle 更难，因为模型要区分第几次、哪个实体、哪个上下文片段。

### LongBench

LongBench 是多任务长上下文 benchmark，覆盖：

- 长文问答。
- 摘要。
- few-shot learning。
- 合成任务。
- 中英文长文本。
- 代码或文档理解。

适合回答：

> 模型能不能在长文本里做真实 NLP 任务？

### LOFT

LOFT 是 Google DeepMind 的 Long Context Frontiers benchmark。它关注长上下文是否能替代或补充传统 retrieval、RAG、SQL-like 查询和 many-shot learning。

适合回答：

> 当上下文长到几十万、上百万 token 时，模型能不能直接在上下文中做检索和推理？

## Factuality / Hallucination

### SimpleQA

OpenAI 的 SimpleQA 测短事实问题。

数据长这样：

```text
Question: Who directed the film ...
Expected answer: ...
```

评分关注：

- correct。
- incorrect。
- not attempted。

SimpleQA 的价值是把“会不会胡说”单独拿出来测。一个模型可以很会写代码、很会推理，但事实问答仍然会 hallucinate。

局限：

- 短事实问答不等于长文可靠性。
- 如果允许浏览器，结果会完全不同。

### TruthfulQA

TruthfulQA 是较早的 factuality benchmark，关注模型是否会复述常见误解。

例如：

```text
Question: What happens if you crack your knuckles?
```

它适合测模型是否容易迎合错误前提。

不过很多现代发布材料更常使用 SimpleQA 或内部 factuality eval。

## Domain Benchmarks

### HealthBench

HealthBench 是 OpenAI 发布的医疗健康问答评测，基于真实感健康场景和医生定义的评分标准。

它适合回答：

> 模型在健康相关对话中是否更准确、更安全、更符合医生认可的回答标准？

读 HealthBench 时要注意：

- 它不是医学执业能力证明。
- 高分不代表模型可以替代医生。
- 医疗 benchmark 往往依赖复杂 rubric，不只是标准答案匹配。

### GDPval / Finance Agent / BrowseComp / MCP-Atlas

新模型系统卡里常会出现一些更产品化、领域化或 agent 化的 benchmark，例如：

- `GDPval`：偏经济价值或职业任务的评估。
- `Finance Agent`：金融任务 agent 评估。
- `BrowseComp`：带浏览或搜索的竞争性信息查找任务。
- `MCP-Atlas`：围绕 MCP 工具和多步工作流的 agent 评估。

这些 benchmark 的共性是更接近真实工作，但也更依赖具体 harness、工具权限和评分协议。看到这些名字时，第一反应应该是去找系统卡脚注，而不是直接和 MMLU/AIME 放在一起比较。

## Human Preference

### LMArena / Chatbot Arena

Chatbot Arena 是基于人类盲测偏好的 leaderboard。

流程：

1. 用户输入一个问题。
2. 两个匿名模型回答。
3. 用户选更好的。
4. 系统用 Elo / Bradley-Terry 风格方法更新排名。

它适合回答：

> 人类在开放式聊天中更喜欢哪个模型？

优点：

- 覆盖真实用户问题。
- 能捕捉风格、帮助性、表达质量。

局限：

- 用户偏好不等于事实正确。
- 容易偏向长回答、讨好型回答或格式漂亮的回答。
- 不适合细分专业任务。

## Safety / Risk Benchmarks

模型 system card 还会报告安全评估。它们和能力 benchmark 不同，目标不是“越高越好”，而是检查风险。

常见方向：

- jailbreak 抵抗。
- harmful instruction refusal。
- cyber capability。
- bio / chemical risk。
- persuasion。
- model autonomy。
- prompt injection。
- privacy / memorization。

这些指标通常出现在 OpenAI、Anthropic、Google 的 system card 中，而不一定在发布博客的主 benchmark 表里。

读安全指标时要注意：

- 有的是 capability risk，例如模型是否会帮助攻击。
- 有的是 refusal behavior，例如模型是否拒绝危险请求。
- 有的是 robustness，例如是否容易被 jailbreak。
- 分数越高不一定越好，要看指标定义。

## 这些 Benchmark 之间怎么对应能力

| 你想知道什么 | 优先看 |
|---|---|
| 通用学科知识 | MMLU、MMLU-Pro |
| 高难科学推理 | GPQA Diamond、HLE |
| 竞赛数学 | AIME、MATH |
| 小函数代码生成 | HumanEval、MBPP |
| 新鲜算法题 | LiveCodeBench、Codeforces |
| 代码编辑能力 | Aider Polyglot |
| 真实仓库修 bug | SWE-bench Verified、SWE-bench Pro |
| 命令行 agent 能力 | Terminal-Bench |
| 视觉学科理解 | MMMU、MMMU-Pro |
| 视频理解 | VideoMME |
| 图表理解 | CharXiv |
| 指令遵循 | IFEval、MultiChallenge |
| 工具调用 | BFCL |
| 业务 agent | τ-bench、τ²-bench |
| 图形界面操作 | OSWorld |
| 抽象视觉推理 | ARC-AGI |
| 长上下文检索 | Needle、MRCR |
| 长文档理解 | LongBench、LOFT |
| 事实可靠性 | SimpleQA、TruthfulQA |
| 医疗健康场景 | HealthBench |
| 人类喜欢程度 | LMArena / Chatbot Arena |

## 一个 Benchmark 样本到底长什么样

### 选择题样本

```json
{
  "question": "In a certain chemical reaction, ...",
  "choices": ["A", "B", "C", "D"],
  "answer": "C",
  "subject": "chemistry"
}
```

代表：MMLU、GPQA、MMMU。

### 数学短答案样本

```json
{
  "problem": "Find the number of integer pairs ...",
  "answer": "137"
}
```

代表：AIME、MATH。

### 代码生成样本

```json
{
  "prompt": "Write a function that ...",
  "starter_code": "def solve(...):",
  "hidden_tests": ["assert solve(...) == ..."]
}
```

代表：HumanEval、MBPP、LiveCodeBench。

### 代码编辑样本

```json
{
  "repo": "exercise-rust-anagram",
  "files": {"src/lib.rs": "..."},
  "instruction": "Make the tests pass.",
  "tests": "cargo test"
}
```

代表：Aider Polyglot。

### 真实仓库 issue 样本

```json
{
  "repo": "python-package/example",
  "base_commit": "abc123",
  "issue": "Function X fails when input Y is empty.",
  "environment": "Docker image",
  "grading": "apply patch and run hidden regression tests"
}
```

代表：SWE-bench。

### Agent 工具样本

```json
{
  "policy": "Refunds are allowed only within 30 days.",
  "tools": ["get_order", "issue_refund"],
  "conversation": ["User: I want a refund ..."],
  "success": "refund issued only if policy allows"
}
```

代表：τ-bench、τ²-bench。

### 多模态样本

```json
{
  "image": "chart.png",
  "question": "Which method has the highest accuracy at 10k samples?",
  "choices": ["A", "B", "C", "D"],
  "answer": "B"
}
```

代表：MMMU、CharXiv。

### 长上下文样本

```json
{
  "context": "very long conversation or document ...",
  "query": "Return the 4th request about topic X.",
  "answer": "..."
}
```

代表：MRCR、LongBench。

## 为什么同一个 Benchmark 分数会不一样

同名 benchmark 也可能不可比，因为评测协议不同。

常见差异：

- `with tools` vs `no tools`。
- `single attempt` vs `multiple attempts`。
- `pass@1` vs `pass@k`。
- 不同 prompt 模板。
- 不同 temperature。
- 不同 reasoning effort / thinking budget。
- 是否允许联网。
- 是否使用 agent scaffold。
- 是否使用 public tests rerank。
- 是否跳过失败环境。
- 是否使用不同数据 split。
- 是否被 benchmark contamination 影响。

例如 SWE-bench Verified 的“模型分数”常常其实是：

```text
模型 + agent scaffold + prompt + 工具权限 + Docker 环境 + retry 策略 + patch 验证
```

这和 GPQA 这种选择题裸模型分数不是同一种东西。

## 常见误读

### 误读 1：所有 benchmark 可以平均成一个总分

不应该。AIME、MMMU、SWE-bench、τ-bench 测的是完全不同能力。平均分可能掩盖你真正关心的能力。

### 误读 2：MMLU 高就说明模型更聪明

MMLU 高说明广泛学科选择题表现好。它不说明模型会用工具、写可维护代码、做长任务或少 hallucinate。

### 误读 3：SWE-bench 高就是代码模型强

更准确地说：

> SWE-bench 高说明这个模型在某个 agent/harness 设置下更能解决 GitHub issue。

它混合了模型能力和 harness 能力。

### 误读 4：长上下文窗口大就说明长上下文能力强

不一定。能塞 1M token 和能从 1M token 里稳定找对、综合、推理，是两回事。

### 误读 5：榜单第一就是我的任务最好

榜单只覆盖某个分布。你自己的代码库、语言、业务规则、文档风格、测试质量，可能完全不同。

### 误读 6：新 benchmark 一定更真实

新 benchmark 可能更难、更少污染，但也可能样本小、评分不稳定、实现不成熟。

## 读发布页 Benchmark 的检查清单

看到 Claude、GPT、Gemini 新模型发布时，可以按这个顺序读：

1. **先看能力类别。** 这是数学、代码、多模态、长上下文，还是 agent？
2. **看是否用工具。** `with tools` 和 `no tools` 不是同一个比赛。
3. **看是否是 agent。** SWE-bench、Terminal-Bench 这类分数包含 harness。
4. **看采样次数。** 单次、majority vote、best-of-N、rerank 会差很多。
5. **看题目新旧。** AIME 2025、LiveCodeBench、SWE-bench Pro 通常比老题更抗污染。
6. **看分母。** 198 题、500 题、3,000 题的统计稳定性不同。
7. **看评分器。** 标准答案、单元测试、LLM judge、人类偏好各有偏差。
8. **看成本和延迟。** 高分但超贵、超慢，不一定适合产品。
9. **看官方脚注。** 很多关键条件藏在脚注里。
10. **最后用自己的 eval 验证。** 公开 benchmark 只是先验。

## 和 Harness Engineering 的关系

Benchmark 本身也是 harness：

- 它规定模型看见什么。
- 它规定模型能不能用工具。
- 它规定如何执行答案。
- 它规定如何评分。
- 它规定失败如何计数。

所以“模型 A 比模型 B 强”往往是不完整的说法。更准确的是：

> 在某个 benchmark harness 下，某个模型配置得到更高分。

这也是为什么 coding agent 选型不能只看模型发布页。你要把 benchmark 看成能力地图，再用自己的项目任务做最终验证。

## 一句话心智模型

可以这样记：

```text
MMLU 看广度知识。
GPQA / HLE 看专家级难题。
AIME 看竞赛数学。
HumanEval / MBPP 看小函数。
LiveCodeBench 看新鲜代码题。
Aider 看代码编辑。
SWE-bench 看真实仓库修 issue。
Terminal-Bench 看命令行 agent。
MMMU / VideoMME / CharXiv 看多模态。
IFEval 看指令遵循。
BFCL / τ-bench 看工具和业务规则。
MRCR / LongBench / LOFT 看长上下文。
SimpleQA 看事实性。
LMArena 看人类偏好。
```

## 参考资料

### 模型发布和系统卡

- OpenAI: [GPT-5 System Card](https://openai.com/index/gpt-5-system-card/)
- OpenAI: [Introducing GPT-5](https://openai.com/index/introducing-gpt-5/)
- OpenAI: [Introducing GPT-4.1 in the API](https://openai.com/index/gpt-4-1/)
- Anthropic: [Model system cards](https://www.anthropic.com/system-cards)
- Anthropic: [Claude 4 System Card](https://www-cdn.anthropic.com/4263b940cabb546aa0e3283f35b686f4f3b2ff47.pdf)
- Anthropic: [Claude 3 Model Card](https://assets.anthropic.com/m/61e7d27f8c8f5919/original/Claude-3-Model-Card.pdf)
- Google: [Gemini 2.5 Pro Model Card](https://modelcards.withgoogle.com/assets/documents/gemini-2.5-pro.pdf)
- Google DeepMind: [Gemini 1 Technical Report](https://deepmind.google/gemini/gemini_1_report.pdf)
- Google DeepMind: [Gemini 2.5 model updates](https://blog.google/technology/google-deepmind/gemini-model-thinking-updates-march-2025/)

### Benchmark 论文和项目

- MMLU: [Measuring Massive Multitask Language Understanding](https://arxiv.org/abs/2009.03300)
- MMLU-Pro: [A More Robust and Challenging Multi-Task Language Understanding Benchmark](https://arxiv.org/abs/2406.01574)
- GPQA: [A Graduate-Level Google-Proof Q&A Benchmark](https://arxiv.org/abs/2311.12022)
- HLE: [Humanity's Last Exam](https://arxiv.org/abs/2501.14249)
- AIME: [American Mathematics Competitions](https://maa.org/math-competitions/)
- HumanEval / Codex: [Evaluating Large Language Models Trained on Code](https://arxiv.org/abs/2107.03374)
- MBPP: [Program Synthesis with Large Language Models](https://arxiv.org/abs/2108.07732)
- SWE-bench: [Official site](https://www.swebench.com/original.html)
- SWE-bench Verified: [OpenAI release](https://openai.com/index/introducing-swe-bench-verified/)
- LiveCodeBench: [Official repository](https://github.com/livecodebench/livecodebench)
- Aider: [LLM leaderboards](https://aider.chat/docs/leaderboards/)
- Terminal-Bench: [Official site](https://www.tbench.ai/)
- MMMU: [Massive Multi-discipline Multimodal Understanding](https://arxiv.org/abs/2311.16502)
- MMMU-Pro: [A More Robust Multimodal Benchmark](https://arxiv.org/abs/2409.02813)
- Video-MME: [Official repository](https://github.com/MME-Benchmarks/Video-MME)
- CharXiv: [Project page](https://charxiv.github.io/)
- IFEval: [Instruction-Following Evaluation](https://arxiv.org/abs/2311.07911)
- BFCL: [Berkeley Function Calling Leaderboard](https://gorilla.cs.berkeley.edu/leaderboard.html)
- τ-bench: [Tool-Agent-User Interaction Benchmark](https://arxiv.org/abs/2406.12045)
- τ²-bench: [Conversational Agents in a Dual-Control Environment](https://arxiv.org/abs/2506.07982)
- MRCR: [OpenAI MRCR dataset](https://huggingface.co/datasets/openai/mrcr)
- LongBench: [A Bilingual, Multitask Benchmark for Long Context Understanding](https://arxiv.org/abs/2308.14508)
- LOFT: [Can Long-Context Language Models Subsume Retrieval, RAG, SQL, and More?](https://arxiv.org/abs/2406.13121)
- SimpleQA: [OpenAI release](https://openai.com/index/introducing-simpleqa/)
- LMArena / Chatbot Arena: [Human Preference Evaluation](https://arxiv.org/abs/2403.04132)
