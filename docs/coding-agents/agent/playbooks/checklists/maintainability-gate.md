# Checklist: Maintainability Gate

适合判断项目是否已经从 AI-assisted prototype 进入 personally maintainable software。

- [ ] 我能解释核心模块的数据流和控制流。
- [ ] 我能指出关键不变量和失败路径。
- [ ] 项目有 `README.md`。
- [ ] 项目有 `docs/development.md` 或等价开发说明。
- [ ] 项目有 `docs/testing.md` 或等价测试说明。
- [ ] 项目有架构边界说明。
- [ ] 项目有稳定的 test / check 命令。
- [ ] 核心行为有测试覆盖。
- [ ] 高风险代码经过人工或独立 reviewer 审查。
- [ ] 不存在我完全看不懂但很核心的抽象。
- [ ] 不存在未被审查的生产资源、secret、权限、数据删除、迁移或部署风险。

状态判断：

| 状态 | 含义 |
|---|---|
| prototype | 能跑，但不能长期维护 |
| partially maintainable | 核心路径可解释，但文档、测试或 review 有缺口 |
| personally maintainable | 主要行为可解释、可验证、可 review、可接手 |

可用 prompt：[`../prompts/independent-review.md`](../prompts/independent-review.md)。
