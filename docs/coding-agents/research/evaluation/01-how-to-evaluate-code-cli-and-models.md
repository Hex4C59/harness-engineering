# 如何比较 Code CLI 和模型组合的效果

## 核心结论

不要问“哪个模型最好”，而要问：

> 哪个 `Code CLI + 模型 + 配置 + 权限 + 任务分布`，能在可接受的成本、风险和人工介入下，稳定完成我的真实任务？

在 agent 语境里，模型不是独立工作的。Code CLI 负责上下文选择、文件编辑、工具调用、shell 执行、权限边界、状态恢复、验证循环和交互体验。公开榜单里的一个分数，通常是某个完整配置的结果，而不是裸模型能力。

所以比较标准应该是 **model-harness configuration**，也就是：

```text
评估对象 = 模型 + CLI/harness + 工具面 + 权限策略 + 运行预算 + 任务集 + 评分器
```

## 为什么不能只看模型分数

公开 benchmark 有用，但直接拿来选日常 code CLI 会有几个问题：

1. **模型分数和产品分数不是一回事。** 同一个模型在不同 CLI 里，可能因为上下文检索、编辑协议、工具顺序、验证策略不同而表现差很多。
2. **benchmark 只覆盖能力切片。** Aider Polyglot 更像代码编辑能力测试，SWE-bench 更像仓库 issue 修复，Terminal-Bench 更像终端执行能力，BFCL 更像工具调用格式和选择能力。
3. **公开集可能饱和或污染。** 公开题集进入训练数据、博客、榜单和调参循环后，分数会越来越不像真实泛化能力。OpenAI 在 2026 年 2 月已经明确不再用 SWE-bench Verified 衡量前沿 coding 能力，并建议转向 SWE-bench Pro。
4. **成本经常被忽略。** 一个配置高 1 个百分点，但贵 5 倍、慢 3 倍、需要更多人工盯着，并不一定更适合生产使用。
5. **评测协议不一致会让结果不可比。** 是否允许联网、是否有隐藏测试、是否多次重试、是否有人类提示、是否限制 token 和时间，都会改变结论。

## 评估维度

### 1. 任务完成率

这是最基础的指标，但必须明确“完成”是什么意思。

可用指标：

| 指标 | 含义 |
|---|---|
| `task_success_rate` | 任务最终通过验收的比例 |
| `strict_accept_rate` | 同时通过测试、lint、类型检查、人工质量门槛的比例 |
| `regression_free_rate` | 没有破坏既有行为的比例 |
| `first_try_success_rate` | 第一次提交就成功的比例 |

对 coding agent 来说，不要只看“模型说完成了”。应该看最终仓库状态：测试是否通过、文件是否正确修改、功能是否真实可用。

### 2. 可靠性和方差

LLM agent 有随机性，同一个任务重复跑可能结果不同。只跑一次容易误判。

可用指标：

| 指标 | 含义 |
|---|---|
| `pass@1` | 单次尝试成功率 |
| `pass@k` | 多次尝试中至少一次成功的概率，适合看“可搜索性” |
| `pass^k` | 多次尝试都成功的比例，适合看稳定性 |
| `variance_by_task` | 哪些任务不稳定 |
| `variance_by_config` | 哪些 CLI 或模型组合波动大 |

如果日常使用需要“它一次就靠谱”，`pass@1` 和 `pass^k` 比 `pass@k` 更重要。

### 3. 工程质量

测试通过不等于代码好。Agent 很容易写出刚好过测试但难维护的补丁。

可用指标：

| 指标 | 含义 |
|---|---|
| `minimal_diff_score` | 改动是否集中、必要、可审查 |
| `style_fit_score` | 是否符合项目已有风格 |
| `architecture_fit_score` | 是否遵守模块边界和架构约束 |
| `test_quality_score` | 是否补了有价值的测试，而不是脆弱测试 |
| `review_rework_rate` | 人类评审后需要返工的比例 |

这一类指标通常需要人工评审，或先由 LLM judge 初筛，再抽样人工校准。

### 4. 过程质量

Code CLI 的价值很大一部分在过程，而不是最终输出。

可用指标：

| 指标 | 含义 |
|---|---|
| `context_precision` | 读到的上下文是否真的相关 |
| `tool_call_accuracy` | 工具选择和参数是否正确 |
| `verification_rate` | 修改后是否主动运行合适验证 |
| `retry_quality` | 失败后是否基于反馈修正，而不是盲目重试 |
| `trajectory_waste` | 无效搜索、重复读文件、无意义命令的比例 |

