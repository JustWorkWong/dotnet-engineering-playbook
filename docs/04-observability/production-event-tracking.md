# 数据埋点设计生产级方案

> 适用场景：C# + Vue + MAF / Agent / Workflow 的 AI 数据库优化系统  
> 目标：把用户行为、业务流程、Agent 执行、Workflow 节点、人工审核、AI 成本、效果评估串成一个可分析、可审计、可复盘、可优化的数据体系。

---

# 1. 埋点到底解决什么问题？

埋点不是简单记录“用户点了什么”，它解决的是产品、业务、AI 效果、成本、性能、安全审计等问题。

| 目标 | 说明 |
|---|---|
| 产品分析 | 用户用了哪些功能、在哪些页面流失、哪个入口转化差 |
| 业务分析 | SQL 优化任务提交量、审核通过率、优化建议采纳率 |
| AI 效果分析 | Agent 是否真的解决问题，失败原因是什么 |
| 成本分析 | 每次 Agent Run 消耗多少 Token、调用多少工具、花多少钱 |
| 性能分析 | 页面慢、接口慢、Agent 慢、Workflow 卡在哪个节点 |
| 风险审计 | 谁操作了什么，是否调用了高危工具，审核是否绕过 |
| 运营增长 | 哪类用户最活跃，哪个入口带来最多任务 |

一句话：

> 埋点是为了让系统上线后，可以回答“发生了什么、为什么发生、影响什么、是否值得、未来如何复现”。

---

# 2. 埋点、日志、Trace 的区别

很多初学者会把埋点和日志混在一起，这是生产系统的大坑。

| 类型 | 目的 | 示例 |
|---|---|---|
| Log 日志 | 排查系统问题 | SQL 执行异常、Redis 超时、接口抛异常 |
| Trace 链路 | 定位性能瓶颈 | 一次请求经过 API、Agent、Tool、DB 的耗时 |
| Event 埋点 | 产品与业务分析 | 用户点击“生成优化建议”、审核通过、采纳建议 |

## 2.1 正确理解

```text
Log 看系统为什么坏了
Trace 看慢在哪里
Event 看用户和业务发生了什么
```

## 2.2 推荐关系

```text
业务埋点事件 analytics_event
        ↓ 关联
OpenTelemetry trace_id / span_id
        ↓ 反查
链路性能、接口耗时、工具耗时、数据库耗时
```

---

# 3. 埋点总体架构

## 3.1 第一版架构

第一版不需要太复杂，可以先用 PostgreSQL 存储埋点明细。

```text
Vue 前端
Backend API
MAF Agent / Workflow
        ↓
TrackingService
        ↓
PostgreSQL: analytics_event
```

## 3.2 生产级演进架构

当事件量变大后，再引入 MQ 和 ClickHouse。

```text
┌──────────────┐
│ Vue 前端      │
│ click/view   │
└──────┬───────┘
       │
       │ /api/track/events
       ▼
┌────────────────────┐
│ Tracking API        │
│ 校验 / 补全 / 脱敏   │
└──────┬─────────────┘
       │
       ├──────────────┐
       │              │
       ▼              ▼
┌─────────────┐   ┌──────────────┐
│ PostgreSQL  │   │ MQ / Kafka    │
│ 明细存储     │   │ 异步分析      │
└─────────────┘   └──────┬───────┘
                         ▼
                  ┌──────────────┐
                  │ ClickHouse/ES │
                  │ 高性能分析     │
                  └──────────────┘
```

## 3.3 推荐分工

| 存储 | 适合存什么 |
|---|---|
| PostgreSQL | 业务明细、审计、低中频事件 |
| ClickHouse | 高频行为分析、BI 聚合、漏斗分析 |
| OpenTelemetry Backend | Trace、Metric、Log |
| MQ | 异步削峰、失败重试、解耦埋点写入 |
| 对象存储 | 大型快照、原始输入、长文本归档 |

---

# 4. 埋点分层设计

建议分成 6 层：

```text
前端行为埋点
后端接口埋点
Agent 执行埋点
Workflow 节点埋点
人工审核埋点
业务效果埋点
```

---

# 5. 前端行为埋点

前端埋点记录用户在页面上的行为。

| 事件名 | 触发时机 |
|---|---|
| page_view | 页面打开 |
| button_click | 按钮点击 |
| form_submit | 表单提交 |
| tab_switch | 切换 SQL 优化 / DB 优化 |
| file_upload | 上传慢 SQL 文件 |
| review_click | 点击审核按钮 |
| copy_result | 复制优化建议 |
| export_report | 导出报告 |

## 5.1 示例事件

