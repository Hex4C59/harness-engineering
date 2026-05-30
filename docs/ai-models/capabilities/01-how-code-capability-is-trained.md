# 代码能力是怎么训练出来的

## 核心结论

代码能力不是靠一种数据或一个训练阶段学出来的。现代 code LLM 通常是多阶段训练的结果：

```text
通用预训练
  -> 代码继续预训练
  -> Fill-in-the-Middle / 长上下文 / 仓库级训练
  -> 代码指令微调
  -> 偏好对齐 / 执行反馈 / 可验证奖励
  -> 面向 coding agent 的轨迹训练和环境交互训练
```

如果只看“会不会写一个函数”，主要来自代码预训练和指令微调。如果看“能不能在真实仓库里修 bug、跑测试、改多文件、处理失败”，则越来越依赖 **execution feedback**、**repository context** 和 **agentic training**。

闭源商业模型通常不会公开完整训练数据、混合比例和后训练流程。下面的总结主要基于公开论文、开源模型技术报告和数据集说明，因此更适合理解行业共通配方，而不是反推出某个闭源模型的完整训练集。

## 代码能力包含什么

“代码能力”不是单一能力，至少包括这些层次：

| 能力 | 例子 | 常见训练信号 |
|---|---|---|
| 语法和局部补全 | 补全一行代码、闭合括号、补 import | 大规模源代码 next-token training |
| API 和库使用 | 使用 `pandas`、`React`、`pytest` | 源码、文档、notebook、问答、示例代码 |
| 函数级生成 | 根据 docstring 写函数 | HumanEval/MBPP 类 instruction data |
| 代码修改 | 修 bug、重构、补测试 | commit diff、PR、issue、code review |
| 代码理解 | 解释代码、找调用链、读仓库 | repo-level packing、长上下文训练 |
| 可执行正确性 | 通过单元测试、编译、静态检查 | test cases、compiler feedback、execution reward |
| agentic coding | 调工具、跑命令、改多文件、恢复失败 | 轨迹数据、sandbox 环境、RL with verifiable rewards |

所以训练数据也会从“单个源码文件”逐步扩展到“仓库、提交、问题、测试、工具轨迹和执行环境”。

## 训练数据长什么样

### 1. 原始代码语料

最基础的是从公开代码仓库收集源码文件。典型来源包括：

- GitHub 仓库。
- Software Heritage 归档。
- 包管理生态里的源码包。
- 代码文档、README、API docs。
- GitHub issues、pull requests、commit messages。
- notebooks，例如 Kaggle notebooks。
- Stack Overflow 或 Stack Exchange 这类代码相关问答。

以 The Stack v2 为例，它从 Software Heritage 图数据中收集 GitHub 代码，记录了文件路径、仓库、语言、许可证、commit/revision 元数据等字段，并做 exact / near dedup、许可证检测、文件大小过滤、二进制文件过滤、语言检测和 opt-out 机制。

典型原始样本类似：

```json
{
  "repo_name": "owner/project",
  "path": "src/auth/token.ts",
  "language": "TypeScript",
  "content": "export function refreshToken(...) { ... }",
  "detected_licenses": ["MIT"],
  "revision_id": "...",
  "is_generated": false,
  "is_vendor": false
}
```

这种数据主要训练模型学习：

- 编程语言语法。
- 常见框架和库的使用方式。
- 项目组织方式。
- 注释、docstring 和代码之间的对应关系。
- 代码风格和惯用法。

### 2. 代码相关自然语言

纯源码不够。模型还要学会把自然语言需求映射到代码，所以会混入代码相关文本：

- README。
- API 文档。
- 教程。
- issue 描述。
- PR 描述。
- commit message。
- Stack Overflow 问答。
- bug report。
- 测试失败日志。

Code Llama 的公开报告就是一个典型例子：它的代码继续预训练数据以公开代码为主，同时混入代码相关自然语言和少量通用自然语言，以减少模型变成“只会续写代码、不理解需求”的窄模型。

### 3. Text-code grounding 数据

这一类数据把自然语言和代码绑定在一起：

```json
{
  "instruction": "Write a Python function that returns the longest palindromic substring.",
  "solution": "def longest_palindrome(s): ...",
  "tests": [
    "assert longest_palindrome('babad') in ['bab', 'aba']",
    "assert longest_palindrome('cbbd') == 'bb'"
  ]
}
```

常见形式包括：