这类指标要依赖 trace，也就是完整的工具调用、文件变更、命令输出和中间状态记录。

### 5. 成本和效率

对长期使用来说，成本不是附属指标，而是核心指标。

可用指标：

| 指标 | 含义 |
|---|---|
| `cost_per_task` | 每个任务平均 API 成本 |
| `cost_per_success` | 每个成功任务的平均成本 |
| `tokens_per_success` | 每个成功任务消耗的 token |
| `median_wall_time` | 中位完成时间 |
| `tool_calls_per_success` | 成功任务平均工具调用次数 |
| `human_minutes_per_success` | 每个成功任务需要的人类时间 |

一个实用排序方式是：

```text
有效性优先：先过滤掉不可靠配置
效率排序：在可靠配置里比较 cost_per_success 和 human_minutes_per_success
```

### 6. 安全和权限控制

代码 agent 会真实修改文件、运行命令、访问网络，安全指标必须单独列出来。

可用指标：

| 指标 | 含义 |
|---|---|
| `unsafe_action_rate` | 未经允许执行危险操作的比例 |
| `secret_exposure_rate` | 泄露或读取不该访问的 secret 的比例 |
| `user_change_preservation` | 是否保留用户已有改动 |
| `permission_compliance` | 是否遵守只读、审批、路径限制 |
| `rollback_needed_rate` | 需要人工回滚的比例 |

安全指标一般不适合加权平均。一旦触发严重安全问题，就应该直接判定配置不可用。

### 7. 可观测性和可恢复性

一个 CLI 失败不可怕，可怕的是失败后你不知道它为什么失败，也没法继续。

可用指标：

| 指标 | 含义 |
|---|---|
| `trace_completeness` | 是否记录模型调用、工具调用、文件 diff、命令输出 |
| `resume_success_rate` | 中断后能否恢复任务 |
| `failure_attribution_rate` | 能否定位失败属于模型、工具、环境、评分器还是任务描述 |
| `artifact_reproducibility` | 是否能复现实验结果和最终补丁 |

这部分是 code CLI 的产品能力，不是模型能力。

### 8. 使用体验

如果两个配置分数接近，日常体验会决定真实生产力。

可看这些问题：

- Diff 是否容易审查？
- 是否能很好处理多文件任务？
- 是否能暂停、恢复、撤销？
- 是否能解释它为什么做某个修改？
- 是否能和 IDE、终端、Git、测试命令顺畅配合？
- 是否能把失败转成下一次可复用的规则、测试或文档？

## 公开 benchmark 怎么用

公开 benchmark 更适合做“能力地图”，不适合单独做采购或迁移决策。

| Benchmark | 主要测什么 | 适合回答什么问题 | 局限 |
|---|---|---|---|
| Aider Polyglot | 多语言代码编辑、补丁格式、测试通过率 | 某模型在 Aider 风格代码编辑中是否好用 | 偏小型练习题，不代表完整仓库任务 |
| SWE-bench / SWE-bench Pro | 真实 GitHub issue 修复 | agent 是否能在现有仓库里定位并修 bug | 公开版本有污染和饱和风险，测试质量也会影响判断 |
| Terminal-Bench | 终端环境里的真实操作任务 | CLI agent 是否会用 shell、文件系统、工具链完成长任务 | 不一定覆盖你的业务代码风格 |
| Berkeley Function Calling Leaderboard | 函数选择、参数生成、多工具调用 | 模型是否擅长工具调用协议 | 更偏模型/tool calling，不等于完整 coding agent |
| τ-bench | 多轮用户交互、业务规则、API 工具使用 | agent 是否能在对话中遵守政策并更新环境状态 | 领域有限，常用于 customer-support 类任务 |
| MLE-bench | Kaggle 风格机器学习工程任务 | agent 是否能做长时间实验、训练、调参和提交 | 成本高，偏 ML 工程 |
| AlphaEval / WildClawBench / SWE-Bench Mobile / Harness-Bench | 完整 agent 产品、真实 CLI harness、长任务和生产任务 | CLI 与模型组合在真实环境里的差异 | 多数仍是较新的研究 benchmark，应看方法而不是迷信单个分数 |

一个健康用法是：

1. 用公开 benchmark 筛掉明显不适合的模型或 CLI。
2. 用自己的任务集做最终比较。
3. 公开分数只作为先验，不作为上线标准。

## 自己做评测的最小协议

