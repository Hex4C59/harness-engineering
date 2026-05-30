# Prompt: 新项目 Bootstrap

适合从空目录初始化一个可被 coding agent 接手的新项目 harness。

```text
我要初始化一个新项目。请你按照当前这份流程文档执行。

如果你已经能读取当前文档，请直接按本文执行。
如果你还没有看到这份文档，请先让我提供它的路径或内容，不要猜测流程。

项目名称：
项目类型：
技术栈：如果还没确定，请先执行“技术栈选择”
项目目标：
暂时不做：
技术选型决策：如果刚做过选型，请把推荐结论贴在这里

请先不要写业务代码，只做项目 harness 初始化：
1. 如果当前目录还不是 Git 仓库，执行 git init。
2. 创建 .gitignore，写入通用忽略项，并按技术栈补充编译产物、依赖目录、缓存、日志、覆盖率和本地环境文件。
3. 创建 AGENTS.md。
4. 创建 README.md。
5. 创建 docs/README.md。
6. 创建 docs/roadmap.md，用于长期方向、Now/Next/Later 和暂时不做。
7. 创建 docs/project-status.md，用于当前状态、active work、最近验证和下一步。
8. 创建 docs/development.md。
9. 创建 docs/testing.md。
10. 创建 docs/architecture/overview.md。
11. 创建 docs/decisions/README.md。
12. 如果技术栈已经确定，创建 docs/decisions/0001-choose-technology-stack.md，记录技术选型决策；如果技术栈未定，只在 docs/project-status.md 标明待确认。
13. 创建 docs/specs/README.md，用于复杂功能的 Spec；小项目可以先只保留 README，具体 spec 以后再写。
14. 创建 docs/plans/README.md，用于单个任务的 Execution Plan。
15. 创建 docs/troubleshooting.md。
16. 创建 scripts/bootstrap、scripts/dev、scripts/test、scripts/check。
17. 如果技术栈已经确定，同时把技术栈、工具链、包管理器、常用命令和 Git 提交约定写入 docs/development.md。
18. 如果当前技术栈已有 package.json、Cargo.toml、pyproject.toml 等配置，请把脚本接到真实命令；如果还没有，请先写占位命令并在 docs/development.md 标明待补。
19. 最后运行一次能运行的最小验证命令，检查 git status，并把结果写入 docs/project-status.md。

完成时请告诉我：
- 创建了哪些文件。
- 是否初始化了 Git 仓库。
- .gitignore 覆盖了哪些产物和本地文件。
- 还缺哪些技术栈信息。
- 是否创建了技术选型 ADR。
- 建议的首次提交拆分和 commit message；不要直接 git commit，除非我明确授权。
- 下一步应该做什么。
- 如果技术栈还未确定，请停止在 harness 占位层，不要初始化具体框架。
```