- 问题描述 + 解法代码。
- 函数签名 + docstring + 函数体。
- bug 描述 + patch。
- commit message + diff。
- issue + PR。
- prompt + expected code。
- 代码 + 解释。
- 代码 + 单元测试。

Qwen2.5-Coder 报告中把数据概括为 source code、text-code grounding、synthetic data、math data 和 text data 这几类，说明现代代码模型通常不是只吃源码，而是吃“代码和意图之间的连接”。

### 4. Git commit / PR / diff 数据

代码编辑能力通常需要 diff 数据。OctoPack 的 CommitPack 就是典型方法：把 Git commits 看成天然的 instruction tuning 数据，因为 commit message 描述意图，code diff 描述执行。

样本可以表示成：

```json
{
  "instruction": "fix race condition in token refresh",
  "input_files": {
    "src/auth/session.ts": "...old code..."
  },
  "patch": "diff --git a/src/auth/session.ts b/src/auth/session.ts\n..."
}
```

这类数据训练的是：

- 根据人类意图修改已有代码。
- 生成 patch，而不是从零写完整文件。
- 理解局部上下文。
- 保持改动集中。
- 处理多文件变更。

缺点是噪声很大：commit message 可能很短，commit 可能混了多个无关修改，bot commit 很多，diff 未必代表最优修法。

### 5. 合成指令数据

真实高质量 instruction data 很贵，所以很多 code model 会生成合成数据：

- 从开源代码片段生成问题。
- 从函数生成 docstring。
- 从题目生成多种解法。
- 从代码生成 bug，再生成修复。
- 从 API 文档生成使用任务。
- 让强模型生成 instruction / solution / tests。
- 用执行器过滤不能运行或过不了测试的样本。

Magicoder 的 OSS-Instruct 是一个代表：它用开源代码片段引导模型生成更贴近真实代码的合成指令。OpenCodeInstruct 则把样本扩展到 programming question、solution、test cases、execution feedback 和质量评估。

### 6. 偏好和反馈数据

代码不是“能跑”就一定好。模型还要学会人类偏好：

- 改动是否最小。
- 是否符合项目风格。
- 是否安全。
- 是否可维护。
- 是否解释清楚。
- 是否避免过度设计。

偏好数据常见格式：

```json
{
  "instruction": "Refactor this function without changing behavior.",
  "chosen": "small, readable patch",
  "rejected": "large rewrite with unrelated abstractions",
  "criteria": ["correctness", "minimal_diff", "style_fit"]
}
```

CodeUltraFeedback 这类数据集会让多个模型回答同一 coding instruction，再用 judge 评分或排序，随后用 SFT、DPO、RLAIF 等方式对齐模型的 coding preferences。

### 7. 执行反馈和可验证奖励数据

代码训练有一个很强的优势：很多任务可以自动验证。

可验证信号包括：

- 是否编译通过。
- 是否通过单元测试。
- 是否通过隐藏测试。
- 是否满足 lint / typecheck。
- 是否通过 benchmark。
- 是否生成了期望文件。
- 是否修复了 failing test。

这类数据可以表示成：

```json
{
  "prompt": "Fix the failing parser tests.",
  "repo_snapshot": "parser-lib@abc123",
  "candidate_patch": "...",
  "commands": ["npm test"],
  "result": {
    "passed": true,
    "stdout": "...",
    "reward": 1.0
  }
}
```

这就是 code domain 特别适合 **reinforcement learning with verifiable rewards** 的原因。相比“这篇文章写得好不好”，代码任务常常能用测试和编译器给出硬反馈。

### 8. Agent 轨迹数据

coding agent 不只输出代码，还会经历一串操作：

```text
读 README
-> 搜索关键函数
-> 打开文件
-> 修改代码
-> 运行测试
-> 观察失败
-> 再修改
-> 重新测试
-> 总结结果
```

Agentic coding 训练会把这种轨迹也纳入数据：

```json
{
  "task": "Fix issue #123",
  "trajectory": [
    {"thought": "...", "tool": "rg", "args": "refreshToken"},
    {"observation": "..."},
    {"tool": "edit", "patch": "..."},
    {"tool": "shell", "cmd": "npm test"},
    {"observation": "1 failing test ..."},
    {"tool": "edit", "patch": "..."}
  ],
  "final_patch": "...",
  "verified": true
}
```

Qwen3-Coder-Next 报告中的关键词就是 agentic training：通过大规模合成可验证 coding tasks 和 executable environments，让模型从环境反馈中学习。

## 训练方式怎么分阶段

### Stage 1：通用语言预训练

