# Checklist: Review 和完成前收口

- [ ] 本轮实际修改文件已列出。
- [ ] 每个文件为什么修改已说明。
- [ ] 没有超出原需求、Spec 或 Plan。
- [ ] 没有无关重构、改名、格式化或依赖升级。
- [ ] 新增或修改测试已说明。
- [ ] 验证命令已实际运行。
- [ ] 已读取完整验证输出和 exit code。
- [ ] `git status` 或等价文件列表已检查。
- [ ] `git diff` 或等价 diff 已检查。
- [ ] 没有 secret、本地绝对路径、debug 代码或生成产物误入。
- [ ] 必要 docs 已更新。
- [ ] 如果有 reviewer 反馈，只修影响正确性、验收、测试、兼容性或安全的问题。
- [ ] 回复中包含已运行、未运行、剩余风险和下一步。

可用 prompt：

- [`../prompts/review-gate.md`](../prompts/review-gate.md)
- [`../prompts/final-check.md`](../prompts/final-check.md)
