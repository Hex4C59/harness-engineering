# Workflow: 用费曼学习法学习编程和仓库资料

整理日期：2026-05-30

这个流程用于三类场景：

- 想真正学会一个编程概念，而不是只看懂一段解释。
- 想用 coding agent 边做边学，同时避免学习债。
- 想系统学习本仓库里的 harness engineering、coding agent、agent runtime、eval 和模型能力资料。

一句话：

```text
费曼学习法不是把话说得幼稚，而是用简单解释暴露理解缺口，再用验证把缺口补上。
```

## 核心定义

费曼学习法常见版本可以整理成四步：

1. 选一个具体概念。
2. 闭卷用自己的话解释给初学者。
3. 标出讲不清、讲不准、只能背原文的地方。
4. 回到资料补洞，再重写成更清楚的解释。

更适合编程和 coding agent 的版本是：

```text
Select -> Explain -> Gap -> Re-read -> Tiny Example -> Verify -> Write Back
```

也就是：

1. Select：选一个小概念，不选一个大领域。
2. Explain：闭卷白话解释。
3. Gap：找出解释里的含糊处。
4. Re-read：带着缺口回到资料、源码、文档或测试。
5. Tiny Example：写一个最小例子。
6. Verify：运行、测试、调试或 review。
7. Write Back：把修正后的理解写成笔记、规则或 prompt。

## 边界说明

网上常把这套方法说成 Richard Feynman 本人发明的固定四步法。更稳妥的说法是：这是一套后人基于 Feynman 式理解和讲解习惯整理出来的学习流程。

它可靠的部分不在名人故事，而在几个稳定的学习机制：

| 机制 | 在费曼法里怎么体现 | 在编程学习里怎么体现 |
|---|---|---|
| 检索练习 | 闭卷解释，而不是边看边复述 | 不看答案写出 API、流程或概念图 |
| 自我解释 | 解释每一步为什么成立 | 解释代码为什么能跑、为什么会失败 |
| 以教促学 | 面向初学者重写 | 写给未来的自己或队友看 |
| 间隔复习 | 过一段时间重新解释 | 隔天重写 tiny example 或修同类 bug |
| 反馈验证 | 找出解释和事实的偏差 | 跑测试、看错误、读 diff、做 review |

## 什么时候用

适合：

- 学陌生语言、框架、库或工程实践。
- 阅读概念密集的技术文章。
- 理解 AI 生成代码和解释。
- 把一次调研沉淀成可复用判断框架。
- 学习本仓库这种研究和写作型资料库。

不适合单独使用：

- 只需要临时查一个命令。
- 高风险生产改动的最终验收。
- 需要系统训练的大型技能，只靠一次复述。

对高风险任务，费曼法只能帮助理解，不能替代测试、review、权限边界和回滚方案。

## 学编程的最小循环

```text
概念 -> 白话解释 -> tiny example -> 自己重写 -> 制造错误
-> 调试解释 -> 项目小切片 -> 测试验证 -> 复盘沉淀
```

### 1. 一次只选一个概念

坏例子：

```text
我今天学习 Rust。
```

好例子：

```text
我今天只学习 Rust ownership 里 move 和 borrow 的区别。
```

选题越小，越容易暴露真实缺口。这个原则和 [`../principles/learning-first.md`](../principles/learning-first.md) 一致：

```text
一次只学习一个概念。
一次只改变一个行为。
一次只接受自己能解释的 diff。
```

### 2. 先写白话解释

解释时不用追求完整，先追求可检查。

模板：

```text
我以为这个概念是：
它解决的问题是：
如果没有它，会发生什么：
一个最小例子是：
我还说不清的是：
```

如果一句话里出现很多术语，要继续解释术语。比如不能只写：

```text
ownership 是 Rust 的内存安全机制。
```

更好的解释是：

```text
ownership 决定一个值当前由谁负责使用和释放。
当值被 move 到新变量后，旧变量不能继续使用。
这样 Rust 可以在不依赖 GC 的情况下避免很多悬垂引用和重复释放问题。
```

### 3. 写 tiny example

每个新概念先写 10-40 行最小例子，再进入项目代码。

tiny example 要能回答三件事：

- 最小 happy path 是什么？
- 常见错误是什么？
- 错误信息或测试失败说明了什么？

示例任务：

```text
用 20 行以内代码展示 move 以后旧变量不能再使用。
再改成 borrow 版本。
解释编译器分别在保护什么。
```

### 4. 故意弄坏一次

只看正确例子容易形成理解错觉。学编程时要故意制造错误：

- 删除一个 `await`。
- 去掉一个错误处理分支。
- 把 borrow 改成 move。
- 把同步调用放进异步路径。
- 把测试断言改弱，看它是否还能错误通过。

然后解释：

```text
我预期它怎么坏？
它实际怎么坏？
错误信息支持哪个判断？
我的原解释哪里需要修正？
```

### 5. 进入项目时只做 reviewable slice

从 tiny example 进入真实项目时，必须缩小改动：

```text
只改变一个行为。
只触碰必要文件。
能单独测试或验证。
人工 5-10 分钟能 review。
```

这和 [`../principles/reviewable-slices.md`](../principles/reviewable-slices.md) 的核心判断一致：

```text
不可 review 的 diff = 未完成。
```

### 6. 完成前要有验证证据

解释清楚不等于完成。编程学习里的费曼法必须接上验证：

```text
Identify -> Run -> Read -> Verify -> Claim
```

也就是先说明什么命令能证明理解或改动成立，再实际运行、读取输出、确认输出支持结论。这个流程来自 [`../principles/tdd-and-verification.md`](../principles/tdd-and-verification.md)。

## 和 coding agent 一起用

