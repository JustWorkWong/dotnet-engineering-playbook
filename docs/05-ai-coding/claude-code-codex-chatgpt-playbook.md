# Claude Code / Codex / ChatGPT 生产实践手册

> 这篇文档承接 [`ai-coding-workflow-core.md`](./ai-coding-workflow-core.md)，重点不再是“AI coding workflow 是什么”，而是“当前主流工具怎么安全、可控地进入研发流程”。

## 1. 先讲结论

`Claude Code` 和 `Codex` 不该被比较成“谁更聪明”，更稳妥的比较方式是：

- 谁更适合当前工作面
- 谁的控制面更清楚
- 谁更容易接入团队规则、验证和 review
- 谁在权限、memory、工具接入和协作上更适配当前仓库

## 2. 官方锚点

### Claude Code

- Overview: [Claude Code overview](https://docs.anthropic.com/en/docs/claude-code/overview)
- Memory: [Claude Code memory](https://docs.anthropic.com/en/docs/claude-code/memory)
- Hooks: [Claude Code hooks](https://docs.anthropic.com/en/docs/claude-code/hooks)
- MCP: [Claude Code MCP](https://docs.anthropic.com/en/docs/claude-code/mcp)
- Subagents: [Claude Code sub-agents](https://docs.anthropic.com/en/docs/claude-code/sub-agents)
- Security: [Claude Code security](https://docs.anthropic.com/en/docs/claude-code/security)

### Codex

- Docs: [OpenAI Codex docs](https://developers.openai.com/codex)
- CLI: [OpenAI Codex CLI](https://developers.openai.com/codex/cli)
- Security: [OpenAI Codex security](https://developers.openai.com/codex/security)
- Internet access: [Codex cloud internet access](https://developers.openai.com/codex/cloud/internet-access)
- Prompting: [Codex prompting guide](https://developers.openai.com/cookbook/examples/gpt-5/codex_prompting_guide)
- AGENTS.md: [Codex AGENTS.md guide](https://developers.openai.com/codex/guides/agents-md)
- MCP: [OpenAI Docs MCP](https://platform.openai.com/docs/docs-mcp)

### ChatGPT

- 更适合做方案发散、问题拆解、原理解释和输出整理
- 更适合先把模糊需求压成结构化任务，再交给 coding agent 落地

## 3. 生产里先看什么

### 3.1 repo instructions

这是第一层护栏。无论叫 `AGENTS.md`、`CLAUDE.md` 还是别的，核心目标都一样：

- 任务边界清楚
- 禁止事项清楚
- 代码风格清楚
- 验证要求清楚
- 协作方式清楚

如果这层没写清，后面再好的模型也会在错误边界里高效乱跑。

### 3.2 memory

要区分三类：

- 会话级临时上下文
- 仓库级长期规则
- 用户或团队级偏好

更稳妥的原则不是“记得越多越好”，而是“长期记忆只留下稳定规则，任务细节不要污染长期 memory”。

### 3.3 tool boundary

生产环境必须先定义：

- 能读什么
- 能写什么
- 能执行什么命令
- 哪些操作必须审批
- 哪些外部系统只能只读

默认最小权限，比默认全放开稳得多。

### 3.4 task spec

更稳妥的任务写法至少要有：

- `Goal`
- `Context`
- `Constraints`
- `Done when`

如果任务只是“修一下这个”，agent 很容易把自己的猜测当成需求。

### 3.5 环境隔离

生产里最好把运行面分开看：

- 本地终端
- worktree / 独立工作区
- 非交互自动化
- 云端任务

隔离的意义不是“更复杂”，而是：

- 降低写错范围
- 降低上下文串扰
- 让 review 和回退更清楚

## 4. Claude Code 更该怎么用

### 4.1 它擅长什么

- 在本地仓库里读写和执行
- 配合 `CLAUDE.md`、hooks、subagents 做协作流程
- 把 agent 行为和仓库约束强绑定

### 4.2 生产关键点

- `CLAUDE.md` 分层要清楚，不要把临时任务写进长期规则
- hooks 适合做前置检查、审计和阻断
- subagents 适合拆独立任务，不适合抢同一写集
- MCP 接外部资源时要有权限边界和输出预算

### 4.3 最容易说错的地方

- 把 subagent 讲成“多开几个模型更快”
- 把 hooks 讲成“自动执行脚本”而忽略风险
- 把 memory 讲成“越多越聪明”

## 5. Codex 更该怎么用

### 5.1 它擅长什么

- 通过 `AGENTS.md` 规范仓库行为
- 以 agent loop 的方式推进分析、修改、验证
- 适合把工作流程压成明确任务和验证闭环

### 5.2 生产关键点

- `AGENTS.md` 只写稳定规则，不写临时操作指令
- 模型与 reasoning effort 选择要看任务复杂度，不是默认拉满
- 明确最小验证命令和失败后的降级动作
- 能自动做的，不等于应该自动放行

### 5.3 最容易说错的地方

- 把 Codex 讲成“自动程序员”
- 把模型选择讲成“越大越好”
- 把 AGENTS.md 写成一次性任务说明书

## 6. 共同最佳实践

### 6.1 指令分层

- 仓库长期规则放 `AGENTS.md` 或 `CLAUDE.md`
- 单次任务放 task spec
- 敏感边界放 approvals / policies
- 验证规则放 tests / CI gate

### 6.2 自动化半径

更稳妥的顺序：

1. 先读仓库、写小改动、跑最小验证
2. 再进入多文件改动
3. 再进入跨模块重构
4. 最后才考虑高风险自动化

### 6.3 不是 MCP，也不等于企业治理平台

- `Claude Code / Codex` 是 coding agent runtime
- `MCP` 是协议层
- 企业级平台治理还需要权限、组织规则、审计和交付系统配合

### 6.4 review gate

人工 review 仍然必要，因为很多错误不是“语法错”，而是：

- 任务理解错
- 范围发散
- 设计取舍错
- 风险判断错

### 6.5 CI gate

至少要有：

- 编译或构建
- 单测或最小回归
- 格式化或 lint
- 失败即阻断

### 6.6 ChatGPT 的最佳位置

更稳妥的用法不是让 ChatGPT 直接承担高风险仓库改动，而是让它承担：

- 方案比较
- 任务拆解
- 文档初稿
- 面试表达整理
- 复盘问题追问

当任务进入“读仓库 -> 改文件 -> 跑验证 -> 处理失败”这个闭环时，Claude Code 或 Codex 往往更合适。

## 7. Claude Code vs Codex

| 维度 | Claude Code | Codex |
| --- | --- | --- |
| 核心工作面 | 本地仓库协作、命令执行、上下文化开发 | agent loop、任务执行、仓库规则驱动 |
| 长期规则入口 | `CLAUDE.md` | `AGENTS.md` |
| 主要控制面 | hooks、subagents、权限与审批 | permission modes、agent loop、规则文件、云端与自动化 |
| 工具接入 | hooks、MCP、subagents | MCP、agent loop、仓库规则 |
| 强项 | 本地交互感、协作流程、任务拆分 | 任务闭环、规则化、结构化执行 |
| 常见误区 | 过度放权、memory 污染 | 把规则文件写成临时提示词 |

## 8. 当前更稳妥的团队接入顺序

1. 先把 repo instruction 文件写稳
2. 再定义最小验证命令
3. 再定义审批边界和禁止动作
4. 再补 hooks、MCP、subagents
5. 最后接 PR / CI 和更高自动化

## 9. 什么时候不该高自动化

- 任务边界不清
- 测试覆盖太弱
- 仓库规则还不稳定
- 需要高层架构判断
- 涉及数据迁移、安全或生产变更

一句话：

高风险、低验证、需求模糊的任务，宁可慢一点，也不要装成“AI 已经能完全接管”。

## 10. 生产 checklist

- 有稳定的 `AGENTS.md` 或 `CLAUDE.md`
- 有最小验证命令
- 有明确审批边界
- 有只读/读写/执行权限划分
- 有环境隔离策略
- 有 review gate
- 有 CI gate
- 有失败后的回退方式
- 有 memory 污染控制
- 有 MCP / 外部工具访问边界
- 有 rate limit / quota / cost 预算
- 有 canary / rollback 思路
- 有线上验证指标

## 11. 面试一句话

### Claude Code

它更像一个贴近本地仓库和协作流程的 agent 工作台，强项是把规则、命令、hooks、subagents 和 review 接到一起。

### Codex

它更像一个以仓库规则和任务闭环为核心的 coding agent 体系，重点不只是会改代码，而是能在规则、验证和人工放行下稳定推进任务。

### 总结

真正该比较的不是“哪个模型更神”，而是“哪个更适合当前仓库、当前风险、当前团队纪律”。