```json
{
  "eventName": "sql_optimize_submit_click",
  "eventType": "frontend",
  "page": "/sql-optimize",
  "component": "OptimizeButton",
  "action": "click",
  "properties": {
    "databaseType": "postgresql",
    "inputType": "manual_sql"
  }
}
```

## 5.2 Vue track.ts 封装

```ts
import axios from "axios";

export interface TrackEvent {
  eventName: string;
  eventType: string;
  pageUrl?: string;
  referrer?: string;
  taskId?: string;
  workflowRunId?: string;
  agentRunId?: string;
  reviewId?: string;
  properties?: Record<string, any>;
}

export async function track(event: TrackEvent) {
  try {
    await axios.post("/api/track/events", {
      ...event,
      pageUrl: window.location.pathname,
      referrer: document.referrer,
      clientOccurredAt: new Date().toISOString()
    });
  } catch {
    // 埋点失败不能影响主流程
  }
}
```

## 5.3 页面浏览埋点

```ts
router.afterEach((to, from) => {
  track({
    eventName: "page_view",
    eventType: "frontend",
    properties: {
      from: from.fullPath,
      to: to.fullPath
    }
  });
});
```

## 5.4 按钮点击埋点

```ts
async function submitSqlOptimize() {
  track({
    eventName: "sql_optimize_submit_click",
    eventType: "frontend",
    properties: {
      databaseType: form.databaseType,
      inputType: form.inputType
    }
  });

  await api.createSqlOptimizationTask(form);
}
```

---

# 6. 后端接口埋点

后端埋点记录接口调用、业务动作、异常、耗时。

| 事件名 | 触发时机 |
|---|---|
| api_request_started | 接口开始 |
| api_request_completed | 接口结束 |
| api_request_failed | 接口异常 |
| optimization_task_created | 创建优化任务 |
| review_submitted | 提交审核意见 |
| report_exported | 导出报告 |

## 6.1 示例事件

```json
{
  "eventName": "optimization_task_created",
  "eventType": "backend",
  "properties": {
    "taskId": "task_001",
    "taskType": "sql_optimize",
    "databaseType": "mysql",
    "source": "slow_query_capture"
  }
}
```

---

# 7. Agent 埋点

AI 系统里，Agent 埋点非常关键。它用于分析模型效果、工具调用、Token 成本、错误原因、风险行为。

| 事件名 | 说明 |
|---|---|
| agent_run_started | Agent 开始执行 |
| agent_run_completed | Agent 成功结束 |
| agent_run_failed | Agent 执行失败 |
| agent_model_called | 调用大模型 |
| agent_tool_called | 调用工具 |
| agent_tool_failed | 工具失败 |
| agent_context_compacted | 上下文压缩 |
| agent_memory_recalled | 记忆召回 |
| agent_output_generated | 生成最终结果 |

## 7.1 Agent Tool 调用事件

```json
{
  "eventName": "agent_tool_called",
  "eventType": "agent",
  "properties": {
    "agentRunId": "run_001",
    "agentName": "SqlTuningAgent",
    "toolName": "ExplainAnalyzeTool",
    "toolType": "mcp",
    "approved": true,
    "durationMs": 820
  }
}
```

## 7.2 Agent 运行指标建议

除了事件表，还建议单独建 Agent Run 汇总表。

需要关注：

```text
agent_run_id
task_id
workflow_run_id
agent_name
model_name
provider
prompt_tokens
completion_tokens
total_tokens
tool_call_count
memory_recall_count
context_compaction_count
duration_ms
estimated_cost
status
error_code
error_message
started_at
completed_at
```

---

# 8. Workflow 埋点

MAF Workflow 需要记录每个节点状态、耗时、输入输出规模、失败原因和 checkpoint。

| 事件名 | 说明 |
|---|---|
| workflow_run_started | 工作流开始 |
| workflow_step_started | 节点开始 |
| workflow_step_completed | 节点完成 |
| workflow_step_failed | 节点失败 |
| workflow_checkpoint_saved | checkpoint 保存 |
| workflow_resume_started | 从 checkpoint 恢复 |
| workflow_human_review_requested | 请求人工审核 |
| workflow_human_review_completed | 审核完成 |

## 8.1 Workflow Step 完成事件

```json
{
  "eventName": "workflow_step_completed",
  "eventType": "workflow",
  "properties": {
    "workflowRunId": "wf_001",
    "stepName": "SqlAnalysisExecutor",
    "stepType": "executor",
    "durationMs": 2300,
    "status": "completed"
  }
}
```

## 8.2 Workflow 关键指标

```text
每个 workflow_run 的总耗时
每个 step 的 P50 / P95 / P99
失败节点排行
checkpoint 保存次数
resume 成功率
人工审核等待时间
```