coding agent 可以是 tutor，也可以变成代工。区别在工作流。

推荐 prompt：

```text
我想用费曼学习法学习 <概念>。
请先给我一个 20 行以内的 tiny example。
然后问我 5 个检查理解的问题。
等我回答后，指出我的解释哪里含糊、哪里错误、哪里缺少边界条件。
不要直接生成完整项目实现。
```

如果已经有 AI 生成的代码：

```text
请用学习优先方式解释这个 diff。
先列出这个 diff 改变了哪个行为。
再用 tiny example 解释涉及的新概念。
最后把解释分成事实、推断和假设。
不要把“测试通过”说成“设计一定正确”。
```

接受 AI 输出前，问自己：

```text
我能闭卷解释这个 diff 吗？
我知道它怎么验证吗？
我知道它坏了该从哪里查吗？
```

如果任一答案是否定的，先回到解释、tiny example 或 reviewable slice，不继续加功能。

## 学习本仓库的路线

这个仓库不是普通应用代码库，而是 harness engineering、coding agent、AI agent、模型能力和 LLM API 的研究与写作工作区。学习目标不是背目录，而是建立四张地图：

| 地图 | 要回答的问题 |
|---|---|
| Harness Engineering | agent 可靠工作需要哪些模型之外的系统？ |
| Coding Agents | 人如何和 coding agent 协作、review、拆分和验证？ |
| AI Models | 模型代码能力、reasoning 和 benchmark 应该怎么理解？ |
| LLM API | 模型 API 如何进入生产系统？ |

### 7 天学习计划

第 1 天：建立仓库地图

- 读 [`../../../../../README.md`](../../../../../README.md)。
- 读 [`../../../../../AGENTS.md`](../../../../../AGENTS.md)。
- 读 [`../../../README.md`](../../../README.md)。

费曼输出：

```text
这个仓库为什么不是资料堆，而是 agent engineering 的知识系统？
```

第 2 天：理解 harness engineering

- 读 [`../../../../harness-engineering/foundations/01-what-is-harness-engineering.md`](../../../../harness-engineering/foundations/01-what-is-harness-engineering.md)。

费曼输出：

```text
用 300 字解释 Agent = Model + Harness。
举一个 coding agent 的例子。
```

第 3 天：理解 coding agent playbook

- 读 [`../README.md`](../README.md)。
- 读 [`existing-project-onboarding.md`](existing-project-onboarding.md)。

费曼输出：

```text
为什么既有项目接入 agent harness 时，第一步是发现秩序，而不是重建秩序？
```

第 4 天：理解 reviewable slices

- 读 [`../principles/reviewable-slices.md`](../principles/reviewable-slices.md)。

费曼输出：

```text
解释“不可 review 的 diff = 未完成”。
给一个好 slice 和一个坏 slice 的例子。
```

第 5 天：理解 TDD 和验证

- 读 [`../principles/tdd-and-verification.md`](../principles/tdd-and-verification.md)。

费曼输出：

```text
解释 Identify -> Run -> Read -> Verify -> Claim。
它如何防止 AI 假完成？
```

第 6 天：理解学习债和 AI review 边界

- 读 [`learning-first-vibe-coding.md`](learning-first-vibe-coding.md)。
- 读 [`../../../research/workflows/04-vibe-coding-review-and-learning-debt.md`](../../../research/workflows/04-vibe-coding-review-and-learning-debt.md)。

费曼输出：

```text
为什么 AI 降低了产出代码的门槛，但没有自动降低判断代码质量的门槛？
```

第 7 天：整理综合笔记

写一页笔记：

```text
我理解的 coding agent 可靠性：
从模型能力到 harness、测试、review 和学习债
```

要求：

- 不复制原文。
- 每个概念都给一个例子。
- 每个判断标注事实、推断或个人框架。
- 写出下一步要补的缺口。

## 单篇文档学习模板

以后读这个仓库里的任何一篇文档，都可以复制这个模板：

```text
文档：
我以为它在讲：
它真正回答的问题：
3 个核心概念：
我能给初学者的解释：
我卡住的地方：
我回查后的修正：
这个概念能怎么用于真实项目：
我还不能 review 的风险点：
下一篇该读：
```

## 完成标准

一次费曼学习循环完成，不是因为“看完了”，而是满足：

- 能闭卷解释核心概念。
- 能举一个最小例子。
- 能说出至少一个常见误解。
- 能说明如何验证。
- 能把修正后的理解写成笔记、prompt、测试或项目规则。

如果学的是编程概念，还要额外满足：

- tiny example 能运行。
- 至少故意制造过一次错误。
- 能解释错误信息或失败测试。
- 进入项目时只做了一个 reviewable slice。

## 参考资料

- [Bucknell Teaching & Learning Center: Feynman Technique](https://www.bucknell.edu/sites/default/files/teaching_learning_center/feynmantechnique.pdf)
- Roediger, H. L., & Karpicke, J. D. (2006). [Test-Enhanced Learning: Taking Memory Tests Improves Long-Term Retention](https://doi.org/10.1111/j.1467-9280.2006.01693.x).
- Dunlosky, J., Rawson, K. A., Marsh, E. J., Nathan, M. J., & Willingham, D. T. (2013). [Improving Students' Learning With Effective Learning Techniques](https://www.psychologicalscience.org/journals/pspi/1529100612453266/).
- Chi, M. T. H., de Leeuw, N., Chiu, M. H., & LaVancher, C. (1994). [Eliciting Self-Explanations Improves Understanding](https://doi.org/10.1207/s15516709cog1803_3).
- Fiorella, L., & Mayer, R. E. (2013). [The relative benefits of learning by teaching and teaching expectancy](https://doi.org/10.1016/j.cedpsych.2013.06.001).
