# Workflow: Learning-first Vibe Coding

这个流程用于“我想用 coding agent 写一个自己还不熟悉的技术栈，同时又想真正学会并能 review”的场景。

典型例子：

```text
我以前主要写 Python。
现在想用 coding agent 写 Rust。
我希望项目能做出来，但不想只得到一坨自己看不懂的代码。
```

## 先判断目标

| 当前目标 | 推荐入口 |
|---|---|
| 只想快速验证想法 | [`../prompts/prototype.md`](../prompts/prototype.md) |
| 想边做边学陌生技术栈 | [`../prompts/learning-first.md`](../prompts/learning-first.md) |
| 已经有大任务，担心 diff 太大 | [`../prompts/reviewable-slices.md`](../prompts/reviewable-slices.md) |
| 已经有 AI 生成代码，但看不懂 | [`../prompts/explain-and-verify.md`](../prompts/explain-and-verify.md) |
| 已经有 diff，需要审查 | [`../prompts/independent-review.md`](../prompts/independent-review.md) |
| Rust diff 需要专项 review | [`../prompts/rust-review.md`](../prompts/rust-review.md) |
| agent 越改越多 | [`../prompts/stop-and-split.md`](../prompts/stop-and-split.md) |
| 想判断是否可长期维护 | [`../checklists/maintainability-gate.md`](../checklists/maintainability-gate.md) |

## 执行节奏

1. 先用学习优先开场，让 agent 降速。
2. 把任务拆成 reviewable slices。
3. 每个 slice 先讲 tiny example，再进入项目代码。
4. 每轮只接受自己能解释的 diff。
5. 完成后做独立 review。
6. 如果看不懂，先 Explain and Verify，不继续加功能。

原则见 [`../principles/learning-first.md`](../principles/learning-first.md)。

## Rust 专项

如果目标技术栈是 Rust，优先使用：

- [`../prompts/rust-learning.md`](../prompts/rust-learning.md)
- [`../prompts/rust-review.md`](../prompts/rust-review.md)

重点 review ownership 边界、错误处理、`clone`、`unwrap`、`unsafe`、trait/generic 抽象和测试覆盖。
