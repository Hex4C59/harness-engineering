# Workflow: Review 和完成前收口

任务完成不等于实现写完。

```text
实现完成 + 测试通过 + diff review + 状态文档更新 = 任务完成
```

## 收口顺序

1. 运行相关测试或 `./scripts/check`。
2. 读取完整输出和 exit code。
3. 检查 diff 是否只包含本轮范围。
4. 做 Review Gate：[`../prompts/review-gate.md`](../prompts/review-gate.md)。
5. 修复影响正确性、验收、测试、兼容性或安全的问题。
6. 更新必要 docs。
7. 用 [`../prompts/final-check.md`](../prompts/final-check.md) 收口。

完成前检查清单见 [`../checklists/review-and-finish.md`](../checklists/review-and-finish.md)。

## Commit 边界

commit 是项目历史的状态边界，不只是“保存一下”。提交粒度应该服务于 review、回滚和理解历史。

推荐拆分方式：

| 变更类型 | 提交边界 |
|---|---|
| 新项目初始化 | harness、脚本、文档和基础 `.gitignore` 可以作为首次提交 |
| 新功能 | 一个完成的 reviewable slice 一次提交 |
| Bug 修复 | 复现测试和最小修复放在同一提交 |
| 重构 | 不改变行为的重构单独提交 |
| 文档 | 文档-only 改动单独提交 |
| 依赖或工具链 | 依赖、锁文件和配置变更单独提交 |
| 格式化 | 大范围格式化不要混入功能提交 |

agent 可以建议 commit message，但不要直接执行 `git commit`，除非用户明确授权。
