# AGENTS.md

这个项目用于收集和整理 **harness engineering、AI agent、coding agent、agent runtime、agent eval、模型代码能力** 等相关资料、笔记和演示代码。这里不是某个生产应用仓库，而是一个面向长期积累的研究与写作工作区。

## 入口地图

- `README.md`：项目总入口和推荐阅读顺序。
- `docs/harness-engineering/README.md`：harness engineering 主题文档地图。
- `docs/harness-engineering/`：agent harness 方法论、实践和参考资料。
- `docs/coding-agents/`：code CLI、coding agent 插件、个人 TDD 工作流和 Codex 使用经验。
- `docs/ai-models/`：模型代码能力训练、benchmark 和模型报告解读。
- `docs/harness-engineering/references.md`：参考资料和延伸阅读。
- 未来如需保存演示代码，优先放在 `examples/`。
- 未来如需保存原始资料、论文摘录、网页快照或调研素材，优先放在 `materials/` 或 `references/`，不要堆在根目录。

## 文件组织约定

- 根目录保持干净，只放入口文件、项目约定和少量全局配置。
- 成体系的说明文档按主题放在 `docs/<topic>/`，按阅读顺序使用 `NN-topic.md` 命名，例如 `docs/coding-agents/01-how-to-evaluate-code-cli-and-models.md`。
- 新增正式文档后，同步更新 `README.md` 和对应主题目录的 `README.md`。
- 临时草稿不要直接混进正式序列。需要保留时放到 `drafts/`，稳定后再移动到 `docs/`。
- 演示代码按主题分目录，例如 `examples/code-cli-eval/`、`examples/agent-loop/`。每个示例目录应有自己的 `README.md`，说明用途、运行方式和依赖。
- 大文件、下载物、生成产物和可重复生成的缓存不要提交或混入正文目录。

## 写作约定

- 默认使用简体中文写作，保留必要英文术语，例如 `harness`、`agent runtime`、`tool calling`、`eval`。
- 文章应优先沉淀可复用判断框架、概念边界、评估标准和实践清单，而不是只记录一次性的新闻结论。
- 如果引用最新模型、榜单、产品能力或价格，必须标明日期，并优先使用官方来源。
- 不要把公开 benchmark 的单个排名写成长期结论。更推荐记录 benchmark 测什么、怎么用、有什么局限。
- 新增资料时，尽量在主题目录的 `references.md` 或对应文档的“参考资料”部分补充来源链接。
- 避免把 `AGENTS.md` 写成百科全书。这里应只保留入口、约定和协作规则，细节进入 `docs/`。

## 调研约定

- 对可能变化的信息要联网确认，包括模型列表、CLI 能力、价格、排行榜、论文版本和产品文档。
- 优先引用官方文档、论文、项目主页、GitHub 仓库和作者原文。
- 对二手博客、社区讨论和榜单截图要谨慎使用，并在文中标注其局限。
- 调研结论要区分“事实”“推断”和“个人整理框架”。

## 修改约定

- 修改前先查看相关 README 和现有目录结构，沿用已有组织方式。
- 移动文件时同步修正 Markdown 链接。
- 不要删除用户已有资料，除非用户明确要求。
- 不要覆盖用户正在编辑的内容；如果发现同一文件里有不相关改动，应保留并在其基础上继续。
- 这个目录可能不是 Git 仓库，不能假设一定有提交历史可回滚。

## 推荐新增目录

需要扩展时，优先使用这些目录名：

```text
docs/          正式文档
drafts/        草稿和未整理笔记
examples/      可运行演示代码
materials/     原始调研材料、摘录、网页快照
references/    结构化外部参考资料
assets/        图片、图表、截图
```
