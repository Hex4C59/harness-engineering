# Prompt: Rust 学习 Slice

适合用 AI 写 Rust，但你还不熟悉 Rust。

```text
我要用 Rust 实现这个 slice，但我的目标是学习 Rust。

slice 目标：
当前代码位置：
我已理解的 Rust 概念：
我还不理解的 Rust 概念：

请严格按流程：
1. 先说明这个 slice 涉及哪些 Rust 概念，例如 ownership、borrowing、lifetime、Result、trait、async、Send/Sync。
2. 给一个 30 行以内的 tiny example。
3. 解释 Python 直觉和 Rust 语义的差异。
4. 让我预测关键 move / borrow / clone 行为。
5. 我确认后，再改项目代码。
6. 不要用 clone 绕过 borrow checker，除非解释成本和理由。
7. 不要引入 unsafe。
8. 不要引入新 crate，除非先列 trade-off 并等我确认。
9. 完成后列出我应该 review 的 ownership 边界、错误处理、测试覆盖和潜在维护风险。

本轮 diff 上限：80 行。
```
