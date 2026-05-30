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
2. 创建 .gitignore，写入通用忽略项，并按技术栈补充编译产物、依赖目录、缓存、日志、覆盖率和本地环境文件。默认把外部复制进来的 docs/agent/playbooks/ 作为本地工具箱忽略掉；默认忽略 docs/agent/skills/ 和 docs/agent/mcp/ 下除 README.md 之外的本地资产、真实配置和凭据，除非用户明确希望 vendoring；但不要忽略 AGENTS.md、README.md、docs/README.md、docs/project-status.md、docs/development.md、docs/testing.md、docs/roadmap.md、docs/agent/README.md、docs/agent/skills/README.md、docs/agent/mcp/README.md 等项目专属 harness 文档。
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
16. 创建 docs/agent/README.md，说明 agent 协作入口以及 playbooks、skills、mcp 的位置和边界。
17. 创建或保留 docs/agent/playbooks/。如果该目录已经存在，说明用户可能已经预先复制了 playbooks 工具箱，必须保留现有内容，不要删除、覆盖或重复复制；只检查是否有 README.md、workflows、prompts、checklists、templates、principles 和 meta 等入口。如果目录不存在，并且当前能访问完整 playbooks 工具箱源目录，请把 workflows、prompts、checklists、templates、principles 和 meta 复制进去；如果只能看到当前 prompt，不能访问源目录，不要编造内容，只创建 docs/agent/playbooks/README.md 占位，并在 docs/project-status.md 标明待补。
18. 创建或保留 docs/agent/skills/README.md，说明 skills 是可选的项目级或个人能力模板，默认不启用高风险操作。
19. 创建或保留 docs/agent/mcp/README.md，说明 MCP 是可选工具连接配置示例，默认不提交真实凭据、不启用高权限服务。
20. 在 AGENTS.md、docs/README.md 和 docs/agent/README.md 中写入 docs/agent/playbooks/、docs/agent/skills/ 和 docs/agent/mcp/ 的入口说明：日常开发、bugfix、review 和完成前收口应优先按对应 workflow / prompt 执行；skills 复制进项目后只是资产，若要自动触发，需要安装到具体 agent runtime 的 skills 目录，未安装时可以让 agent 直接阅读对应 SKILL.md；MCP 目录只保存示例和说明，真实启用在个人或环境级配置中完成；同时说明这些目录默认是本地 agent 资产，可由用户复制或更新，不作为项目专属文档追踪。
21. 创建 scripts/bootstrap、scripts/dev、scripts/test、scripts/check。
22. 如果技术栈已经确定，同时把技术栈、工具链、包管理器、常用命令和 Git 提交约定写入 docs/development.md。
23. 如果当前技术栈已有 package.json、Cargo.toml、pyproject.toml 等配置，请把脚本接到真实命令；如果还没有，请先写占位命令并在 docs/development.md 标明待补。
24. 最后运行一次能运行的最小验证命令，检查 git status，并把结果写入 docs/project-status.md。

完成时请告诉我：
- 创建了哪些文件。
- 是否初始化了 Git 仓库。
- .gitignore 覆盖了哪些产物和本地文件。
- 是否忽略了 docs/agent/playbooks/ 以及 docs/agent/skills/、docs/agent/mcp/ 中除 README.md 之外的本地资产，哪些项目专属 harness 文档会被 Git 追踪。
- docs/agent/playbooks/ 是新建、复制、保留已有内容还是只创建占位；如果只是占位，说明缺少的来源。
- 还缺哪些技术栈信息。
- 是否创建了技术选型 ADR。
- 建议的首次提交拆分和 commit message；不要直接 git commit，除非我明确授权。
- 下一步应该做什么。
- 如果技术栈还未确定，请停止在 harness 占位层，不要初始化具体框架。
```
