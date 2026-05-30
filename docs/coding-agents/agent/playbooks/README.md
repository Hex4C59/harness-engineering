# Coding Agent Harness Toolkit: Playbooks

这个目录是 **Coding Agent Harness Toolkit** 的 playbooks 层，可以复制到新项目里使用。它不再按“长文章”组织，而是按使用场景、可复制 prompt、项目模板、检查清单和稳定原则组织。

核心原则：

```text
不可 review 的 diff = 未完成。
先探路，再拆小步；先测试，再实现；先 review，再完成。
```

## 先按场景选入口

| 当前要做什么 | 入口 |
|---|---|
| 从零开一个新项目 | [`workflows/new-project-bootstrap.md`](workflows/new-project-bootstrap.md) |
| 给既有项目接入 agent harness | [`workflows/existing-project-onboarding.md`](workflows/existing-project-onboarding.md) |
| 日常做功能、修 bug、重构 | [`workflows/everyday-development.md`](workflows/everyday-development.md) |
| 专门修 bug | [`workflows/bugfix.md`](workflows/bugfix.md) |
| 完成前 review 和收口 | [`workflows/review-and-finish.md`](workflows/review-and-finish.md) |
| 用陌生技术栈边做边学 | [`workflows/learning-first-vibe-coding.md`](workflows/learning-first-vibe-coding.md) |
| 用费曼学习法学习编程和仓库资料 | [`workflows/feynman-learning-for-programming.md`](workflows/feynman-learning-for-programming.md) |
| 只想复制一段 prompt | [`prompts/`](prompts/) |
| 想初始化项目文档模板 | [`templates/project-harness-files.md`](templates/project-harness-files.md) |
| 想检查是否做完 | [`checklists/`](checklists/) |
| 想理解背后的规则 | [`principles/`](principles/) |

## 目录结构

```text
playbooks/
├── README.md
├── workflows/    场景流程：什么时候做什么
├── prompts/      可复制提示词：单一事实源
├── templates/    可复制到项目里的文件模板
├── checklists/   初始化、开发、review 和维护性清单
├── principles/   稳定原则和判断框架
└── meta/         工具箱维护、扩展和命名约定
```

## 谁来读

这个目录不是全部都给 agent 自动读取。默认先由人选择入口，再让 agent 读取当前任务需要的最小材料。

| 目录 | 主要读者 | 什么时候用 | 用法 |
|---|---|---|---|
| `workflows/` | 人为主，agent 可按需读 | 选择当前任务怎么推进 | 人先选一个 workflow；需要 agent 执行时，让 agent 只读对应 workflow |
| `prompts/` | agent 为主 | 需要一段可直接执行的任务指令 | 人复制给 agent，或在 workflow 中点名让 agent 使用 |
| `checklists/` | 人和 agent 共同使用 | 开始前、计划后、完成前、维护性评估 | 人用它做 gate；也可以让 agent 按 checklist 自检并报告证据 |
| `templates/` | 人为主，agent 可辅助填充 | 初始化新项目或既有项目 harness | 人决定采用哪些模板；agent 通过只读审计补事实，不确定处保留 `待确认` |
| `principles/` | 人为主，agent 决策前按需读 | 需要判断依赖、架构、TDD、reviewability、学习债时 | 平时人读来建立判断；高风险任务前点名让 agent 读取对应原则 |
| `meta/` | 维护者为主 | 扩展、重命名、调整工具箱结构 | 新增 workflow / prompt / checklist / template 前先看 |

简化记法：

```text
workflows = 人选路线，agent 可执行路线
prompts = 给 agent 的话
checklists = 人和 agent 一起过关
templates = 复制到项目里的文件雏形
principles = 人的判断标准，agent 高风险决策前按需读
meta = 维护这个工具箱
```

## 使用方式

复制整个 `playbooks/` 目录到新项目，或只复制其中需要的 prompt / template。日常使用时不要从头读完整目录，先打开本页，然后按场景进入对应 workflow。

当某个 workflow 需要一段可复制提示词时，它只链接到 `prompts/`，不在正文重复粘贴。这样后续修改 prompt 时只需要改一个地方。

## 扩展方式

后续看到新的 coding agent 实践经验，不要直接塞进某篇长文。先按类型归类，再放到对应目录。具体规则见 [`meta/extension-guide.md`](meta/extension-guide.md)。