---

# 9. 人工审核埋点

你的系统有 Human-in-the-loop，所以审核链路必须单独设计。

| 事件名 | 说明 |
|---|---|
| review_required | 进入审核 |
| review_page_view | 打开审核页 |
| review_approved | 审核通过 |
| review_rejected | 审核拒绝 |
| review_need_rewrite | 要求 Agent 重写 |
| review_comment_submitted | 提交审核意见 |

## 9.1 审核拒绝事件

```json
{
  "eventName": "review_rejected",
  "eventType": "review",
  "properties": {
    "reviewId": "review_001",
    "taskId": "task_001",
    "reason": "risk_too_high",
    "reviewerRole": "admin"
  }
}
```

## 9.2 审核链路要统计

```text
审核请求数
审核通过率
审核拒绝率
需要重写率
平均审核等待时间
人工推翻 AI 的比例
高风险建议比例
```

---

# 10. 业务效果埋点

业务效果埋点用于证明系统是否真的创造价值。

| 事件名 | 说明 |
|---|---|
| optimization_suggestion_generated | 生成优化建议 |
| optimization_suggestion_accepted | 用户采纳建议 |
| optimization_suggestion_rejected | 用户拒绝建议 |
| before_after_metric_recorded | 优化前后指标记录 |
| slow_query_resolved | 慢 SQL 被解决 |
| index_recommendation_applied | 索引建议被应用 |
| optimization_rollback | 优化上线后回滚 |

## 10.1 建议采纳事件

```json
{
  "eventName": "optimization_suggestion_accepted",
  "eventType": "business",
  "properties": {
    "taskId": "task_001",
    "suggestionType": "add_index",
    "databaseType": "postgresql",
    "estimatedImprovement": "75%",
    "riskLevel": "medium"
  }
}
```

## 10.2 对数据库优化系统特别重要的效果指标

```text
建议采纳率
优化上线率
优化后平均耗时下降比例
优化后慢查询减少数量
索引建议采纳率
索引建议回滚率
Token 成本 / 单次成功优化
人工审核成本 / 单次成功优化
```

---

# 11. 核心数据库设计

## 11.1 analytics_event 通用事件表

```sql
CREATE TABLE analytics_event (
    id BIGSERIAL PRIMARY KEY,

    event_id VARCHAR(64) NOT NULL UNIQUE,
    event_name VARCHAR(128) NOT NULL,
    event_type VARCHAR(64) NOT NULL,
    schema_version VARCHAR(16) NOT NULL DEFAULT '1.0',
    event_source VARCHAR(32),

    user_id VARCHAR(64),
    user_pseudo_id VARCHAR(64),
    tenant_id VARCHAR(64),
    session_id VARCHAR(64),

    request_id VARCHAR(128),
    trace_id VARCHAR(128),
    span_id VARCHAR(128),

    page_url TEXT,
    referrer TEXT,
    user_agent TEXT,
    ip_hash VARCHAR(128),

    task_id VARCHAR(64),
    workflow_run_id VARCHAR(64),
    agent_run_id VARCHAR(64),
    review_id VARCHAR(64),

    experiment_id VARCHAR(64),
    variant_id VARCHAR(64),

    prompt_version VARCHAR(64),
    model_version VARCHAR(64),
    tool_policy_version VARCHAR(64),
    approval_policy_version VARCHAR(64),
    rag_index_version VARCHAR(64),
    retriever_version VARCHAR(64),
    reranker_version VARCHAR(64),

    is_sampled BOOLEAN NOT NULL DEFAULT FALSE,
    sampling_rate NUMERIC(5,4),
    pii_level VARCHAR(16),

    properties JSONB NOT NULL DEFAULT '{}'::jsonb,

    client_occurred_at TIMESTAMPTZ,
    server_received_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at TIMESTAMPTZ,

    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_analytics_event_name_time
ON analytics_event(event_name, server_received_at DESC);

CREATE INDEX idx_analytics_event_user_time
ON analytics_event(user_id, server_received_at DESC);

CREATE INDEX idx_analytics_event_session_time
ON analytics_event(session_id, server_received_at DESC);

CREATE INDEX idx_analytics_event_task
ON analytics_event(task_id);

CREATE INDEX idx_analytics_event_workflow_run
ON analytics_event(workflow_run_id);

CREATE INDEX idx_analytics_event_agent_run
ON analytics_event(agent_run_id);

CREATE INDEX idx_analytics_event_review
ON analytics_event(review_id);

CREATE INDEX idx_analytics_event_trace
ON analytics_event(trace_id);

CREATE INDEX idx_analytics_event_properties_gin
ON analytics_event USING GIN(properties);
```