很多代码模型不是从零开始，而是从通用语言模型继续训练。

目标函数通常是自回归 next-token prediction：

```text
给定 token_1 ... token_n，预测 token_{n+1}
```

这个阶段让模型学到：

- 自然语言理解。
- 常识和推理基础。
- 文档阅读能力。
- 问题描述理解能力。

对于 coding 来说，通用语言能力很重要，因为真实任务往往是自然语言需求、错误日志、issue 描述和代码混在一起。

### Stage 2：代码继续预训练

这是把通用模型变成 code model 的核心阶段。

训练目标仍然主要是 next-token prediction，但数据换成代码密集语料：

```text
输入：仓库文件、源码片段、README、issue、notebook
目标：预测下一个 token
```

这个阶段学到：

- 语法。
- 代码结构。
- API 调用习惯。
- 常见算法模板。
- 错误处理模式。
- 语言之间的迁移。

公开例子：

| 模型 | 训练方式和数据线索 |
|---|---|
| Codex | 在 GPT 模型基础上，用公开 GitHub 代码微调，重点研究 Python 代码生成。 |
| Code Llama | 从 Llama 2 初始化，继续训练 500B 代码相关 token，70B 版本训练 1T token，并有 Python 专门化阶段。 |
| StarCoder2 | 基于 The Stack v2、GitHub PR、Kaggle notebooks 和代码文档，训练 3.3T 到 4.3T token。 |
| DeepSeek-Coder | 从零训练，2T token，约 87% code 和 13% 中英文自然语言，支持 80+ 语言。 |
| Qwen2.5-Coder | 基于 Qwen2.5 继续预训练，公开报告称使用超过 5.5T token，并混合代码、text-code grounding、合成数据、数学和文本。 |
| Granite Code | 面向企业代码任务，公开资料称使用 116 种语言的大规模代码语料，并混合高质量代码和自然语言。 |

### Stage 3：Fill-in-the-Middle 训练

IDE 里的真实补全经常不是“从左到右续写”，而是在文件中间补一段代码。例如：

```text
prefix: def load_user(id):
            ...

suffix:     return user

middle:     if user is None:
                raise NotFound()
```

所以很多 code model 会加入 **Fill-in-the-Middle**，简称 FIM。常见格式包括：

```text
<prefix> 前文 <suffix> 后文 <middle> 要补的代码
```

或：

```text
<suffix> 后文 <prefix> 前文 <middle> 要补的代码
```

FIM 训练的价值：

- 更适合 IDE inline completion。
- 更适合在已有函数中补局部逻辑。
- 更适合修改中间代码而不是只追加。
- 能利用前后文约束生成更贴合的片段。

Code Llama、DeepSeek-Coder、StarCoder 系列都明确支持或讨论了 infilling / FIM。

### Stage 4：文件级和仓库级训练

早期 code model 常见上下文很短，只能看一个函数或一个文件片段。真实 coding agent 需要看：

- 多文件调用链。
- 配置文件。
- 测试文件。
- README。
- package manifest。
- 目录结构。
- 历史约定。

所以训练数据会从“单个文件”变成“仓库内文件打包”。例如：

```text
<repo_name> my-api
<file_sep> package.json
...
<file_sep> src/routes/auth.ts
...
<file_sep> tests/auth.test.ts
...
```

这会训练模型理解：

- 文件之间的依赖关系。
- import/export。
- 测试和实现的对应关系。
- 项目级风格。
- 长上下文检索。

Qwen2.5-Coder 报告提到 file-level 和 repository-level pretraining；Granite 也有把模型扩展到 128K context 的 repository-level long-context 训练。

### Stage 5：代码指令微调

预训练模型擅长续写，但不一定擅长“听指令”。指令微调用来把模型变成 assistant。

训练样本通常是：

```json
{
  "messages": [
    {"role": "user", "content": "Write tests for this function."},
    {"role": "assistant", "content": "Here are pytest tests..."}
  ]
}
```

代码指令任务包括：

- 写函数。
- 修 bug。
- 解释代码。
- 生成测试。
- 翻译语言。
- 优化性能。
- 补类型。
- 写 SQL。
- 写 shell 命令。
- 根据错误日志定位问题。

来源包括：

- 人工标注。
- Git commit / PR 转换。
- 从 benchmark 或在线编程题改写。
- 强模型生成。
- execution-filtered synthetic data。

