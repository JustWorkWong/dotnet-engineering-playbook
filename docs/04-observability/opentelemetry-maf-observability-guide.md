# OpenTelemetry + MAF 可观测性完整指南

> 整理时间：2026-04-26  
> 适用场景：C# / .NET 后端、Microsoft Agent Framework（MAF）、数据库优化 AI 项目、Redis、MQ、EF Core、Workflow、Agent、MCP、LLM 调用链路观测。  
> 文档目标：把本次会话中关于 OpenTelemetry 的概念、MAF 原生支持、Trace / Metric / Log 的选择规则、生产级落地方式、常见误区和代码示例整理成一份完整 Markdown 文档。

---

## 目录

1. [OpenTelemetry 是什么](#1-opentelemetry-是什么)
2. [它解决什么问题](#2-它解决什么问题)
3. [OpenTelemetry 和自己打印 Log 的区别](#3-opentelemetry-和自己打印-log-的区别)
4. [OpenTelemetry 的核心信号 Signals](#4-opentelemetry-的核心信号-signals)
5. [Trace、Metric、Log、Event、Baggage 怎么选](#5-tracemetriclogeventbaggage-怎么选)
6. [MAF 是否有原生 OpenTelemetry](#6-maf-是否有原生-opentelemetry)
7. [MAF 原生 OTel 和手动埋点的边界](#7-maf-原生-otel-和手动埋点的边界)
8. [.NET + MAF OpenTelemetry 对接示例](#8-net--maf-opentelemetry-对接示例)
9. [对你最有价值的落地点](#9-对你最有价值的落地点)
10. [生产级必须补充的关键点](#10-生产级必须补充的关键点)
11. [Collector：生产环境的观测数据网关](#11-collector生产环境的观测数据网关)
12. [Sampling：采样策略](#12-sampling采样策略)
13. [Metric 标签基数 Cardinality](#13-metric-标签基数-cardinality)
14. [Context Propagation：链路能否串起来的关键](#14-context-propagation链路能否串起来的关键)
15. [Log Correlation：日志和 Trace 的关联](#15-log-correlation日志和-trace-的关联)
16. [Semantic Conventions：字段规范](#16-semantic-conventions字段规范)
17. [Resource：服务身份信息](#17-resource服务身份信息)
18. [安全脱敏策略](#18-安全脱敏策略)
19. [OTel 不能替代业务数据库表](#19-otel-不能替代业务数据库表)
20. [告警、SLI、SLO 设计](#20-告警slislo-设计)
21. [环境策略：Dev / Test / Prod](#21-环境策略dev--test--prod)
22. [如何验证接入成功](#22-如何验证接入成功)
23. [推荐的观测规范](#23-推荐的观测规范)
24. [面试级回答模板](#24-面试级回答模板)
25. [最终总结](#25-最终总结)
26. [参考资料](#26-参考资料)

---

# 1. OpenTelemetry 是什么

OpenTelemetry，简称 **OTel**，可以理解为：

> 一套统一的可观测性数据采集标准、SDK、API、语义规范和 Collector 管道。

它负责把应用运行过程中的遥测数据统一采集出来，例如：

- HTTP 请求耗时
- SQL 执行耗时
- Redis 调用耗时
- MQ 发布 / 消费链路
- LLM 调用耗时
- Agent 执行耗时
- Tool / MCP 调用耗时
- GC、线程池、CPU、内存
- 异常日志
- 慢 SQL
- Token 消耗
- Workflow 执行链路

然后再把这些数据导出到后端系统中：

- Prometheus / VictoriaMetrics / Mimir：主要存 Metrics
- Jaeger / Tempo：主要看 Traces
- Loki / Elastic / ClickHouse：主要存 Logs
- Grafana：主要展示 Dashboard 和告警
- SkyWalking / Datadog / New Relic / Elastic APM：完整 APM 平台

需要特别明确：

```text
OpenTelemetry 不是监控平台。
OpenTelemetry 不是日志库。
OpenTelemetry 不是数据库。
OpenTelemetry 不是可视化界面。

OpenTelemetry 是统一采集、统一上下文、统一语义、统一导出的标准层。
```

---

# 2. 它解决什么问题

假设你做的是一个数据库优化 AI 系统：

```text
Vue 前端
  ↓
.NET API
  ↓
MAF Workflow
  ↓
SqlPlannerAgent
  ↓
MCP Tool：获取表结构、索引、执行计划
  ↓
SqlAnalyzeAgent
  ↓
PostgreSQL / MySQL
  ↓
Redis Cache
  ↓
MQ
  ↓
人工审核
  ↓
优化结果入库
```

线上出问题时，你会问：

```text
这次请求为什么慢？
慢在 API 层，还是 SQL，还是 Redis，还是 LLM？
Agent 调用了哪些工具？
哪个 Tool 失败了？
MCP Server 是否超时？
MQ 消息是否发出去了？
消费者是否处理成功？
人工审核等待了多久？
GC 是否频繁？
线程池是否堵塞？
日志和具体请求是否能对应上？
```

如果只靠普通日志，你通常只能看到一些分散的记录：

```text
[10:00:01] 收到 SQL 优化请求
[10:00:02] 开始调用 Agent
[10:00:05] 调用 explain_sql tool
[10:00:08] SQL 优化失败
```

但是你很难快速回答：

```text
这些日志是不是同一次请求？
对应哪个 workflowRunId？
对应哪个 agentRunId？
失败前调用过哪些下游服务？
整体耗时是多少？
P95 是否异常？
是否只是个例，还是整体趋势变坏？
```

OpenTelemetry 的核心价值就是：

```text
把一次请求经过的 API、DB、Redis、MQ、Agent、Tool、LLM、Workflow 全部串起来。
同时还能把日志、指标、链路追踪关联起来。
```

---

# 3. OpenTelemetry 和自己打印 Log 的区别

## 3.1 一句话区别

```text
自己打印 Log：记录某个点发生了什么。
OpenTelemetry：把 Log、Trace、Metric 统一采集、关联、导出，形成完整观测体系。
```

## 3.2 普通 Log 的价值

普通日志仍然非常重要，它适合记录：

- 异常堆栈
- 业务失败原因
- 审核意见
- 外部服务返回错误
- 权限拒绝原因
- 关键业务状态变化

例如：

```csharp
_logger.LogError(
    ex,
    "SQL 分析 Agent 执行失败，WorkflowRunId={WorkflowRunId}, AgentName={AgentName}, SqlHash={SqlHash}",
    command.WorkflowRunId,
    "SqlAnalyzeAgent",
    command.SqlHash);
```

## 3.3 普通 Log 的局限

普通日志天然有几个问题：

### 问题一：日志是点，不是链路

你看到一条日志：

```text
SQL 优化失败
```

但你不知道：

```text
它属于哪次 HTTP 请求？
之前调用过哪些 Tool？
是否先查过 Redis？
是否执行过慢 SQL？
LLM 调用了几次？
```

Trace 可以把它串成：

```text
HTTP POST /api/sql-optimization
 ├─ LoadProjectConfig
 ├─ QueryTableSchema
 ├─ RedisCacheGet
 ├─ MAF Workflow
 │   ├─ SqlPlannerAgent
 │   ├─ Tool: get_table_schema
 │   ├─ Tool: explain_sql
 │   └─ SqlAnalyzeAgent
 └─ SaveResult
```

### 问题二：日志不好做趋势和告警

你可以打印：

```text
检测到慢 SQL
```

但你不好高效回答：

```text
过去 5 分钟慢 SQL 有多少条？
Agent P95 耗时是多少？
Tool 失败率是多少？
MQ 消费失败率是多少？
今天 Token 消耗是多少？
```

这些应该用 Metric。

### 问题三：日志字段容易不统一

不同人可能写：

```text
userId
user_id
uid
UserID
```

OpenTelemetry 通过 Semantic Conventions 和团队自定义字段规范，统一字段命名。

### 问题四：日志和 Trace 默认不一定能关联

理想情况下，每条日志都应该带：

```text
trace_id
span_id
workflow_run_id
agent_run_id
project_id
```

这样看到日志时，可以跳转到对应 Trace。

---

# 4. OpenTelemetry 的核心信号 Signals

OpenTelemetry 中最重要的几个信号：

| Signal | 中文 | 解决的问题 |
|---|---|---|
| Trace | 链路追踪 | 一次请求经过了哪些步骤，慢在哪里 |
| Span | 链路中的一步 | 一次 DB 查询、一次 Redis、一次 Agent、一次 Tool |
| Metric | 指标 | 统计趋势、P95、失败率、告警 |
| Log | 日志 | 具体事件、异常堆栈、业务原因 |
| Event | Span 内事件 | 当前链路中的关键瞬间 |
| Attribute / Tag | 属性 | 查询和过滤维度 |
| Resource | 资源 | 这条数据来自哪个服务、环境、实例 |
| Baggage | 跨服务上下文 | 把 tenantId、projectId 等传给下游 |
| Link | 链路关联 | 异步、批处理、fan-in/fan-out 的关联 |

---

# 5. Trace、Metric、Log、Event、Baggage 怎么选

## 5.1 最实用口诀

```text
单次请求排障，看 Trace。
整体趋势告警，看 Metric。
具体错误原因，看 Log。
链路中的关键瞬间，用 Event。
跨服务传递上下文，用 Baggage。
描述服务是谁，用 Resource。
查询过滤维度，用 Attribute。
```

再简化：

```text
Trace：一次请求。
Metric：一段时间。
Log：一件事情的细节。
Event：一次请求中的关键点。
Baggage：跨服务带上下文。
```

## 5.2 Trace 什么时候用

Trace 用来回答：

```text
这一次请求到底发生了什么？
慢在哪里？
失败在哪里？
经过了哪些服务和组件？
```

适合建 Span 的场景：

- HTTP 请求
- 外部 HTTP 调用
- gRPC 调用
- DB 查询
- Redis 调用
- MQ 发布
- MQ 消费
- Workflow Run
- Workflow Node
- Agent Run
- Tool Call
- MCP Call
- LLM Call
- Checkpoint Save / Restore
- 人工审核恢复

代码示例：

```csharp
using System.Diagnostics;

public static class AppTelemetry
{
    public static readonly ActivitySource Source =
        new("DbOptimizer.App");
}

public async Task RunSqlOptimizationAsync(
    SqlOptimizationCommand command,
    CancellationToken cancellationToken)
{
    using var activity = AppTelemetry.Source.StartActivity(
        "sql_optimization.run",
        ActivityKind.Internal);

    activity?.SetTag("workflow.run_id", command.WorkflowRunId);
    activity?.SetTag("project.id", command.ProjectId);
    activity?.SetTag("db.system", command.DbSystem);
    activity?.SetTag("sql.hash", command.SqlHash);

    await _workflowRunner.RunAsync(command, cancellationToken);
}
```

## 5.3 Metric 什么时候用

Metric 用来回答：

```text
整体情况怎么样？
过去 5 分钟失败率是多少？
P95 耗时是多少？
慢 SQL 每分钟多少条？
MQ Lag 是否增长？
Agent Token 消耗是否异常？
```

适合 Metric 的内容：

- 请求数
- 失败数
- 耗时分布
- P95 / P99
- 队列长度
- 连接池使用率
- GC 次数
- 线程池队列长度
- Token 消耗
- Tool 调用失败率
- 慢 SQL 数量

代码示例：

```csharp
using System.Diagnostics.Metrics;

public static class BusinessMetrics
{
    public static readonly Meter Meter = new("DbOptimizer.Business");

    public static readonly Counter<long> SlowSqlDetected =
        Meter.CreateCounter<long>(
            "dboptimizer.slow_sql.detected",
            unit: "{count}");

    public static readonly Histogram<double> AgentRunDuration =
        Meter.CreateHistogram<double>(
            "dboptimizer.agent.run.duration",
            unit: "s");

    public static readonly Counter<long> AgentRunFailed =
        Meter.CreateCounter<long>(
            "dboptimizer.agent.run.failed",
            unit: "{count}");
}
```

记录慢 SQL：

```csharp
BusinessMetrics.SlowSqlDetected.Add(
    1,
    new KeyValuePair<string, object?>("db.system", "postgresql"),
    new KeyValuePair<string, object?>("source", "auto_capture"));
```

## 5.4 Metric 类型怎么选

| Metric 类型 | 适合什么 | 例子 |
|---|---|---|
| Counter | 只增不减的次数 | 请求数、失败数、慢 SQL 数 |
| UpDownCounter | 可增可减的数量 | 当前运行中的 Workflow 数 |
| Histogram | 分布、耗时、大小 | Agent 耗时、SQL 耗时、审核等待时间 |
| ObservableGauge | 定时观测当前值 | MQ Lag、连接池使用数、队列长度 |

你的项目建议：

```text
Counter：
- agent.run.total
- agent.run.failed
- tool.call.total
- tool.call.failed
- slow_sql.detected

Histogram：
- agent.run.duration
- tool.call.duration
- sql.execution.duration
- review.wait.duration
- workflow.run.duration

ObservableGauge：
- mq.consumer.lag
- db.connection_pool.active
- workflow.pending.count
- review.pending.count

UpDownCounter：
- workflow.running.count
- agent.running.count
```

## 5.5 Log 什么时候用

Log 用来回答：

```text
具体为什么失败？
异常堆栈是什么？
外部系统返回了什么错误？
人工审核意见是什么？
```

适合 Log 的内容：

- 异常堆栈
- 业务拒绝原因
- 审核意见
- 外部系统错误码
- 参数摘要
- 安全拦截原因
- Workflow 恢复失败原因

示例：

```csharp
try
{
    await _agent.RunAsync(input, cancellationToken);
}
catch (Exception ex)
{
    _logger.LogError(
        ex,
        "SQL 分析 Agent 执行失败，WorkflowRunId={WorkflowRunId}, AgentName={AgentName}, SqlHash={SqlHash}",
        command.WorkflowRunId,
        "SqlAnalyzeAgent",
        command.SqlHash);

    throw;
}
```

## 5.6 Event 什么时候用

Event 是 Span 内部的关键时间点。

选择规则：

```text
有开始、有结束、有耗时：Span。
只是某个瞬间发生的事：Event。
```

适合 Event：

- 检测到慢 SQL
- 触发人工审核
- 命中缓存
- 发生降级
- 触发 Compaction
- Workflow 进入等待状态

示例：

```csharp
Activity.Current?.AddEvent(new ActivityEvent(
    "slow_sql.detected",
    tags: new ActivityTagsCollection
    {
        ["db.system"] = "postgresql",
        ["sql.hash"] = sqlHash,
        ["duration.ms"] = durationMs
    }));
```

## 5.7 Baggage 什么时候用

Baggage 用来跨服务传递少量上下文。

适合放：

- tenant.id
- project.id
- workflow.run_id
- request.source

不适合放：

- 完整 SQL
- 用户手机号
- 身份证
- API Key
- 大段 Prompt
- 大段 JSON
- 高敏感数据

---

# 6. MAF 是否有原生 OpenTelemetry

结论：

```text
MAF / Microsoft Agent Framework 有原生 OpenTelemetry 支持。
但是不是完全不用配置，也不是能覆盖整个业务系统。
```

MAF 原生 OpenTelemetry 主要覆盖：

- Agent 执行链路
- LLM 调用
- Tool / Function 调用
- Token 使用
- Agent / Tool 执行耗时
- Agent 错误

常见自动 Span：

| 自动 Span | 说明 |
|---|---|
| invoke_agent `<agent_name>` | 每次 Agent 调用的顶层 span |
| chat `<model_name>` | Agent 内部调用模型的 span |
| execute_tool `<function_name>` | Agent 调用工具函数的 span |

常见自动 Metrics：

| Metric | 说明 |
|---|---|
| gen_ai.client.operation.duration | LLM 操作耗时 |
| gen_ai.client.token.usage | Token 使用量 |
| agent_framework.function.invocation.duration | Function Tool 执行耗时 |

---

# 7. MAF 原生 OTel 和手动埋点的边界

## 7.1 MAF 原生负责什么

```text
Agent Run
LLM Call
Tool Call
Token Usage
Tool Duration
Agent Error
```

通过：

```csharp
.UseOpenTelemetry(...)
.WithOpenTelemetry(...)
```

## 7.2 你仍然需要手动处理什么

```text
OTel Exporter 配置
ASP.NET Core / EF Core / Redis / MQ / Runtime 监控
workflowRunId / projectId / userId 等业务标签
慢 SQL 业务归因
人工审核等待时长
MQ 消费积压、重试、死信
业务告警指标
Checkpoint Save / Restore 业务事件
Workflow 恢复失败
权限 / 审计 / 安全拦截
```

准确理解：

```text
MAF 原生 OTel 解决 Agent / LLM / Tool 层的自动观测。
生产级系统还需要补 API、DB、Redis、MQ、Runtime、Workflow、审核、业务状态等观测。
```

---

# 8. .NET + MAF OpenTelemetry 对接示例

## 8.1 Program.cs 基础配置

```csharp
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;
using OpenTelemetry.Logs;

var builder = WebApplication.CreateBuilder(args);

const string ServiceName = "db-optimizer-api";
const string ServiceVersion = "1.0.0";

const string MafSourceName = "DbOptimizer.Maf";
const string AppSourceName = "DbOptimizer.App";
const string BusinessMeterName = "DbOptimizer.Business";

builder.Services.AddControllers();

builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource =>
    {
        resource
            .AddService(
                serviceName: ServiceName,
                serviceVersion: ServiceVersion,
                serviceInstanceId: Environment.MachineName)
            .AddAttributes(new Dictionary<string, object>
            {
                ["deployment.environment"] = builder.Environment.EnvironmentName,
                ["app.module"] = "db-optimizer"
            });
    })
    .WithTracing(tracing =>
    {
        tracing
            .AddAspNetCoreInstrumentation(options =>
            {
                options.RecordException = true;
            })
            .AddHttpClientInstrumentation(options =>
            {
                options.RecordException = true;
            })
            .AddEntityFrameworkCoreInstrumentation(options =>
            {
                // 生产环境谨慎开启，避免泄露 SQL 和参数
                options.SetDbStatementForText = builder.Environment.IsDevelopment();
            })
            .AddSource(MafSourceName)
            .AddSource(AppSourceName)
            .AddOtlpExporter(options =>
            {
                options.Endpoint = new Uri(
                    builder.Configuration["OTEL_EXPORTER_OTLP_ENDPOINT"]
                    ?? "http://localhost:4317");
            });
    })
    .WithMetrics(metrics =>
    {
        metrics
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddRuntimeInstrumentation()
            .AddProcessInstrumentation()
            .AddMeter(BusinessMeterName)
            .AddMeter(MafSourceName)
            .AddOtlpExporter(options =>
            {
                options.Endpoint = new Uri(
                    builder.Configuration["OTEL_EXPORTER_OTLP_ENDPOINT"]
                    ?? "http://localhost:4317");
            });
    });

builder.Logging.AddOpenTelemetry(logging =>
{
    logging.IncludeFormattedMessage = true;
    logging.IncludeScopes = true;

    logging.AddOtlpExporter(options =>
    {
        options.Endpoint = new Uri(
            builder.Configuration["OTEL_EXPORTER_OTLP_ENDPOINT"]
            ?? "http://localhost:4317");
    });
});

var app = builder.Build();

app.MapControllers();

app.Run();
```

## 8.2 MAF ChatClient 启用 OpenTelemetry

```csharp
var instrumentedChatClient = rawChatClient
    .AsBuilder()
    .UseOpenTelemetry(
        sourceName: "DbOptimizer.Maf",
        configure: cfg =>
        {
            // 生产环境建议 false
            cfg.EnableSensitiveData = false;
        })
    .Build();
```

## 8.3 MAF Agent 启用 OpenTelemetry

```csharp
var agent = new ChatClientAgent(
    instrumentedChatClient,
    name: "SqlAnalyzeAgent",
    instructions: "你是 SQL 优化分析 Agent。",
    tools:
    [
        AIFunctionFactory.Create(GetTableSchemaAsync),
        AIFunctionFactory.Create(GetSlowSqlAsync),
        AIFunctionFactory.Create(ExplainSqlAsync)
    ])
    .WithOpenTelemetry(
        sourceName: "DbOptimizer.Maf",
        configure: cfg =>
        {
            cfg.EnableSensitiveData = false;
        });
```

## 8.4 生产环境不要开启 SensitiveData

不建议生产开启：

```csharp
cfg.EnableSensitiveData = true;
```

因为可能记录：

- Prompt
- Response
- Function 参数
- Function 返回结果
- Tool 输入输出
- SQL 内容
- 业务字段

建议：

```csharp
cfg.EnableSensitiveData = false;
```

或者仅本地开发：

```csharp
cfg.EnableSensitiveData = builder.Environment.IsDevelopment();
```

---

# 9. 对你最有价值的落地点

结合你的 C# + MAF + 数据库优化 AI 项目，OpenTelemetry 最有价值的落地点如下。

## 9.1 应用层监控

需要观测：

```text
HTTP 请求耗时
接口错误率
P95 / P99
异常堆栈
traceId
userId / userHash
projectId
workflowRunId
```

## 9.2 数据库监控

需要观测：

```text
EF Core SQL 执行耗时
慢 SQL
数据库连接池耗尽
事务耗时
死锁异常
查询超时
```

## 9.3 Redis 监控

需要观测：

```text
Redis 命中率
Redis 调用耗时
Redis 超时
缓存穿透 / 击穿相关计数
```

## 9.4 MQ 监控

需要观测：

```text
消息发布耗时
消息消费耗时
消费失败次数
重试次数
死信数量
消费者 lag
同一个业务 key 的顺序处理耗时
```

## 9.5 MAF / Agent 监控

需要观测：

```text
WorkflowRun 耗时
AgentRun 耗时
ToolCall 耗时
LLM Token 消耗
模型错误率
MCP 调用失败率
人工审核等待时长
Compaction 触发次数
Session 恢复失败次数
Checkpoint Save / Restore 耗时
```

## 9.6 Runtime 监控

需要观测：

```text
GC 次数
GC 暂停
线程池队列长度
线程池可用线程
CPU
内存
进程句柄
异常数量
```

---

## 9.7 业务 Span 示例：WorkflowRun

```csharp
using System.Diagnostics;

public static class AppTelemetry
{
    public const string AppSourceName = "DbOptimizer.App";

    public static readonly ActivitySource ActivitySource =
        new(AppSourceName);
}

public sealed class SqlOptimizationAppService
{
    private readonly AIAgent _sqlAnalyzeAgent;
    private readonly ILogger<SqlOptimizationAppService> _logger;

    public SqlOptimizationAppService(
        AIAgent sqlAnalyzeAgent,
        ILogger<SqlOptimizationAppService> logger)
    {
        _sqlAnalyzeAgent = sqlAnalyzeAgent;
        _logger = logger;
    }

    public async Task<SqlOptimizationResult> RunAsync(
        SqlOptimizationCommand command,
        CancellationToken cancellationToken)
    {
        using var activity = AppTelemetry.ActivitySource.StartActivity(
            "sql_optimization.run",
            ActivityKind.Internal);

        activity?.SetTag("project.id", command.ProjectId);
        activity?.SetTag("workflow.run_id", command.WorkflowRunId);
        activity?.SetTag("db.instance_id", command.DbInstanceId);
        activity?.SetTag("db.system", command.DbSystem);
        activity?.SetTag("sql.hash", command.SqlHash);
        activity?.SetTag("optimization.source", command.Source);

        using var scope = _logger.BeginScope(new Dictionary<string, object?>
        {
            ["ProjectId"] = command.ProjectId,
            ["WorkflowRunId"] = command.WorkflowRunId,
            ["DbInstanceId"] = command.DbInstanceId,
            ["SqlHash"] = command.SqlHash
        });

        try
        {
            _logger.LogInformation("开始 SQL 优化分析");

            var agentInput = BuildAgentInput(command);

            var agentResult = await _sqlAnalyzeAgent.RunAsync(
                agentInput,
                cancellationToken: cancellationToken);

            activity?.SetTag("agent.result.status", "completed");
            activity?.SetStatus(ActivityStatusCode.Ok);

            return new SqlOptimizationResult
            {
                WorkflowRunId = command.WorkflowRunId,
                Summary = agentResult.Text
            };
        }
        catch (Exception ex)
        {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            activity?.AddException(ex);

            _logger.LogError(ex, "SQL 优化分析失败");
            throw;
        }
    }

    private static string BuildAgentInput(SqlOptimizationCommand command)
    {
        return $"""
        请分析以下 SQL：

        DB: {command.DbSystem}
        SQL Hash: {command.SqlHash}
        """;
    }
}
```

## 9.8 业务 Metric 示例

```csharp
using System.Diagnostics.Metrics;

public static class BusinessMetrics
{
    public const string MeterName = "DbOptimizer.Business";

    public static readonly Meter Meter = new(MeterName);

    public static readonly Counter<long> SlowSqlDetected =
        Meter.CreateCounter<long>(
            name: "dboptimizer.slow_sql.detected",
            unit: "{count}",
            description: "检测到的慢 SQL 数量");

    public static readonly Counter<long> AgentRunFailed =
        Meter.CreateCounter<long>(
            name: "dboptimizer.agent.run.failed",
            unit: "{count}",
            description: "Agent 执行失败次数");

    public static readonly Histogram<double> AgentRunDuration =
        Meter.CreateHistogram<double>(
            name: "dboptimizer.agent.run.duration",
            unit: "s",
            description: "Agent 执行耗时");

    public static readonly Histogram<double> ReviewWaitDuration =
        Meter.CreateHistogram<double>(
            name: "dboptimizer.review.wait.duration",
            unit: "s",
            description: "人工审核等待耗时");

    public static readonly Counter<long> MqConsumeFailed =
        Meter.CreateCounter<long>(
            name: "dboptimizer.mq.consume.failed",
            unit: "{count}",
            description: "MQ 消费失败次数");
}
```

记录慢 SQL：

```csharp
BusinessMetrics.SlowSqlDetected.Add(
    1,
    new KeyValuePair<string, object?>("db.system", "postgresql"),
    new KeyValuePair<string, object?>("db.instance_id", dbInstanceId),
    new KeyValuePair<string, object?>("slow_sql.source", "auto_capture"));
```

记录审核等待时间：

```csharp
var waitSeconds = (review.CompletedAt - review.CreatedAt).TotalSeconds;

BusinessMetrics.ReviewWaitDuration.Record(
    waitSeconds,
    new KeyValuePair<string, object?>("review.type", "sql_optimization"),
    new KeyValuePair<string, object?>("review.result", review.Decision.ToString()));
```

---

## 9.9 慢 SQL 对接示例：EF Core Interceptor

EF Core 自动 instrumentation 可以采集 SQL 执行 span，但“超过 500ms 算慢 SQL，并打业务指标”这种逻辑要自己补。

```csharp
using System.Data.Common;
using Microsoft.EntityFrameworkCore.Diagnostics;
using System.Diagnostics;

public sealed class SlowSqlTelemetryInterceptor : DbCommandInterceptor
{
    private readonly ILogger<SlowSqlTelemetryInterceptor> _logger;

    private static readonly TimeSpan SlowThreshold =
        TimeSpan.FromMilliseconds(500);

    public SlowSqlTelemetryInterceptor(
        ILogger<SlowSqlTelemetryInterceptor> logger)
    {
        _logger = logger;
    }

    public override DbDataReader ReaderExecuted(
        DbCommand command,
        CommandExecutedEventData eventData,
        DbDataReader result)
    {
        RecordIfSlow(command, eventData);
        return result;
    }

    public override object? ScalarExecuted(
        DbCommand command,
        CommandExecutedEventData eventData,
        object? result)
    {
        RecordIfSlow(command, eventData);
        return result;
    }

    public override int NonQueryExecuted(
        DbCommand command,
        CommandExecutedEventData eventData,
        int result)
    {
        RecordIfSlow(command, eventData);
        return result;
    }

    private void RecordIfSlow(
        DbCommand command,
        CommandExecutedEventData eventData)
    {
        if (eventData.Duration < SlowThreshold)
        {
            return;
        }

        var dbSystem = DetectDbSystem(command);
        var sqlHash = SqlHashHelper.Hash(command.CommandText);

        Activity.Current?.AddEvent(new ActivityEvent(
            name: "db.slow_sql.detected",
            tags: new ActivityTagsCollection
            {
                ["db.system"] = dbSystem,
                ["db.duration_ms"] = eventData.Duration.TotalMilliseconds,
                ["sql.hash"] = sqlHash
            }));

        Activity.Current?.SetTag("db.slow_sql", true);
        Activity.Current?.SetTag("db.slow_sql.duration_ms", eventData.Duration.TotalMilliseconds);
        Activity.Current?.SetTag("db.slow_sql.hash", sqlHash);

        BusinessMetrics.SlowSqlDetected.Add(
            1,
            new KeyValuePair<string, object?>("db.system", dbSystem));

        _logger.LogWarning(
            "检测到慢 SQL，DbSystem={DbSystem}, DurationMs={DurationMs}, SqlHash={SqlHash}",
            dbSystem,
            eventData.Duration.TotalMilliseconds,
            sqlHash);
    }

    private static string DetectDbSystem(DbCommand command)
    {
        var typeName = command.GetType().FullName ?? "";

        if (typeName.Contains("Npgsql", StringComparison.OrdinalIgnoreCase))
        {
            return "postgresql";
        }

        if (typeName.Contains("MySql", StringComparison.OrdinalIgnoreCase))
        {
            return "mysql";
        }

        return "unknown";
    }
}
```

注册：

```csharp
builder.Services.AddScoped<SlowSqlTelemetryInterceptor>();

builder.Services.AddDbContext<AppDbContext>((sp, options) =>
{
    options.UseNpgsql(builder.Configuration.GetConnectionString("Postgres"));

    options.AddInterceptors(
        sp.GetRequiredService<SlowSqlTelemetryInterceptor>());
});
```

## 9.10 Redis 对接示例

```csharp
public sealed class SqlAnalysisCache
{
    private readonly IDatabase _redis;
    private readonly ILogger<SqlAnalysisCache> _logger;

    public SqlAnalysisCache(
        IConnectionMultiplexer connectionMultiplexer,
        ILogger<SqlAnalysisCache> logger)
    {
        _redis = connectionMultiplexer.GetDatabase();
        _logger = logger;
    }

    public async Task<string?> GetAsync(
        string sqlHash,
        CancellationToken cancellationToken)
    {
        using var activity = AppTelemetry.ActivitySource.StartActivity(
            "cache.sql_analysis.get",
            ActivityKind.Client);

        activity?.SetTag("cache.system", "redis");
        activity?.SetTag("cache.type", "sql_analysis_result");
        activity?.SetTag("sql.hash", sqlHash);

        var key = $"sql-analysis:{sqlHash}";
        var value = await _redis.StringGetAsync(key);

        var hit = value.HasValue;

        activity?.SetTag("cache.hit", hit);

        _logger.LogInformation(
            "读取 SQL 分析缓存，SqlHash={SqlHash}, Hit={Hit}",
            sqlHash,
            hit);

        return hit ? value.ToString() : null;
    }
}
```

## 9.11 MQ 发布和消费链路打通

MQ 最关键的是 Trace Context Propagation。

发布消息时注入上下文：

```csharp
using OpenTelemetry;
using OpenTelemetry.Context.Propagation;
using System.Diagnostics;

public sealed class SqlOptimizationMessagePublisher
{
    private static readonly TextMapPropagator Propagator =
        Propagators.DefaultTextMapPropagator;

    private readonly IMessageBus _bus;

    public SqlOptimizationMessagePublisher(IMessageBus bus)
    {
        _bus = bus;
    }

    public async Task PublishAsync(
        SqlOptimizationCreatedEvent domainEvent,
        CancellationToken cancellationToken)
    {
        using var activity = AppTelemetry.ActivitySource.StartActivity(
            "mq.publish.sql_optimization_created",
            ActivityKind.Producer);

        activity?.SetTag("messaging.system", "rabbitmq");
        activity?.SetTag("messaging.destination.name", "sql-optimization-created");
        activity?.SetTag("workflow.run_id", domainEvent.WorkflowRunId);

        var message = new BusMessage
        {
            Body = JsonSerializer.Serialize(domainEvent),
            Headers = new Dictionary<string, string>()
        };

        var propagationContext = new PropagationContext(
            Activity.Current?.Context ?? default,
            Baggage.Current);

        Propagator.Inject(
            propagationContext,
            message.Headers,
            static (headers, key, value) =>
            {
                headers[key] = value;
            });

        await _bus.PublishAsync(message, cancellationToken);
    }
}
```

消费消息时提取上下文：

```csharp
public sealed class SqlOptimizationCreatedConsumer
{
    private static readonly TextMapPropagator Propagator =
        Propagators.DefaultTextMapPropagator;

    public async Task ConsumeAsync(
        BusMessage message,
        CancellationToken cancellationToken)
    {
        var parentContext = Propagator.Extract(
            default,
            message.Headers,
            static (headers, key) =>
            {
                return headers.TryGetValue(key, out var value)
                    ? [value]
                    : [];
            });

        Baggage.Current = parentContext.Baggage;

        using var activity = AppTelemetry.ActivitySource.StartActivity(
            "mq.consume.sql_optimization_created",
            ActivityKind.Consumer,
            parentContext.ActivityContext);

        try
        {
            activity?.SetTag("messaging.system", "rabbitmq");
            activity?.SetTag("messaging.operation.name", "process");
            activity?.SetTag("messaging.destination.name", "sql-optimization-created");

            var domainEvent = JsonSerializer.Deserialize<SqlOptimizationCreatedEvent>(
                message.Body)!;

            activity?.SetTag("workflow.run_id", domainEvent.WorkflowRunId);
            activity?.SetTag("project.id", domainEvent.ProjectId);

            await HandleBusinessAsync(domainEvent, cancellationToken);

            activity?.SetStatus(ActivityStatusCode.Ok);
        }
        catch (Exception ex)
        {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            activity?.AddException(ex);

            BusinessMetrics.MqConsumeFailed.Add(
                1,
                new KeyValuePair<string, object?>("message.type", "SqlOptimizationCreatedEvent"));

            throw;
        }
    }
}
```

## 9.12 Human Review 对接示例

```csharp
public sealed class ReviewTelemetryService
{
    public void MarkReviewCreated(ReviewTask reviewTask)
    {
        Activity.Current?.AddEvent(new ActivityEvent(
            "human_review.created",
            tags: new ActivityTagsCollection
            {
                ["review.task_id"] = reviewTask.Id,
                ["workflow.run_id"] = reviewTask.WorkflowRunId,
                ["review.type"] = reviewTask.Type,
                ["review.status"] = "pending"
            }));
    }

    public void MarkReviewCompleted(ReviewTask reviewTask)
    {
        var waitSeconds =
            (reviewTask.CompletedAt!.Value - reviewTask.CreatedAt).TotalSeconds;

        Activity.Current?.AddEvent(new ActivityEvent(
            "human_review.completed",
            tags: new ActivityTagsCollection
            {
                ["review.task_id"] = reviewTask.Id,
                ["workflow.run_id"] = reviewTask.WorkflowRunId,
                ["review.result"] = reviewTask.Decision,
                ["review.wait_seconds"] = waitSeconds
            }));

        BusinessMetrics.ReviewWaitDuration.Record(
            waitSeconds,
            new KeyValuePair<string, object?>("review.type", reviewTask.Type),
            new KeyValuePair<string, object?>("review.result", reviewTask.Decision));
    }
}
```

---

# 10. 生产级必须补充的关键点

初次接触 OpenTelemetry，除了 Trace / Metric / Log，还必须补齐下面这些认知：

```text
1. OTel 不是监控平台，而是采集标准 + SDK + Collector 管道。
2. 自动采集只是起点，业务观测必须手动补。
3. Collector 很重要，生产上不建议应用直接连所有后端。
4. Sampling 很关键，否则成本和数据量会爆。
5. Metric 标签基数是大坑，乱加 label 会打爆时序存储。
6. Trace 上下文传播是分布式链路能否串起来的关键。
7. 日志必须和 Trace 关联，否则还是“孤岛日志”。
8. Semantic Conventions 要尽早定规范。
9. 安全脱敏必须提前设计，尤其是 AI / SQL / Prompt。
10. OTel 不能替代业务审计表、数据库记录、Run 记录。
11. 告警不是有数据就行，必须设计 SLI / SLO。
12. 本地、测试、生产环境的采集策略不同。
13. OTel 有成本，不能无脑全量采集。
14. MAF 原生 OTel 只能覆盖 Agent 层，不是全系统闭环。
```

---

# 11. Collector：生产环境的观测数据网关

Collector 可以理解成：

> 可观测性数据的网关。

典型结构：

```text
.NET 应用 / MAF / API
  ↓ OTLP
OpenTelemetry Collector
  ↓
Trace Backend / Metric Backend / Log Backend
```

Collector 的价值：

```text
1. 统一接收应用数据
2. 批量发送，减少网络开销
3. 做采样，降低成本
4. 做脱敏，保护敏感信息
5. 做过滤，丢弃无价值数据
6. 做多路导出，同时发到多个后端
7. 避免业务应用绑定某个监控厂商
```

简化配置：

```yaml
receivers:
  otlp:
    protocols:
      grpc:
      http:

processors:
  memory_limiter:
    limit_mib: 512
    spike_limit_mib: 128
    check_interval: 5s

  batch:

exporters:
  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true

  prometheus:
    endpoint: "0.0.0.0:9464"

  debug:

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp/tempo, debug]

    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheus, debug]
```

生产上要思考：

```text
OTLP 发给谁？
Collector 是否高可用？
Collector 是否限流？
Collector 是否采样？
Collector 是否脱敏？
Collector 掉了应用会怎样？
```

---

# 12. Sampling：采样策略

Trace 如果全量采集，数据量可能非常大，尤其是你的系统包含：

```text
HTTP
EF Core SQL
Redis
MQ
MAF Workflow
Agent
LLM
ToolCall
MCP
```

一次请求可能产生几十个 span。

## 12.1 Head Sampling

请求刚开始就决定是否保留。

优点：

- 简单
- 省资源

缺点：

- 可能把错误请求提前丢掉

## 12.2 Tail Sampling

请求结束后，根据结果决定是否保留。

优点：

- 可以保留错误请求
- 可以保留慢请求
- 可以保留特定业务请求

缺点：

- Collector 配置复杂
- Collector 内存和计算成本更高

## 12.3 生产建议

```text
开发环境：100% 采集。
测试环境：50% 或 100%。
生产环境：
  - 错误 Trace 100% 保留
  - 慢请求 Trace 100% 保留
  - Agent / Tool 失败 Trace 100% 保留
  - Workflow 恢复失败 Trace 100% 保留
  - 普通成功请求按比例采样
```

示例策略：

```text
HTTP 5xx：保留
Agent 执行失败：保留
Tool 调用失败：保留
SQL 超过 1s：保留
普通成功请求：保留 5% ~ 20%
```

---

# 13. Metric 标签基数 Cardinality

Metric 可以带标签，例如：

```text
agent.name = SqlAnalyzeAgent
status = failed
db.system = postgresql
```

这些没问题，因为取值有限。

但是不要这样：

```text
workflow.run_id = 每次都不同
user.id = 每个用户不同
sql.text = 每条 SQL 不同
trace.id = 每次请求不同
message.id = 每条消息不同
```

这些是高基数字段，会导致时序数量爆炸。

## 13.1 错误示例

```csharp
BusinessMetrics.AgentRunFailed.Add(
    1,
    new KeyValuePair<string, object?>("workflow.run_id", workflowRunId),
    new KeyValuePair<string, object?>("user.id", userId),
    new KeyValuePair<string, object?>("sql.hash", sqlHash));
```

## 13.2 正确示例

```csharp
BusinessMetrics.AgentRunFailed.Add(
    1,
    new KeyValuePair<string, object?>("agent.name", "SqlAnalyzeAgent"),
    new KeyValuePair<string, object?>("workflow.name", "SqlOptimizationWorkflow"),
    new KeyValuePair<string, object?>("status", "failed"));
```

## 13.3 放置规则

| 字段 | Trace Attribute | Log | Metric Label | DB 表 |
|---|---:|---:|---:|---:|
| workflow.run_id | 可以 | 可以 | 不建议 | 必须 |
| agent.run_id | 可以 | 可以 | 不建议 | 必须 |
| project.id | 可以 | 可以 | 谨慎 | 必须 |
| sql.hash | 可以 | 可以 | 谨慎 | 必须 |
| db.system | 可以 | 可以 | 可以 | 可以 |
| agent.name | 可以 | 可以 | 可以 | 可以 |
| tool.name | 可以 | 可以 | 可以 | 可以 |
| raw_sql | 不建议 | 不建议 | 禁止 | 视脱敏策略 |
| prompt.raw | 不建议 | 不建议 | 禁止 | 视脱敏策略 |

---

# 14. Context Propagation：链路能否串起来的关键

分布式链路能否串起来，关键在于上下文传播。

例如：

```text
API 服务
  ↓ HTTP
MCP Server
  ↓ MQ
Consumer
  ↓ HTTP
LLM Proxy
```

如果上下文没有传播，下游会生成新的 Trace：

```text
Trace A：API
Trace B：MCP
Trace C：Consumer
Trace D：LLM Proxy
```

正确情况下应该是：

```text
Trace A：API → MCP → MQ Consumer → LLM Proxy
```

HTTP / gRPC 通常自动传播，常见 header：

```text
traceparent
tracestate
baggage
```

MQ、后台任务、定时任务通常需要手动 Inject / Extract。

发布消息：

```text
把 trace context 注入 message headers。
```

消费消息：

```text
从 message headers 提取 trace context 作为 parent context。
```

---

# 15. Log Correlation：日志和 Trace 的关联

日志最好自动带上：

```text
trace_id
span_id
service.name
deployment.environment
workflow_run_id
agent_run_id
project_id
```

这样可以做到：

```text
看到错误日志
  ↓
点击 trace_id
  ↓
看到完整请求链路
  ↓
定位慢在 SQL、Redis、LLM、Tool、MQ 还是审核等待
```

.NET 中可以使用：

```csharp
builder.Logging.AddOpenTelemetry(logging =>
{
    logging.IncludeFormattedMessage = true;
    logging.IncludeScopes = true;

    logging.AddOtlpExporter();
});
```

业务字段通过 Scope 进入日志：

```csharp
using var scope = _logger.BeginScope(new Dictionary<string, object?>
{
    ["WorkflowRunId"] = command.WorkflowRunId,
    ["ProjectId"] = command.ProjectId,
    ["SqlHash"] = command.SqlHash
});

_logger.LogInformation("开始执行 SQL 优化 Workflow");
```

---

# 16. Semantic Conventions：字段规范

Semantic Conventions 解决的是：

```text
不同服务、不同语言、不同库，对同一种东西使用同一套字段名。
```

例如 HTTP 不要乱写：

```text
method
httpMethod
request_method
```

优先遵循标准字段：

```text
http.request.method
url.path
http.response.status_code
```

数据库字段：

```text
db.system
db.name
db.operation
server.address
```

消息字段：

```text
messaging.system
messaging.destination.name
messaging.operation.name
```

GenAI / Agent 相关字段优先参考 GenAI Semantic Conventions。

自定义业务字段建议统一命名：

```text
dboptimizer.workflow.name
dboptimizer.workflow.run_id
dboptimizer.workflow.node.name

dboptimizer.agent.name
dboptimizer.agent.run_id

dboptimizer.tool.name
dboptimizer.tool.type

dboptimizer.review.task_id
dboptimizer.review.type
dboptimizer.review.result

dboptimizer.sql.hash
dboptimizer.sql.source
dboptimizer.sql.optimization_id
```

原则：

```text
标准字段优先用 OTel 官方语义。
业务字段使用自己的 namespace。
```

---

# 17. Resource：服务身份信息

Resource 描述的是：

> 这条遥测数据来自哪个服务、哪个环境、哪个实例。

如果没有 Resource，你的 Trace / Metric / Log 会变得很混乱：

```text
不知道是哪个服务产生的
不知道是 dev 还是 prod
不知道是哪台机器
不知道是哪一个版本
```

建议每个服务都设置：

```text
service.name
service.version
service.instance.id
deployment.environment
host.name
```

示例：

```csharp
builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource =>
    {
        resource.AddService(
            serviceName: "db-optimizer-api",
            serviceVersion: "1.0.0",
            serviceInstanceId: Environment.MachineName);

        resource.AddAttributes(new Dictionary<string, object>
        {
            ["deployment.environment"] = builder.Environment.EnvironmentName,
            ["team.name"] = "ai-platform"
        });
    });
```

---

# 18. 安全脱敏策略

AI + 数据库优化系统中敏感信息非常多：

```text
完整 SQL
表结构
字段名
业务数据
Prompt
LLM Response
Tool 参数
Tool 返回结果
数据库连接字符串
用户信息
API Key
```

生产环境建议：

```text
不记录完整 prompt
不记录完整 LLM response
不记录完整 SQL
不记录连接字符串
不记录用户隐私
只记录 hash、长度、类型、状态、耗时
```

推荐：

```csharp
activity?.SetTag("sql.hash", sqlHash);
activity?.SetTag("sql.length", sql.Length);
activity?.SetTag("llm.model", modelName);
activity?.SetTag("prompt.length", prompt.Length);
activity?.SetTag("tool.name", toolName);
```

不推荐：

```csharp
activity?.SetTag("sql.text", sql);
activity?.SetTag("prompt.raw", prompt);
activity?.SetTag("llm.response", response);
```

## 18.1 日志等级规范

| Level | 用法 |
|---|---|
| Trace / Debug | 本地调试，生产默认关闭 |
| Information | 关键业务状态变化 |
| Warning | 可恢复异常、降级、慢 SQL、重试 |
| Error | 请求失败、任务失败、不可忽略异常 |
| Critical | 系统不可用、数据损坏、核心依赖不可用 |

你的项目可以这样定：

```text
Information：
- Workflow 创建
- AgentRun 开始 / 结束
- ReviewTask 创建
- 优化结果保存

Warning：
- 慢 SQL
- Tool 重试
- LLM 输出校验失败
- 缓存穿透
- MQ 重试

Error：
- Workflow 执行失败
- Agent 执行失败
- Tool 最终失败
- Checkpoint 恢复失败
- 数据库连接失败

Critical：
- 主数据库不可用
- MQ 整体不可用
- Checkpoint 数据损坏
```

---

# 19. OTel 不能替代业务数据库表

这是 MAF / Agent 系统里非常关键的点。

OpenTelemetry 是观测数据，不是业务事实表。

它适合回答：

```text
为什么慢？
哪里错？
失败率多少？
这次链路经过了什么？
```

业务数据库适合回答：

```text
用户历史记录是什么？
这个 Agent 输出了什么？
这次审核意见是什么？
这个优化建议最终是否采纳？
这个 Workflow 能不能恢复？
这个工单当前状态是什么？
```

所以你仍然需要业务表：

```text
AgentSession
AgentMessage
AgentRun
AgentToolCall
AgentApproval
AgentEvent
WorkflowRun
WorkflowCheckpoint
ReviewTask
AuditLog
SqlOptimizationTask
SqlOptimizationResult
```

关系：

```text
业务数据库：保存事实、状态、审计、恢复数据。
OpenTelemetry：保存观测、性能、链路、指标、排障数据。
```

不要把 OTel 当成主业务存储。

---

# 20. 告警、SLI、SLO 设计

Metric 的目的不只是画图，而是告警和 SLO。

## 20.1 推荐 SLI

```text
接口成功率
接口 P95 / P99 延迟
Agent 成功率
Agent P95 耗时
Tool 成功率
MCP 调用失败率
慢 SQL 数量
MQ 消费失败率
MQ Lag
Workflow 恢复失败数
人工审核超时数量
Token 消耗异常
```

## 20.2 示例告警规则

```text
API 5xx > 1% 持续 5 分钟
SQL 优化接口 P95 > 5s 持续 10 分钟
SqlAnalyzeAgent 失败率 > 5% 持续 10 分钟
MCP Tool 失败率 > 10% 持续 5 分钟
MQ Lag > 1000 持续 5 分钟
Workflow 恢复失败数 > 0
人工审核等待超过 24 小时
LLM Token 使用量突增 3 倍
```

没有告警，Observability 只是事后排查。

有告警，才是主动发现问题。

---

# 21. 环境策略：Dev / Test / Prod

## 21.1 Dev

```text
Trace 100%
Log Debug 可开
可以记录更多上下文
可以用 Aspire Dashboard
可以临时开启 SensitiveData，但必须控制环境
```

## 21.2 Test / Staging

```text
Trace 50% ~ 100%
验证 Trace 是否完整
验证 Metric 是否能查到
验证 Log 是否能关联 Trace
验证 Collector 是否收到数据
验证告警是否有效
```

## 21.3 Prod

```text
Trace 采样
错误和慢请求优先保留
SensitiveData 关闭
Metric 全量，但控制 label
Log 控制数量和等级
Collector 高可用
告警开启
敏感字段脱敏 / hash / truncate
```

---

# 22. 如何验证接入成功

接入 OTel 后，不是程序能跑就算成功，要验证下面这些：

```text
1. HTTP 请求是否有 Trace
2. Trace 是否包含 EF Core SQL Span
3. Trace 是否包含 MAF Agent Span
4. Trace 是否包含 Tool Span
5. Error 是否带异常状态
6. Log 是否带 trace_id / span_id
7. Metric 是否能查到
8. Collector 是否收到数据
9. Grafana / Tempo / Jaeger 是否能看到
10. 采样是否按预期工作
11. 敏感字段是否没有泄露
12. MQ 发布和消费是否在同一 Trace 或能通过 Link 关联
13. Workflow 恢复后是否能继续关联业务 ID
```

你的 MAF 项目要专门做测试用例：

```text
正常 SQL 优化请求
慢 SQL 请求
Tool 调用失败
LLM 调用失败
Workflow 进入人工审核
Workflow 恢复执行
Checkpoint 保存和恢复
MQ 发布和消费
Redis 缓存命中和未命中
```

---

# 23. 推荐的观测规范

## 23.1 必须建 Span 的地方

```text
HTTP API 入口
Workflow Run
Workflow Node / Executor
Agent Run
Tool Call / MCP Call
LLM Call
DB Query
Redis Call
MQ Publish
MQ Consume
外部 HTTP API
Checkpoint Save
Checkpoint Restore
Review Resume
```

## 23.2 必须打 Metric 的地方

```text
HTTP 请求耗时、错误数
Agent 执行耗时、失败数
Tool 调用耗时、失败数
LLM token 使用量
慢 SQL 数量
DB 连接池状态
Redis 命中率、超时数
MQ 发布 / 消费失败数、lag
人工审核等待时间
Workflow 恢复失败次数
Checkpoint 保存 / 恢复耗时
```

## 23.3 必须打 Log 的地方

```text
业务异常
外部依赖失败
人工审核意见
Agent 输出校验失败
Tool 参数校验失败
Workflow 恢复失败
权限拒绝
安全拦截
Checkpoint 数据异常
```

## 23.4 必须放 Attribute 的字段

```text
service.name
deployment.environment
workflow.name
workflow.run_id
workflow.node.name
agent.name
agent.run_id
tool.name
tool.call_id
db.system
db.instance_id
sql.hash
project.id
review.task_id
```

其中：

```text
workflow.run_id / agent.run_id / sql.hash
可以放 Trace 和 Log。
不建议作为 Metric label。
```

## 23.5 推荐最终链路结构

```text
HTTP POST /api/sql-optimization/runs
 ├─ workflow.sql_optimization.run
 │   ├─ workflow.node.load_context
 │   ├─ db.query.project_config
 │   ├─ cache.sql_analysis.get
 │   ├─ agent.sql_planner.run
 │   │   └─ chat model
 │   ├─ tool.mysql.get_table_schema
 │   ├─ tool.mysql.explain_sql
 │   ├─ agent.sql_analyzer.run
 │   │   └─ chat model
 │   ├─ event: slow_sql.detected
 │   ├─ event: human_review.created
 │   └─ result.save
```

---

# 24. 面试级回答模板

## 24.1 什么是 OpenTelemetry

OpenTelemetry 是一套厂商中立的可观测性标准和工具链，提供 API、SDK、Collector 和语义规范，用于统一采集 traces、metrics、logs 等遥测数据。它不是监控平台，而是观测数据的标准层和采集管道。后端可以接 Prometheus、Grafana、Jaeger、Tempo、Loki、Elastic 或商业 APM。

## 24.2 OpenTelemetry 和日志有什么区别

日志记录的是某个点发生了什么；OpenTelemetry 更关注把一次请求的完整链路、指标趋势和日志细节关联起来。实际项目中日志仍然要保留，但通过 OpenTelemetry 可以让日志带 trace_id / span_id，并和 Trace、Metric 统一查询。

## 24.3 Trace、Metric、Log 分别解决什么问题

Trace 用于定位一次请求慢在哪里、失败在哪里；Metric 用于统计趋势、失败率、P95/P99、告警；Log 用于记录具体异常堆栈、业务原因和人工可读上下文。Event 用于标记 Span 内关键瞬间，Baggage 用于跨服务传递上下文。

## 24.4 MAF 是否原生支持 OpenTelemetry

MAF 有原生 OpenTelemetry 支持，可以自动生成 Agent、LLM 调用和 Tool 调用相关的 traces、metrics、logs，比如 invoke_agent、chat、execute_tool 这些 span。但它主要覆盖 Agent 层，不会自动理解业务中的 workflowRunId、projectId、慢 SQL、人工审核等待、MQ Lag 等，所以生产系统仍然需要补充自定义 Activity、Metric 和日志。

## 24.5 为什么生产上需要 Collector

Collector 是观测数据网关，用于统一接收、处理和导出 telemetry。它可以做批量发送、内存限制、采样、脱敏、过滤和多后端导出。生产上不建议每个应用直接对接多个观测后端，因为这样会导致配置复杂、耦合厂商、难以统一治理。

## 24.6 Metric 标签为什么不能乱加

Metric label 会影响时序数量。如果把 userId、workflowRunId、traceId、sqlText 这类高基数字段放进 label，会导致指标存储爆炸。正确做法是 Metric label 只放低基数字段，比如 db.system、agent.name、tool.name、status。高基数字段放到 Trace Attribute、Log 或业务数据库中。

## 24.7 OpenTelemetry 能不能替代业务表

不能。OpenTelemetry 是观测数据，用于排障、性能分析、告警和链路追踪；业务表用于保存事实、状态、审计和恢复数据。例如 AgentRun、AgentMessage、ToolCall、WorkflowCheckpoint、ReviewTask 这些仍然必须落库。OTel 不能作为业务主存储。

---

# 25. 最终总结

OpenTelemetry 的正确定位：

```text
OpenTelemetry 不是高级日志工具。
OpenTelemetry 是生产系统的观测数据标准层。
```

它负责把下面这些系统统一串起来：

```text
应用 API
数据库
Redis
MQ
MAF Workflow
Agent
LLM
ToolCall
MCP
人工审核
Runtime
```

但是它不能替代：

```text
业务数据库
审计表
WorkflowCheckpoint
AgentSession
AgentMessage
ReviewTask
权限系统
业务状态机
```

对于你的数据库优化 AI 项目，推荐最终方案是：

```text
MAF 原生 OpenTelemetry
+
.NET 自动 Instrumentation
+
自定义业务 Activity / Meter / ILogger
+
OpenTelemetry Collector
+
Prometheus / Tempo / Loki / Grafana
+
业务数据库持久化
```

核心原则：

```text
Trace 看链路。
Metric 看趋势。
Log 看细节。
Event 标记关键点。
Baggage 传上下文。
Collector 做管道治理。
Sampling 控制成本。
Cardinality 控制指标爆炸。
Semantic Conventions 统一字段。
业务表保存事实状态。
```

一句话落地判断：

> 如果你想知道“一次请求慢在哪里”，用 Trace；如果你想知道“过去一段时间系统是否健康”，用 Metric；如果你想知道“具体为什么失败”，用 Log；如果你想把 AI Agent、Tool、SQL、MQ、审核全部串起来，就用 OpenTelemetry 做统一观测底座。

---

# 26. 参考资料

- OpenTelemetry 官方文档：<https://opentelemetry.io/docs/>
- OpenTelemetry Signals：<https://opentelemetry.io/docs/concepts/signals/>
- OpenTelemetry Traces：<https://opentelemetry.io/docs/concepts/signals/traces/>
- OpenTelemetry Metrics：<https://opentelemetry.io/docs/concepts/signals/metrics/>
- OpenTelemetry Logs：<https://opentelemetry.io/docs/concepts/signals/logs/>
- OpenTelemetry Collector：<https://opentelemetry.io/docs/collector/>
- OpenTelemetry Semantic Conventions：<https://opentelemetry.io/docs/concepts/semantic-conventions/>
- .NET OpenTelemetry 文档：<https://opentelemetry.io/docs/languages/dotnet/>
- Microsoft .NET Observability with OpenTelemetry：<https://learn.microsoft.com/en-us/dotnet/core/diagnostics/observability-with-otel>
- Microsoft Agent Framework Observability：<https://learn.microsoft.com/en-us/agent-framework/agents/observability>
- Microsoft Agent Framework Pipeline：<https://learn.microsoft.com/en-us/agent-framework/agents/agent-pipeline>

