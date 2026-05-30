# 依赖和架构变化

agent 不应随便引入依赖或改变架构。

需要先说明：

- 为什么现有能力不够。
- 新依赖解决什么问题。
- 替代方案是什么。
- 维护成本是什么。
- 是否影响构建、部署、许可、安全。

然后写入 `docs/decisions/`。

## 方案比较先于代码

当架构方向不确定、依赖选择不确定、影响范围不确定时，不要先写代码。先比较方案。

可用 prompt：[`../prompts/solution-comparison.md`](../prompts/solution-comparison.md)。

只有当方案、blast radius、reviewable slices 和验证方式都清楚后，才进入实现。