DeepSeek-Coder-Instruct 使用 2B tokens 的 instruction data。OctoPack 用 Git commits 做 instruction tuning。Magicoder 用 OSS-Instruct 生成 75K synthetic instruction data。OpenCodeInstruct 则把 instruction 样本扩展到 5M 量级，并包含 test cases 和 execution feedback。

### Stage 6：偏好对齐

SFT 教模型“怎么回答”，但不一定教它“哪个回答更好”。偏好对齐处理的是多答案排序。

常见训练方式：

- RLHF：人类比较多个回答，训练 reward model，再用 PPO 优化。
- RLAIF：用强模型或 judge 产生反馈。
- DPO：直接用 chosen/rejected 对训练，不显式训练 reward model。
- rejection sampling：采样多个答案，保留通过测试或评分高的答案再做 SFT。

代码领域的偏好不只看正确性，还看：

- patch 是否小。
- 是否保留用户已有改动。
- 是否过度重构。
- 是否补了测试。
- 是否能解释风险。
- 是否避免危险命令。
- 是否符合项目风格。

这一步让模型更像“懂 review 的工程师”，而不是“只会吐出能跑代码的生成器”。

### Stage 7：执行反馈和 RLVR

代码任务天然适合可验证奖励：

```text
reward = 1，如果所有测试通过
reward = 0，如果测试失败
reward = -1，如果编译失败、超时、越权或破坏环境
```

更细的 reward 可以包括：

- 语法是否通过。
- 单元测试通过比例。
- 是否减少 failing tests。
- 是否通过 hidden tests。
- lint/typecheck 是否通过。
- diff 是否过大。
- 是否触发安全策略。

这类训练常被称为：

- execution feedback。
- compiler feedback。
- reinforcement learning from verifiable rewards。
- RLVR。
- agentic RL。

它和传统 RLHF 的区别是：奖励可以来自编译器、测试框架和 sandbox，不一定要人类评分。

不过它也有风险：

- 模型可能 reward hack，例如硬编码测试。
- 公开测试太弱会鼓励投机。
- 训练环境和真实环境不一致会导致泛化差。
- RL 成本高，生成和执行都很贵。

### Stage 8：Agentic training

面向 coding agent 的训练是近几年快速发展的方向，它进一步把“输出代码”变成“在环境里完成任务”。这还不是所有代码模型都会公开采用的标准阶段，但已经是 SWE-bench、Terminal-Bench 和真实仓库修复能力提升的重要路径。

训练对象不只是最终答案，而是完整行为：

- 什么时候读文件。
- 搜索什么关键词。
- 怎么调用 shell。
- 怎么解释测试失败。
- 是否继续尝试。
- 什么时候停止。
- 如何写最终总结。

这类训练需要 harness：

```text
任务描述
  -> sandbox 仓库
  -> 工具 API
  -> 模型动作
  -> 环境观测
  -> patch / artifact
  -> 测试和评分
  -> reward / trace
```

Qwen3-Coder-Next、Agentic Reinforcement Learning for Real-World Code Repair 这类工作说明，coding agent 的训练正在从静态语料走向“可执行任务环境”。AgentPack 也说明，未来训练数据可能越来越多来自公开仓库中由人类和 agent 共同产生的代码修改。

## 数据清洗为什么重要

代码数据的质量差异极大。常见清洗步骤包括：

| 清洗步骤 | 目的 |
|---|---|
| 许可证过滤 | 降低法律和再分发风险 |
| exact dedup | 删除完全重复文件 |
| near dedup | 删除近似重复和 fork 重复 |
| language detection | 确认文件语言 |
| generated/vendor filter | 移除生成代码、压缩文件、依赖拷贝 |
| size filter | 移除过大、过小或异常文件 |
| syntax check | 提高训练样本可解析性 |
| secret / PII scan | 降低泄漏风险 |
| benchmark decontamination | 避免评测集污染 |
| quality scoring | 按 star、测试、文档、lint、结构等打分 |
| temporal split | 用时间切分训练和评测，降低未来泄漏 |

如果清洗不好，模型会学到：

- 旧 API。
- 易受攻击写法。
- 复制粘贴垃圾代码。
- 许可证敏感代码。
- token、key、密码。
- benchmark 答案。
- auto-generated boilerplate。

## 为什么要混入数学和自然语言

很多 code model 报告都会提到数学和自然语言数据，这不是装饰。

代码任务经常需要：

- 理解复杂题面。
- 做算法推理。
- 读错误信息。
- 解释权衡。
- 写文档。
- 和人交互。
- 拆解需求。

