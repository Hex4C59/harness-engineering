# 工具箱扩展指南

这个工具箱应该长期演化，但不要重新长成几篇大杂烩。新增经验时先判断它是什么，再决定放在哪里。

## 放哪里

| 新内容类型 | 放置位置 |
|---|---|
| 一段可以直接复制给 agent 的提示词 | `prompts/` |
| 一套完整做事流程 | `workflows/` |
| 一个要复制到项目里的文件模板 | `templates/` |
| 一个初始化、review、完成或维护性检查清单 | `checklists/` |
| 一个长期稳定的判断原则 | `principles/` |
| 维护工具箱自身的规则、命名和扩展约定 | `meta/` |
| 工具箱 workflow / prompt / skill 为什么演化 | `meta/harness-design-log.md` |
| 还没验证、只是摘录或临时灵感 | 本仓库的 `drafts/`、`materials/` 或相关 research 文档 |

## 新增流程

1. 先搜索是否已有同类内容：

   ```bash
   rg "关键词" docs/coding-agents/agent/playbooks
   ```

2. 如果是已有 prompt 的变体，优先更新原 prompt，不新建重复文件。
3. 如果是新场景，新增一个 workflow，并在 `README.md` 的场景表里加入入口。
4. 如果 workflow 需要 prompt，在 `prompts/` 新增独立文件，workflow 只链接它。
5. 如果新增了模板或清单，在对应目录中补一个简短 README 或更新相关入口。
6. 最后检查旧链接和目录链接：

   ```bash
   rg "新文件名|旧文件名" docs/coding-agents/agent/playbooks README.md docs/coding-agents/README.md
   ```

## 文件粒度

- 一个 prompt 文件只解决一个场景。
- 一个 workflow 文件只描述一类工作流。
- 一个 template 文件可以收纳一组同类项目文件模板。
- 原则文档要短，只保留判断框架，不写长篇调研。
- meta 文档只维护这个工具箱本身，不承载具体工作流。
- 新增或升级 gate 前，优先在 `meta/harness-design-log.md` 记录失败模式、触发条件和验证方式。

## 写作标准

- 默认简体中文，保留必要英文术语。
- 优先写可执行步骤、判断规则和检查清单。
- 避免复制同一段 prompt；通用 prompt 以 `prompts/` 为单一事实源。
- 如果引用最新模型、产品能力、价格或 benchmark，必须标明日期并优先用官方来源。
- 新实践还没有经过项目验证时，先放到 research 或 draft，不要直接升级成原则。