---

## 11.2 agent_run_metric 汇总表

```sql
CREATE TABLE agent_run_metric (
    id BIGSERIAL PRIMARY KEY,

    agent_run_id VARCHAR(64) NOT NULL UNIQUE,
    task_id VARCHAR(64),
    workflow_run_id VARCHAR(64),
    agent_name VARCHAR(128),

    model_name VARCHAR(128),
    model_version VARCHAR(64),
    provider VARCHAR(64),
    prompt_version VARCHAR(64),

    prompt_tokens INT DEFAULT 0,
    completion_tokens INT DEFAULT 0,
    total_tokens INT DEFAULT 0,

    tool_call_count INT DEFAULT 0,
    memory_recall_count INT DEFAULT 0,
    context_compaction_count INT DEFAULT 0,

    duration_ms BIGINT,
    estimated_cost NUMERIC(18, 6),

    status VARCHAR(32) NOT NULL,
    error_code VARCHAR(128),
    error_message TEXT,

    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## 11.3 workflow_step_metric 汇总表

```sql
CREATE TABLE workflow_step_metric (
    id BIGSERIAL PRIMARY KEY,

    workflow_run_id VARCHAR(64) NOT NULL,
    step_run_id VARCHAR(64) NOT NULL UNIQUE,
    step_name VARCHAR(128) NOT NULL,
    step_type VARCHAR(64) NOT NULL,

    input_size INT,
    output_size INT,
    duration_ms BIGINT,

    status VARCHAR(32) NOT NULL,
    error_message TEXT,

    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

# 12. 后端 C# 代码设计

## 12.1 TrackEventRequest

```csharp
public sealed class TrackEventRequest
{
    public string EventName { get; set; } = default!;
    public string EventType { get; set; } = default!;
    public string SchemaVersion { get; set; } = "1.0";

    public string? PageUrl { get; set; }
    public string? Referrer { get; set; }

    public string? TaskId { get; set; }
    public string? WorkflowRunId { get; set; }
    public string? AgentRunId { get; set; }
    public string? ReviewId { get; set; }

    public string? ExperimentId { get; set; }
    public string? VariantId { get; set; }

    public Dictionary<string, object?> Properties { get; set; } = new();

    public DateTimeOffset? ClientOccurredAt { get; set; }
}
```

## 12.2 AnalyticsEvent 实体

```csharp
public sealed class AnalyticsEvent
{
    public long Id { get; set; }

    public string EventId { get; set; } = Guid.NewGuid().ToString("N");
    public string EventName { get; set; } = default!;
    public string EventType { get; set; } = default!;
    public string SchemaVersion { get; set; } = "1.0";
    public string? EventSource { get; set; }

    public string? UserId { get; set; }
    public string? UserPseudoId { get; set; }
    public string? TenantId { get; set; }
    public string? SessionId { get; set; }

    public string? RequestId { get; set; }
    public string? TraceId { get; set; }
    public string? SpanId { get; set; }

    public string? PageUrl { get; set; }
    public string? Referrer { get; set; }
    public string? UserAgent { get; set; }
    public string? IpHash { get; set; }

    public string? TaskId { get; set; }
    public string? WorkflowRunId { get; set; }
    public string? AgentRunId { get; set; }
    public string? ReviewId { get; set; }

    public string? ExperimentId { get; set; }
    public string? VariantId { get; set; }

    public string? PromptVersion { get; set; }
    public string? ModelVersion { get; set; }
    public string? ToolPolicyVersion { get; set; }
    public string? ApprovalPolicyVersion { get; set; }
    public string? RagIndexVersion { get; set; }
    public string? RetrieverVersion { get; set; }
    public string? RerankerVersion { get; set; }

    public bool IsSampled { get; set; }
    public decimal? SamplingRate { get; set; }
    public string? PiiLevel { get; set; }

    public string PropertiesJson { get; set; } = "{}";

    public DateTimeOffset? ClientOccurredAt { get; set; }
    public DateTimeOffset ServerReceivedAt { get; set; } = DateTimeOffset.UtcNow;
    public DateTimeOffset? ProcessedAt { get; set; }

    public DateTimeOffset CreatedAt { get; set; } = DateTimeOffset.UtcNow;
}
```

## 12.3 ITrackingService

```csharp
public interface ITrackingService
{
    Task TrackAsync(
        TrackEventRequest request,
        CancellationToken cancellationToken = default);
}
```

## 12.4 TrackingService

```csharp
public sealed class TrackingService : ITrackingService
{
    private readonly AppDbContext _dbContext;
    private readonly IHttpContextAccessor _httpContextAccessor;
    private readonly ILogger<TrackingService> _logger;

    public TrackingService(
        AppDbContext dbContext,
        IHttpContextAccessor httpContextAccessor,
        ILogger<TrackingService> logger)
    {
        _dbContext = dbContext;
        _httpContextAccessor = httpContextAccessor;
        _logger = logger;
    }

    public async Task TrackAsync(
        TrackEventRequest request,
        CancellationToken cancellationToken = default)
    {
        var http = _httpContextAccessor.HttpContext;
        var activity = System.Diagnostics.Activity.Current;

        var entity = new AnalyticsEvent
        {
            EventName = request.EventName,
            EventType = request.EventType,
            SchemaVersion = request.SchemaVersion,

            UserId = http?.User?.FindFirst("sub")?.Value,
            TenantId = http?.User?.FindFirst("tenant_id")?.Value,
            SessionId = http?.Request.Headers["X-Session-Id"].FirstOrDefault(),

            RequestId = http?.TraceIdentifier,
            TraceId = activity?.TraceId.ToString(),
            SpanId = activity?.SpanId.ToString(),

            PageUrl = request.PageUrl,
            Referrer = request.Referrer,
            UserAgent = http?.Request.Headers.UserAgent.ToString(),

            TaskId = request.TaskId,
            WorkflowRunId = request.WorkflowRunId,
            AgentRunId = request.AgentRunId,
            ReviewId = request.ReviewId,

            ExperimentId = request.ExperimentId,
            VariantId = request.VariantId,

            PropertiesJson = JsonSerializer.Serialize(request.Properties),

            ClientOccurredAt = request.ClientOccurredAt,
            ServerReceivedAt = DateTimeOffset.UtcNow
        };

        _dbContext.AnalyticsEvents.Add(entity);

        try
        {
            await _dbContext.SaveChangesAsync(cancellationToken);
        }
        catch (Exception ex)
        {
            // 埋点失败不能影响主流程
            _logger.LogError(ex, "Failed to save analytics event: {EventName}", request.EventName);
        }
    }
}
```

## 12.5 TrackController

```csharp
[ApiController]
[Route("api/track")]
public sealed class TrackController : ControllerBase
{
    private readonly ITrackingService _trackingService;

    public TrackController(ITrackingService trackingService)
    {
        _trackingService = trackingService;
    }

    [HttpPost("events")]
    public async Task<IActionResult> Track(
        [FromBody] TrackEventRequest request,
        CancellationToken cancellationToken)
    {
        await _trackingService.TrackAsync(request, cancellationToken);
        return Ok(new { success = true });
    }
}
```

---

# 13. API 设计

## 13.1 上报事件

```http
POST /api/track/events
```

Request:

```json
{
  "eventName": "sql_optimize_submit_click",
  "eventType": "frontend",
  "schemaVersion": "1.0",
  "pageUrl": "/sql-optimize",
  "taskId": "task_001",
  "properties": {
    "databaseType": "postgresql",
    "inputType": "manual_sql"
  },
  "clientOccurredAt": "2026-04-26T10:00:00Z"
}
```

Response:

```json
{
  "success": true
}
```

---

# 14. OpenTelemetry 如何结合

OpenTelemetry 不替代业务埋点。

推荐做法：

```text
OpenTelemetry 负责 trace / metric / log
业务埋点负责 event
event 中写入 trace_id / span_id
```

## 14.1 典型链路

```text
sql_optimize_submit_click
        ↓
optimization_task_created
        ↓
workflow_run_started
        ↓
workflow_step_completed
        ↓
agent_tool_called
        ↓
review_required
        ↓
review_approved
        ↓
optimization_suggestion_accepted
```

这些事件共同关联：

```text
trace_id
request_id
session_id
task_id
workflow_run_id
agent_run_id
review_id
```

这样可以从业务事件反查链路：

```text
用户点击 → 后端创建任务 → Workflow 执行 → Agent 调用工具 → 人工审核 → 用户采纳建议
```

---

# 15. 第一版必须埋的事件

第一版不要贪多，先把核心闭环打通。

```text
page_view
sql_optimize_submit_click
db_optimize_submit_click
optimization_task_created
workflow_run_started
workflow_step_completed
agent_run_started
agent_tool_called
agent_run_completed
review_required
review_approved
review_rejected
optimization_suggestion_accepted
optimization_suggestion_rejected
```

后续再加：

```text
copy_result
export_report
context_compacted
memory_recalled
model_retry
tool_approval_required
workflow_resumed
checkpoint_saved
```

---

# 16. 初学者最容易遗漏的生产级问题

---

## 16.1 事件模型不是随便起名

错误：

```text
sql_submit
submit_sql
sql_optimize_submit
optimize_sql_click
```

正确：建立统一事件字典。

推荐命名：

```text
领域.对象.动作
```

例如：

```text
sql.optimize.submit
sql.optimize.success
sql.optimize.fail
review.requested
review.approved
agent.tool.called
workflow.step.completed
```

每个事件必须定义：

| 字段 | 示例 |
|---|---|
| event_name | sql.optimize.submit |
| schema_version | 1.0 |
| owner_team | ai-platform |
| source | frontend / backend / agent |
| required_fields | task_id, user_id |
| pii_level | none / masked / restricted |

---

## 16.2 Schema Version

未来字段一定会变化。

v1：

```json
{
  "databaseType": "pgsql"
}
```

v2：

```json
{
  "databaseType": "postgresql",
  "dbVersion": "16"
}
```

如果没有 schema_version，老报表、新报表、ETL 都会混乱。

正确：

```json
{
  "eventName": "sql.optimize.submit",
  "schemaVersion": "2.0"
}
```

---

## 16.3 身份统一 Identity Resolution

新手常漏的问题：

```text
前端有 anonymousId
登录后有 userId
后端有 sessionId
Workflow 有 workflowRunId
Agent 有 agentRunId
```

如果不统一，一个用户会被统计成多个人。

推荐身份体系：

```text
user_pseudo_id   设备或匿名用户
user_id          登录用户
session_id       一次访问会话
request_id       一次请求
trace_id         链路追踪
task_id          业务任务
workflow_run_id  工作流运行
agent_run_id     Agent 运行
review_id        审核单
```

---

## 16.4 漏斗 Funnel 设计

不要只埋事件，要设计漏斗。

SQL 优化系统示例：

```text
进入页面
↓
输入 SQL
↓
点击提交
↓
任务创建
↓
Agent 分析完成
↓
人工审核
↓
用户采纳
↓
优化上线
```

关键转化率：

```text
提交率 = submit / page_view
Agent 成功率 = agent_success / submit
审核通过率 = approved / review_requested
采纳率 = accepted / approved
上线率 = applied / accepted
回滚率 = rollback / applied
```

---

## 16.5 北极星指标

你的系统推荐的北极星指标：

```text
每周成功被采纳并上线的优化建议数
```

辅助指标：

| 指标 | 含义 |
|---|---|
| Time To First Suggestion | 从提交到首次建议的时间 |
| Suggestion Acceptance Rate | 建议采纳率 |
| Human Override Rate | 人工推翻 AI 的比例 |
| Avg Tokens / Task | 每个任务平均 Token 成本 |
| Avg Cost Saved | 平均节省 SQL 耗时或资源成本 |
| Rollback Rate | 优化上线后回滚率 |

---

## 16.6 数据质量

常见问题：

```text
按钮双击导致重复 submit
页面刷新导致重复 page_view
网络重试导致重复事件
客户端时间错误
```

解决方案：

```text
event_id 唯一
数据库 UNIQUE(event_id)
客户端 retry queue
服务端去重窗口
client_occurred_at + server_received_at
```

---

## 16.7 时序问题

不要只信客户端时间。

推荐保留三个时间：

```text
client_occurred_at    客户端发生时间
server_received_at    服务端接收时间
processed_at          后续消费处理时间
```

---

## 16.8 采样 Sampling

不是所有事件都要全量保存。

全量保存：

```text
支付
审核
错误
安全事件
Agent 最终结果
Workflow 状态变化
```

可以抽样：

```text
page_view
debug_event
token_stream
mousemove
heartbeat
```

示例：

```json
{
  "eventName": "page_view",
  "isSampled": true,
  "samplingRate": 0.1
}
```

---

## 16.9 PII / 合规

数据库优化系统尤其危险，因为用户可能提交包含敏感信息的 SQL。

错误：

```json
{
  "sql": "select * from users where phone = '138xxxx'"
}
```

正确：

```json
{
  "sqlHash": "a8f31c...",
  "sqlType": "select",
  "tableCount": 3,
  "hasWhere": true,
  "hasJoin": true,
  "sensitiveFieldDetected": true
}
```

SQL 原文应该进入受控业务表，不能随便进入分析事件表。

建议数据分级：

| 等级 | 示例 |
|---|---|
| P0 | 密码、密钥 |
| P1 | 手机、身份证、邮箱 |
| P2 | SQL 原文、数据库连接信息 |
| P3 | 聚合指标、脱敏统计 |

---

## 16.10 死信队列 DLQ

生产上埋点写入会失败。

推荐链路：

```text
Client Queue
↓
Tracking API
↓
MQ
↓
Consumer
↓
DB / ClickHouse
```

失败进入：

```text
Dead Letter Queue
```

DLQ 用于：

```text
排查脏数据
重放失败事件
修复 ETL
避免数据静默丢失
```

---

## 16.11 数据回放 Replay

AI 系统未来一定会做模型升级、Prompt 升级、工具升级。

要能回答：

```text
为什么旧版本 Agent 给出了这个建议？
换新模型后，同样输入会不会更好？
某次事故是不是由某个 prompt 版本造成？
```

因此要保存：

```text
input_snapshot
prompt_version
model_version
tool_version
policy_version
rag_index_version
retriever_version
reranker_version
```

---

## 16.12 Feature Flag / 实验分组

AI 系统一定会做实验，例如：

```text
GPT vs Claude vs DeepSeek
不同 Prompt 版本
不同 RAG chunk 策略
不同审核规则
```

事件里要带：

```json
{
  "experimentId": "sql_optimizer_v3",
  "variant": "claude"
}
```

---

## 16.13 SLO / SLA

埋点不只是产品分析，也服务于运维。

示例 SLO：

```text
95% SQL 优化任务 < 15 秒
99% 审核请求响应 < 2 秒
Agent 工具调用失败率 < 1%
Workflow resume 成功率 > 99%
```

需要支持：

```text
P50
P95
P99
错误率
超时率
成功率
```

---

## 16.14 埋点治理后台

3 个月后，团队会忘记事件含义。

所以需要一个简单的埋点治理后台：

```text
事件字典
字段说明
Owner
版本
弃用状态
测试环境校验
字段必填校验
PII 等级
示例 Payload
```

---

# 17. AI 系统特有埋点补充

传统系统没有这些字段，但 AI 系统必须有。

## 17.1 Prompt Version

```text
prompt_version
system_prompt_hash
developer_prompt_hash
user_prompt_hash
```

用途：

```text
定位某次输出是否由 Prompt 变更引起
对比不同 Prompt 的效果
支持事故复盘
```

---

## 17.2 Model Version

```text
model_provider
model_name
model_version
temperature
max_tokens
```

用途：

```text
成本统计
效果对比
模型迁移评估
```

---

## 17.3 Tool Policy Version

```text
tool_policy_version
approval_policy_version
tool_allowlist_version
tool_risk_level
```

用途：

```text
判断某个危险工具是否被错误放行
审计工具调用策略
支持安全复盘
```

---

## 17.4 RAG / Grounding Version

```text
rag_index_version
retriever_version
reranker_version
embedding_model_version
top_k
similarity_threshold
grounding_score
```

用途：

```text
评估召回效果
定位幻觉问题
判断答案是否被证据支持
```

---

## 17.5 反指标

不能只看成功率，还要看风险指标。

| 指标 | 意义 |
|---|---|
| hallucination_rate | 幻觉率 |
| unsafe_tool_attempt_rate | 危险工具尝试率 |
| human_override_rate | 人工推翻率 |
| rollback_rate | 优化上线后回滚率 |
| grounding_fail_rate | 证据不支持答案的比例 |
| tool_error_rate | 工具失败率 |

---

# 18. 数据库优化系统专属埋点建议

你的 AI 数据库优化系统尤其应该补这些：

```text
SQL 指纹，而不是原 SQL
Explain Plan Hash
优化建议版本
审核规则版本
回滚追踪
成本 / 收益比
Token 成本 vs 性能提升
工具调用风险等级
数据库版本
表数量
索引数量
慢查询来源
优化前后耗时
优化前后 rows examined
优化前后 buffer hit ratio
```

## 18.1 SQL 指纹示例

不要保存：

```sql
SELECT * FROM users WHERE phone = '13812345678'
```

应该保存：

```json
{
  "sqlFingerprint": "select * from users where phone = ?",
  "sqlHash": "xxx",
  "sqlType": "select",
  "tableCount": 1,
  "hasWhere": true,
  "hasJoin": false,
  "hasSubQuery": false
}
```

## 18.2 Explain Plan Hash

```json
{
  "explainPlanHash": "plan_abc123",
  "scanType": "seq_scan",
  "estimatedRows": 1000000,
  "actualRows": 980000
}
```

---

# 19. 常见错误与正确做法

## 19.1 错误：只打日志，不做埋点

错误：

```csharp
_logger.LogInformation("用户点击了优化按钮");
```

问题：

```text
日志字段不稳定
日志难以做 BI 分析
日志通常有采集和保留周期限制
日志不是业务事实表
```

正确：

```csharp
await _trackingService.TrackAsync(new TrackEventRequest
{
    EventName = "sql_optimize_submit_click",
    EventType = "frontend",
    Properties = new()
    {
        ["databaseType"] = "postgresql"
    }
});
```

---

## 19.2 错误：把敏感信息直接写入埋点

错误：

```json
{
  "sql": "select * from users where phone = '138xxxx'"
}
```

正确：

```json
{
  "sqlHash": "a8f31c...",
  "sqlType": "select",
  "tableCount": 3,
  "hasWhere": true,
  "hasJoin": true
}
```

---

## 19.3 错误：事件名随便起

错误：

```text
click1
button_click_test
user_do_sql
```

正确：

```text
sql.optimize.submit
review.approved
agent.tool.called
workflow.checkpoint.saved
```

---

## 19.4 错误：埋点影响主流程

错误：

```text
埋点失败 → 用户提交失败
```

正确：

```text
业务成功优先
埋点失败只记录日志
必要时进入本地队列或 MQ
```

---

# 20. 组织分工

埋点不是纯后端任务。

推荐 RACI：

| 角色 | 责任 |
|---|---|
| PM | 定义业务事件、漏斗、北极星指标 |
| FE | 前端行为埋点 |
| BE | 后端业务事件、接口事件 |
| AI Engineer | Agent、Prompt、Tool、RAG、Eval 埋点 |
| Data Engineer | 数仓、ETL、BI、数据质量 |
| Security | PII、审计、合规 |
| QA | 埋点测试、事件校验 |

---

# 21. 推荐最终方案

你的项目建议采用：

```text
前端行为埋点
+ 后端业务埋点
+ Agent / Workflow 埋点
+ OpenTelemetry Trace 关联
+ 审核与安全事件
+ 业务效果闭环
```

## 21.1 第一阶段

```text
PostgreSQL 存 analytics_event
前端封装 track()
后端封装 TrackingService
Agent / Workflow 手动接入关键事件
trace_id 写入 event
```

## 21.2 第二阶段

```text
MQ 异步写入
ClickHouse 做高频分析
埋点治理后台
事件字典
Feature Flag / Experiment
DLQ
BI 看板
```

## 21.3 第三阶段

```text
AI 效果评估
Prompt / Model / Tool 版本对比
Grounding 检查
自动生成周报
异常埋点告警
成本收益分析
```

---

# 22. 生产级 Checklist

## 22.1 基础能力

```text
[ ] 统一事件命名规范
[ ] 事件字典
[ ] schema_version
[ ] event_id 幂等
[ ] user_id / anonymous_id / session_id
[ ] trace_id / span_id
[ ] task_id / workflow_run_id / agent_run_id
[ ] client_occurred_at / server_received_at
```

## 22.2 数据质量

```text
[ ] 重复事件去重
[ ] 客户端 retry queue
[ ] 服务端校验必填字段
[ ] 事件字段类型校验
[ ] 埋点失败不影响主流程
[ ] DLQ
[ ] 数据回放能力
```

## 22.3 安全合规

```text
[ ] SQL 原文不进入 analytics_event
[ ] IP hash
[ ] PII 分级
[ ] 字段脱敏
[ ] 高危事件审计
[ ] 数据保留周期
[ ] 权限控制
```

## 22.4 AI 特有能力

```text
[ ] prompt_version
[ ] model_version
[ ] tool_policy_version
[ ] approval_policy_version
[ ] rag_index_version
[ ] retriever_version
[ ] reranker_version
[ ] token 成本
[ ] 工具调用次数
[ ] grounding_score
[ ] hallucination / override / rollback 反指标
```

## 22.5 产品分析

```text
[ ] 漏斗设计
[ ] 北极星指标
[ ] 采纳率
[ ] 审核通过率
[ ] 回滚率
[ ] Time To First Suggestion
[ ] 成本 / 收益比
```

---

# 23. 最终结论

初学者通常以为：

```text
埋点 = 记录用户点击
```

生产级系统应该理解为：

```text
埋点 = 定义业务事实 + 用户行为 + AI 决策过程 + 效果评估 + 成本分析 + 风险审计 + 可观测性关联
```

对于你的 AI 数据库优化系统，最关键的是：

```text
不要只记录“Agent 执行了”
要记录：
为什么执行
用了哪个 Prompt
用了哪个模型
调用了哪个工具
是否经过审核
用户是否采纳
上线后是否真的变快
是否发生回滚
这次收益是否大于 AI 成本
```

只有做到这些，埋点才不只是技术记录，而是能支撑产品迭代、AI 评估、面试表达和生产运维的核心数据系统。