如果只训练源码，模型可能补全很强，但需求理解、调试解释和复杂推理会弱。Qwen2.5-Coder 和 DeepSeek-Coder-V2 这类模型都强调保留或增强 general / math ability。

## 训练方式对能力的影响

| 训练方式 | 最直接提升 |
|---|---|
| 大规模源码预训练 | 语法、补全、API 习惯 |
| 代码相关自然语言 | 从需求到代码、解释能力 |
| FIM | IDE 中间补全和局部修改 |
| repo-level packing | 多文件理解、仓库级一致性 |
| instruction tuning | 按用户要求完成任务 |
| commit / PR tuning | 代码编辑和 patch 生成 |
| synthetic instruction | 扩大任务覆盖面 |
| execution filtering | 减少不能运行的样本 |
| preference tuning | 更符合人类 review 偏好 |
| RLVR / test reward | 提升可验证正确性 |
| agentic trajectories | 工具使用、长任务、失败恢复 |

## 一个现代代码模型的典型训练配方

可以把公开模型的做法抽象成这样：

```text
1. 收集大规模代码和代码相关文本
   - GitHub / Software Heritage
   - docs / issues / PRs / notebooks
   - math / general text

2. 清洗和打包
   - license filter
   - dedup / near-dedup
   - language detection
   - generated/vendor/secret filtering
   - file-level 和 repo-level packing

3. 继续预训练
   - next-token prediction
   - FIM objective
   - long context curriculum

4. 指令微调
   - coding instructions
   - bugfix / explanation / test generation
   - commit message -> patch
   - synthetic question -> solution -> tests

5. 执行验证
   - compile
   - run tests
   - lint/typecheck
   - filter bad generations

6. 偏好或强化学习
   - chosen/rejected pairs
   - DPO / RLHF / RLAIF
   - RLVR with tests

7. Agentic training
   - tool call traces
   - sandbox environments
   - multi-step code repair
   - reward from final repository state
```

## 公开例子速览

| 项目 | 数据和训练特点 |
|---|---|
| OpenAI Codex | GPT 系列模型在公开 GitHub 代码上微调，HumanEval 用来评估从 docstring 生成 Python 函数的能力。 |
| AlphaCode | 先在 GitHub 代码上预训练，再用 CodeContests 这类竞赛编程数据微调，推理时大量采样并用程序行为过滤。 |
| Code Llama | 从 Llama 2 继续预训练，代码数据为主，混入代码相关自然语言和通用自然语言，支持 FIM、长上下文和 Python 专门化。 |
| StarCoder2 | 使用 The Stack v2 和额外高质量来源，强调开放训练数据、许可证治理、SWHID 溯源和大规模多语言代码。 |
| DeepSeek-Coder | 从零训练代码模型，repo-level corpus、16K window、FIM、2T token 预训练和 2B token 指令微调。 |
| DeepSeek-Coder-V2 | 从 DeepSeek-V2 checkpoint 继续用 6T token 训练，扩展语言数和上下文长度。 |
| Qwen2.5-Coder | 使用超过 5.5T token，混合 source code、text-code grounding、synthetic、math、text，并强调清洗、合成和数据配比。 |
| Qwen3-Coder-Next | 面向 coding agent，使用可验证 coding tasks 和可执行环境，通过 mid-training 和 RL 从环境反馈学习。 |
| OctoPack / CommitPack | 从 Git commits 构造 instruction tuning 数据，把 commit message 和 diff 作为意图到修改的映射。 |
| Magicoder | 用开源代码片段生成合成 instruction data，减少纯 LLM 自生成数据的偏差。 |
| OpenCodeInstruct | 5M 代码指令样本，包含问题、解法、测试、执行反馈和质量评估。 |
| CodeUltraFeedback | 用多模型回答和 judge 排序构造代码偏好数据，用于 SFT、RLAIF、DPO。 |
| AgentPack | 收集人类和 coding agent 共同产生的公开代码修改，反映未来 code-edit 数据的一种新来源。 |

## 对 coding agent 的启发

从 harness engineering 角度看，代码能力训练正在发生一个重要迁移：

```text
从：训练模型生成代码文本
到：训练模型在工具环境中完成软件工程任务
```

也就是说，模型越来越需要学习：

- 看哪些文件。
- 如何搜索。
- 何时运行测试。
- 如何利用失败输出。
- 如何保持 diff 可审查。
- 如何遵守权限。
- 如何在长任务中保存状态。

