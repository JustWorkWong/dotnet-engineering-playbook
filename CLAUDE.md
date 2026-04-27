# Project Instructions

## 项目定位

这是一个以 `.NET / C#` 生产实践为主、面试输出为辅的知识仓库。文档必须同时回答四件事：

1. 这个技术解决什么问题
2. 为什么这样设计
3. 有什么 trade-off 和失败模式
4. 在真实业务里怎么落地、观测、恢复

## 目录结构

```text
dotnet-engineering-playbook/
├─ README.md
├─ CLAUDE.md
└─ docs/
   ├─ 01-roadmap/
   │  └─ production-roadmap.md
   ├─ 02-foundations/
   │  ├─ dotnet-backend-foundation.md
   │  └─ ef-core-vs-dapper-vs-raw-sql.md
   ├─ 03-distributed-systems/
   │  ├─ dotnet-cap-production-guide.md
   │  ├─ openapi-vs-grpc-vs-message-boundaries.md
   │  ├─ cache-rate-limit-retry-circuit-breaker.md
   │  └─ modular-monolith-vs-microservices.md
   ├─ 04-observability/
   │  ├─ opentelemetry-maf-observability-guide.md
   │  ├─ multi-tenant-permission-audit.md
   │  └─ production-event-tracking.md
   ├─ 05-ai-coding/
   │  ├─ claude-code-codex-chatgpt-playbook.md
   │  └─ rag-workflow-agent-layering.md
   └─ 06-interview/
      └─ interview-positioning.md
```

## 文件职责

- `README.md`：仓库入口，只讲定位、阅读顺序、目录导航。
- `docs/01-roadmap/production-roadmap.md`：总路线图，决定主线，不承载细节百科。
- `docs/02-foundations/`：`.NET` 运行时、异步、GC、线程池、请求管线，以及数据访问默认解与热点下沉策略。
- `docs/03-distributed-systems/`：消息、最终一致性、工作流、分布式可靠性、通信边界，以及韧性治理判断。
- `docs/04-observability/`：Trace、Metric、Log、埋点、告警、审计，以及租户/权限治理相关的可追踪信号。
- `docs/05-ai-coding/`：`Claude Code / Codex / ChatGPT` 的工程协作模式，以及 AI 业务系统的分层落地方法。
- `docs/06-interview/`：把前面的生产实践压缩成可表达、可训练、可追问的输出层。

## 依赖关系

```text
roadmap
  -> foundations
  -> distributed-systems
  -> observability
  -> ai-coding
  -> interview
```

约束：

- 新增专题前，先判断它属于哪个一级目录，不要平行再造新分类。
- 面试文档不能脱离生产实践单独生长，必须回链到上游专题。
- AI 协作文档只写稳定方法，不写一次性任务记录。

## 文档写作约定

- 先写问题，再写方案，不要直接从工具名开始。
- 每篇专题至少包含：核心问题、最小实现、生产坑、trade-off、何时别用、验证方式。
- 优先保留能复用的判断框架，减少“版本快照式”结论。
- 同一主题超过 3 个分支时，优先改写结构，避免堆例外。

## 变更规则

- 如果新增、删除、移动一级或二级目录，必须同步更新本文件。
- 如果新增专题改变阅读主线，必须同步更新 `README.md` 和 `production-roadmap.md`。