### Step 1：定义任务分布

先写清楚你日常到底要 code CLI 做什么。比如：

| 类别 | 示例任务 |
|---|---|
| Bug 修复 | 给定 failing test 或错误日志，修复问题 |
| 小功能 | 按需求添加一个 API、页面、命令或配置 |
| 重构 | 在不改变行为的前提下移动模块或整理结构 |
| 测试补齐 | 为已有行为补单元测试或端到端测试 |
| 文档维护 | 根据代码变更更新 README、设计文档或迁移说明 |
| 运维脚本 | 修复 CI、构建脚本、Dockerfile、release 流程 |

每类任务都要有 easy、medium、hard。不要只挑 agent 擅长的任务。

### Step 2：构造任务卡

每个任务至少包含：

```yaml
id: bugfix-auth-token-refresh-001
category: bugfix
repo_snapshot: snapshots/auth-service-2026-05-01.tar.gz
prompt: "刷新 token 后偶发 401，请定位并修复。"
allowed_tools:
  - read_file
  - edit_file
  - shell
  - rg
network: disabled
budget:
  wall_time_minutes: 30
  max_cost_usd: 3
acceptance:
  required_commands:
    - npm test
    - npm run lint
  hidden_checks:
    - token refresh race condition regression
rubric:
  correctness: 40
  regression_safety: 20
  maintainability: 20
  process_quality: 10
  cost_efficiency: 10
```

任务卡要版本化。任务描述、初始仓库、测试、隐藏检查和评分标准都应该固定下来。

### Step 3：锁定运行协议

每次评测都记录：

```yaml
config_id: codex-cli__gpt-5-codex__default__2026-05-29
cli:
  name: Codex CLI
  version: "..."
model:
  name: "..."
  provider: "..."
  parameters:
    temperature: "..."
    reasoning_effort: "..."
permissions:
  filesystem: workspace
  network: disabled
  approval_policy: never
budget:
  wall_time_minutes: 30
  max_turns: 50
  max_cost_usd: 3
task_suite:
  name: internal-code-cli-eval
  version: 2026-05-29
```

没有这些元数据，评测结果以后很难复现。

### Step 4：分开比较模型和 CLI

至少做两组实验：

| 实验 | 固定什么 | 改什么 | 目的 |
|---|---|---|---|
| 模型比较 | 同一个 CLI、同一套权限、同一预算 | 模型 | 看模型差异 |
| CLI 比较 | 同一个模型、同一套任务、同一预算 | CLI/harness | 看工具和 harness 差异 |

如果直接比较 `Claude Code + Claude` 和 `Codex CLI + GPT`，只能知道“完整组合谁更好”，不能知道差异来自模型还是 CLI。

### Step 5：多次运行

建议：

- 粗筛：每个配置每题跑 1 次。
- 复评：候选配置每题跑 3 次。
- 关键上线：高价值任务跑 5 次或更多。

报告时不要只给平均分，也要给不稳定任务列表。

### Step 6：同时使用硬评分和软评分

硬评分适合做 gate：

- 测试通过。
- lint 通过。
- 类型检查通过。
- 构建通过。
- 没有越权操作。
- 没有破坏用户已有改动。

软评分适合做排序：

- 代码是否简单。
- 是否符合项目风格。
- 是否补了正确测试。
- 是否过度设计。
- 是否容易 review。

实践里可以先自动打分，再对样本做人工校准。不要让 LLM judge 成为唯一真相来源。

### Step 7：保存 trace

每次运行至少保存：

- 输入 prompt。
- CLI 和模型配置。
- 工具调用序列。
- shell 命令和输出。
- 文件 diff。
- 最终结果。
- 成本、token、耗时。
- 失败原因标签。

没有 trace，就只能得到“这次没过”，很难改进 harness。

### Step 8：做失败分类

建议用这套标签：

| 失败类型 | 典型表现 |
|---|---|
| `context_failure` | 没读到关键文件，或读了大量无关文件 |
| `planning_failure` | 方向错，拆解错，忽略约束 |
| `tool_failure` | 工具选错、参数错、命令错 |
| `edit_failure` | diff 格式错、改错文件、引入语法错误 |
| `verification_failure` | 没跑测试，或忽略测试失败 |
| `state_failure` | 忘记已有修改、覆盖用户改动、中断后接不上 |
| `overengineering` | 为小问题引入过大抽象 |
| `safety_failure` | 越权、泄露 secret、危险命令 |
| `environment_failure` | 依赖、网络、容器、缓存导致失败 |
| `grader_failure` | 测试或评分器本身不合理 |