这和 harness 的关系很紧：如果训练时的环境、工具、反馈和真实使用时不同，模型的 agent 能力就会掉得很快。Agentic Reinforcement Learning for Real-World Code Repair 这类工作也指出，训练环境和测试环境不匹配会显著影响泛化。

## 常见误解

### 误解 1：代码模型只是背了 GitHub

代码模型确实从 GitHub 等公开代码中学习大量模式，但强模型不是简单检索器。它还通过 next-token、FIM、instruction tuning、偏好训练和执行反馈学习组合与泛化。

不过，记忆和泄漏风险真实存在，所以数据去重、benchmark decontamination、许可证治理和 secret filtering 很重要。

### 误解 2：只要有更多代码就会更强

数据量重要，但代码数据质量更重要。过时、重复、生成、vendor、低质量代码会占据训练预算并污染模型行为。近年来很多进步来自更好的过滤、合成、执行验证和数据配比，而不只是更多 token。

### 误解 3：HumanEval 高就说明工程能力强

HumanEval 测的是函数级 Python 生成，而且现代模型可能已接近饱和。真实 coding agent 还要会读仓库、改多文件、跑测试、处理环境问题和生成可审查 patch。

### 误解 4：SFT 就够了

SFT 能教会回答格式和常见任务，但对“能否通过测试”“失败后怎么修”“是否 reward hack”“是否遵守权限”不够。可验证反馈和 agentic training 正在变得更重要。

## 如果自己要做一个代码微调数据集

小团队不太可能从零训练 code foundation model，但可以做 repo-specific fine-tuning 或 SFT 数据。一个保守流程：

1. 收集自己的代码、文档、测试和 issue。
2. 只使用有权训练的数据。
3. 删除 secrets、凭据、生产数据和个人信息。
4. 按仓库或时间切分 train / validation / test，避免同一问题泄漏。
5. 构造多种样本：
   - `path + prefix + suffix -> middle`
   - `issue -> patch`
   - `failing test -> fix`
   - `code -> explanation`
   - `function -> tests`
   - `error log -> diagnosis`
6. 用真实命令验证样本，例如 test、lint、typecheck。
7. 保留失败样本作为评估集，不要全部拿去训练。
8. 用小 LoRA/SFT 先试，不要一开始追求大训练。
9. 用私有 eval 比较是否真的提升真实任务。

最重要的是：不要把训练集和评测集混在一起。代码模型很容易“看起来变强”，其实只是记住了题目、测试或项目局部模式。

## 参考资料

- OpenAI: [Evaluating Large Language Models Trained on Code](https://arxiv.org/abs/2107.03374)
- OpenAI: [Introducing Codex](https://openai.com/index/introducing-codex/)
- OpenAI: [Aligning language models to follow instructions](https://openai.com/research/instruction-following/)
- DeepMind: [Competition-Level Code Generation with AlphaCode](https://arxiv.org/abs/2203.07814)
- Meta: [Code Llama: Open Foundation Models for Code](https://arxiv.org/abs/2308.12950)
- BigCode: [StarCoder 2 and The Stack v2](https://arxiv.org/abs/2402.19173)
- BigCode: [The Stack v2 dataset](https://huggingface.co/datasets/bigcode/the-stack-v2)
- DeepSeek: [DeepSeek Coder](https://deepseekcoder.github.io/)
- DeepSeek: [DeepSeek-Coder-V2](https://arxiv.org/abs/2406.11931)
- Qwen: [Qwen2.5-Coder Technical Report](https://arxiv.org/abs/2409.12186)
- Qwen: [Qwen3-Coder-Next Technical Report](https://arxiv.org/abs/2603.00729)
- IBM: [Granite Code Models](https://arxiv.org/abs/2405.04324)
- BigCode: [OctoPack](https://arxiv.org/abs/2308.07124)
- BigCode: [CommitPackFT](https://huggingface.co/datasets/bigcode/commitpackft)
- Magicoder: [Empowering Code Generation with OSS-Instruct](https://arxiv.org/abs/2312.02120)
- OpenCodeInstruct: [A Large-scale Instruction Tuning Dataset for Code LLMs](https://arxiv.org/abs/2504.04030)
- CodeUltraFeedback: [An LLM-as-a-Judge Dataset for Coding Preferences](https://arxiv.org/abs/2403.09032)
- AgentPack: [A Dataset of Code Changes, Co-Authored by Agents and Humans](https://arxiv.org/abs/2509.21891)
- Agentic RL: [Agentic Reinforcement Learning for Real-World Code Repair](https://arxiv.org/abs/2510.22075)
