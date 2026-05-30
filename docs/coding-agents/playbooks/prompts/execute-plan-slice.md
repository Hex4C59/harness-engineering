# Prompt: 按已有计划执行一个 Slice

适合已经有 `docs/plans/` 计划，并且计划已经确认后的实现阶段。

```text
我要按照已有计划实现代码。请严格按计划文档执行，不要重新规划大方向，也不要一次做完整个计划。

计划文档：
docs/plans/YYYY-MM-DD-feature-name.md

请先读取：
1. AGENTS.md
2. README.md
3. docs/README.md
4. docs/project-status.md
5. docs/roadmap.md
6. docs/development.md
7. docs/testing.md
8. 上面的计划文档
9. 计划文档关联的 Spec / ADR / architecture 文档
10. 计划文档中点名的相关代码和测试

然后先回复一个很短的执行确认：
- 计划目标
- 非目标
- 关联 Spec / Roadmap item
- 当前要执行的 slice / task
- 本轮预计修改文件
- 本轮验证命令

执行规则：
1. 默认只执行计划里的第一个未完成 slice / task。
2. 如果我指定了 slice，就只执行我指定的 slice。
3. 不要顺手做后续 slice。
4. 不要重写计划大方向；如果发现计划有问题，先停下来说明冲突和建议。
5. 先写失败测试，并运行局部测试确认它因为预期原因失败。
6. 再写最小实现，让这个测试通过。
7. 必要时做小范围重构，但不能扩大行为范围。
8. 跑本轮局部验证命令。
9. 如果本轮 slice 完成，再跑 ./scripts/check，除非 docs/testing.md 明确说明本阶段只需局部验证。
10. 更新计划文档里的任务状态和 docs/project-status.md。
11. 如果实现中发现需求意图或长期方向不成立，停下来建议更新 Spec 或 Roadmap，不要在实现里暗改方向。

完成时请告诉我：
- 完成了哪个 slice / task。
- 修改了哪些文件。
- 新增或修改了哪些测试。
- 运行了哪些验证命令，结果如何。
- 计划中下一个建议执行的 slice 是什么。
- 是否发现计划需要调整。
```

如果要指定某一个 slice，可以使用短版：

```text
请按照 docs/plans/YYYY-MM-DD-feature-name.md 执行指定 slice，不要做其它 slice。

指定 slice：

请先确认这个 slice 的目标、影响文件和验证命令。
确认后按 TDD 执行：失败测试 -> 最小实现 -> 局部验证 -> 必要重构 -> ./scripts/check -> 更新计划和 project-status。
如果发现必须扩大范围，请停下来说明原因，不要直接扩大实现。
```
