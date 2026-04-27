# dotnet-engineering-playbook

面向 `.NET / C#` 工程师的生产实践手册。这个仓库不把目标收窄成“面试八股”，而是把 **底层原理、业务开发 best practices、架构 trade-off、技术演进、AI 协作方式** 放到同一张地图里。

## 这个仓库解决什么问题

- 面试资料很多，但和真实业务开发脱节。
- 知道 API 怎么用，不等于知道为什么这样设计。
- 会写 CRUD，不等于能解释并发、GC、可观测性、分布式一致性和故障恢复。
- 会用 `Claude Code / Codex / ChatGPT`，不等于能把它们安全地接入团队研发流程。

一句话定位：

> 让 `.NET` 学习从“会写”升级到“会解释、会取舍、会落地、会借助 AI 放大生产力”。

## 推荐阅读顺序

1. [生产路线图](./docs/01-roadmap/production-roadmap.md)
2. [`.NET` 后端底座](./docs/02-foundations/dotnet-backend-foundation.md)
3. [EF Core vs Dapper vs Raw SQL](./docs/02-foundations/ef-core-vs-dapper-vs-raw-sql.md)
4. [CAP 生产实践](./docs/03-distributed-systems/dotnet-cap-production-guide.md)
5. [OpenAPI vs gRPC vs Message](./docs/03-distributed-systems/openapi-vs-grpc-vs-message-boundaries.md)
6. [模块化单体 vs 微服务](./docs/03-distributed-systems/modular-monolith-vs-microservices.md)
7. [缓存 / 限流 / 重试 / 熔断](./docs/03-distributed-systems/cache-rate-limit-retry-circuit-breaker.md)
8. [OpenTelemetry + MAF 可观测性](./docs/04-observability/opentelemetry-maf-observability-guide.md)
9. [生产级埋点设计](./docs/04-observability/production-event-tracking.md)
10. [多租户 / 权限 / 审计 生产设计](./docs/04-observability/multi-tenant-permission-audit.md)
11. [RAG / Workflow / Agent 分层实践](./docs/05-ai-coding/rag-workflow-agent-layering.md)
12. [Claude Code / Codex / ChatGPT 协作手册](./docs/05-ai-coding/claude-code-codex-chatgpt-playbook.md)
13. [面试定位与训练方法](./docs/06-interview/interview-positioning.md)

## 当前目录结构

```text
dotnet-engineering-playbook/
├─ README.md
├─ CLAUDE.md
└─ docs/
   ├─ 01-roadmap/
   ├─ 02-foundations/
   ├─ 03-distributed-systems/
   ├─ 04-observability/
   ├─ 05-ai-coding/
   └─ 06-interview/
```

## 仓库设计原则

- 简化优先：先给默认解，再讲什么时候升级，不堆一堆特殊分支。
- 实用主义：优先写能运行、能解释、能观测、能恢复的方案。
- 面试从属于生产：面试回答必须能回到真实系统问题，而不是背术语。
- 技术选型要写清代价：不只写“为什么选它”，也写“为什么此时不选别的”。
- AI 是放大器，不是代替判断：任务边界、验证、回滚、审批必须先于自动化。

## 当前收录内容

- 从 `career-knowledge-map` 迁入：
  - `.NET` 后端底座
  - `Claude Code / Codex` 生产实践手册
- 从 `面试` 目录迁入：
  - `DotNetCore.CAP` 完整学习与生产实践
  - `OpenTelemetry + MAF` 可观测性完整指南
  - 数据埋点生产级方案
- 当前新增：
  - `EF Core vs Dapper vs Raw SQL` 决策文档
  - `模块化单体 vs 微服务` 演化文档
  - `RAG / Workflow / Agent` 分层实践文档
  - `OpenAPI vs gRPC vs Message` 边界文档
  - `缓存 / 限流 / 重试 / 熔断` 韧性治理文档
  - `多租户 / 权限 / 审计` 生产设计文档

## 下一步建议

建议后续继续补三类内容：

1. `数据库迁移 / 回滚 / 灰度发布` 的生产文档
2. `任务调度 / 后台作业 / 工作流状态机` 的设计文档
3. `AI 成本 / 评测 / 安全护栏` 的工程治理文档