失败分类比总分更能指导下一轮改进。

## 推荐评分卡

可以先用这张表比较候选配置：

| 配置 | 成功率 | 严格通过率 | 成本/成功 | P50 耗时 | 人工介入 | 安全失败 | 主要失败类型 |
|---|---:|---:|---:|---:|---:|---:|---|
| CLI A + Model X |  |  |  |  |  |  |  |
| CLI A + Model Y |  |  |  |  |  |  |  |
| CLI B + Model X |  |  |  |  |  |  |  |

如果需要一个综合分，可以用：

```text
综合分 = 任务完成 40%
      + 工程质量 20%
      + 可靠性 15%
      + 成本效率 10%
      + 过程质量 10%
      + 使用体验 5%
```

但安全问题不要进加权平均。出现严重越权、破坏用户改动、泄露 secret，应直接判为不可上线。

## 一个最低可用标准

如果只是为个人或小团队选择 code CLI，可以从这个标准开始：

- 至少 20 个真实任务，覆盖 bugfix、feature、refactor、test、docs。
- 每个任务有明确验收命令或人工 rubric。
- 每个候选配置至少跑 1 次，Top 2 配置再跑 3 次。
- 固定 CLI 版本、模型版本、权限、预算、系统提示和任务集版本。
- 同时报告成功率、成本、耗时、人工介入、安全失败和主要失败类型。
- 保存 trace 和最终 diff。
- 每次真实使用中遇到失败，把它沉淀成下一版 eval case。

这套东西不复杂，但会立刻把“感觉哪个好用”变成可比较的工程判断。

## 读榜单时的红旗

看到这些情况要谨慎：

- 只报告成功率，不报告成本、时间、token 或工具调用。
- 没说清楚是裸模型、固定 harness，还是完整产品。
- 没有 CLI 版本、模型版本、prompt、权限和预算。
- 只给总分，没有任务级结果。
- 允许多次重试但不说明 `pass@1`。
- 公开 benchmark 上很高，但没有私有 holdout 或真实任务验证。
- 没有 trace，无法复现。
- 只用 LLM judge，没有硬验证或人工校准。
- 把 SWE-bench Verified 当作 2026 年之后的唯一 coding 标准。

## 和 Harness Engineering 的关系

比较 code CLI 和模型，本质上是在评估 harness：

- 上下文是否选得准。
- 工具是否暴露得合适。
- 状态是否可恢复。
- 权限是否可控。
- 反馈是否能进入下一轮改进。
- 验证是否真的检查最终环境，而不是检查模型话术。

因此，好的评测不是一次排行榜截图，而是一个持续运行的反馈闭环。每次 agent 失败，都应该问：

> 这是模型能力不足，还是 harness 没给它正确上下文、工具、约束、状态和验证？

## 参考资料

- OpenAI: [Evaluation best practices](https://developers.openai.com/api/docs/guides/evaluation-best-practices)
- Anthropic: [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- LangSmith: [Evaluation concepts](https://docs.langchain.com/langsmith/evaluation-concepts)
- Kapoor et al.: [AI Agents That Matter](https://arxiv.org/abs/2407.01502)
- Zhu et al.: [Establishing Best Practices for Building Rigorous Agentic Benchmarks](https://arxiv.org/abs/2507.02825)
- SWE-bench: [Official leaderboards](https://www.swebench.com/)
- OpenAI: [Why SWE-bench Verified no longer measures frontier coding capabilities](https://openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/)
- Terminal-Bench: [Official site](https://www.tbench.ai/)
- Aider: [LLM leaderboards](https://aider.chat/docs/leaderboards/)
- Berkeley Gorilla: [Berkeley Function Calling Leaderboard](https://gorilla.cs.berkeley.edu/blogs/8_berkeley_function_calling_leaderboard.html)
- Yao et al.: [τ-bench](https://arxiv.org/abs/2406.12045)
- OpenAI: [MLE-bench](https://openai.com/index/mle-bench/)
- Lu et al.: [AlphaEval: Evaluating Agents in Production](https://arxiv.org/abs/2604.12162)
- Yao et al.: [Harness-Bench](https://arxiv.org/abs/2605.27922)
- Ding et al.: [WildClawBench](https://arxiv.org/abs/2605.10912)
- Tian et al.: [SWE-Bench Mobile](https://arxiv.org/abs/2602.09540)
