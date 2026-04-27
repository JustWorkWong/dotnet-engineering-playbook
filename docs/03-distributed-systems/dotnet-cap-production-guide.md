# DotNetCore.CAP 完整学习与生产实践文档

> 版本：v1.0  
> 整理时间：2026-04-26  
> 适用对象：.NET / C# 后端开发、微服务开发、面试准备、生产系统设计  
> 关键词：DotNetCore.CAP、MQ、Outbox、EventBus、最终一致性、幂等、顺序性、补偿、重试、可观测性

---

## 目录

1. [CAP 是什么](#1-cap-是什么)
2. [CAP 解决的核心问题](#2-cap-解决的核心问题)
3. [CAP 和直接使用 MQ 的区别](#3-cap-和直接使用-mq-的区别)
4. [CAP 的核心机制：Outbox 本地消息表](#4-cap-的核心机制outbox-本地消息表)
5. [CAP 的最小落地示例](#5-cap-的最小落地示例)
6. [发布事件：业务数据和消息同事务](#6-发布事件业务数据和消息同事务)
7. [订阅事件：消费者如何处理消息](#7-订阅事件消费者如何处理消息)
8. [CAP 不是什么](#8-cap-不是什么)
9. [CAP 的主要问题和生产坑](#9-cap-的主要问题和生产坑)
10. [初次接触 CAP 容易遗漏的关键点](#10-初次接触-cap-容易遗漏的关键点)
11. [生产级事件设计规范](#11-生产级事件设计规范)
12. [幂等设计方案](#12-幂等设计方案)
13. [顺序性设计方案](#13-顺序性设计方案)
14. [重试、毒消息、死信和补偿](#14-重试毒消息死信和补偿)
15. [Consumer Group 与 Fan-out](#15-consumer-group-与-fan-out)
16. [CAP 与 Saga / Workflow / 状态机的关系](#16-cap-与-saga--workflow--状态机的关系)
17. [数据库表、监控与运维治理](#17-数据库表监控与运维治理)
18. [OpenTelemetry 与可观测性](#18-opentelemetry-与可观测性)
19. [前端状态设计](#19-前端状态设计)
20. [完整业务案例：订单创建与库存扣减](#20-完整业务案例订单创建与库存扣减)
21. [完整业务案例：AI 分析完成后进入人工审核](#21-完整业务案例ai-分析完成后进入人工审核)
22. [生产级 CAP 配置建议](#22-生产级-cap-配置建议)
23. [CAP 设计检查清单](#23-cap-设计检查清单)
24. [面试回答模板](#24-面试回答模板)
25. [参考资料](#25-参考资料)

---

# 1. CAP 是什么

DotNetCore.CAP 是 .NET 生态中的一个 **事件总线 + 分布式事务最终一致性框架**。

它不是 MQ 本身，而是站在 RabbitMQ、Kafka、Redis Streams、NATS、Pulsar、Azure Service Bus、Amazon SQS 等消息队列之上的一层可靠消息框架。

可以这样理解：

```text
直接用 MQ：
    你自己负责发消息、处理失败、处理重试、处理幂等、处理消息表。

使用 CAP：
    CAP 帮你封装 EventBus、Outbox 本地消息表、消息重试、消息状态、Dashboard 等能力。
```

一句话总结：

> **MQ 负责传输消息，CAP 负责把“业务数据库变更”和“消息发布”可靠地绑定在一起。**

---

# 2. CAP 解决的核心问题

## 2.1 普通 MQ 代码的问题

假设订单服务创建订单后，需要通知库存服务、积分服务、通知服务。

普通写法：

```csharp
await db.Orders.AddAsync(order);
await db.SaveChangesAsync();

await rabbitMq.PublishAsync("order.created", new OrderCreatedEvent(order.Id));
```

看起来没问题，但生产上会出两个经典事故。

---

## 2.2 问题一：数据库成功，MQ 发送失败

```text
订单写入成功
↓
服务宕机 / 网络抖动 / MQ 不可用
↓
消息没有发出去
↓
库存服务、积分服务、通知服务都不知道订单创建了
```

结果：

```text
数据库里有订单
但是下游系统没有收到事件
系统状态不一致
```

---

## 2.3 问题二：MQ 发送成功，数据库提交失败

如果你把顺序反过来：

```csharp
await rabbitMq.PublishAsync("order.created", eventData);

await db.Orders.AddAsync(order);
await db.SaveChangesAsync();
```

又可能出现：

```text
消息已经发出去了
↓
数据库保存失败
↓
下游以为订单创建了
↓
但订单库里没有这个订单
```

---

## 2.4 本质问题

本质是：

```text
业务数据库事务
和
MQ 消息发送事务

不是同一个事务。
```

普通 MQ 无法天然保证：

```text
业务数据提交成功
和
消息一定成功发出
```

CAP 要解决的就是这个问题。

---

# 3. CAP 和直接使用 MQ 的区别

## 3.1 总体对比

| 对比项 | 直接使用 MQ | 使用 CAP |
|---|---|---|
| 定位 | 消息队列客户端调用 | EventBus + Outbox + 最终一致性框架 |
| 是否替代 MQ | 直接操作 MQ | 不替代 MQ，而是封装 MQ |
| 数据库与消息一致性 | 需要自己处理 | CAP 内置本地消息表模式 |
| 消息丢失防护 | 需要自己设计 | 消息先落库，再投递 MQ |
| 重试机制 | 自己实现 | CAP 内置失败重试 |
| 订阅模型 | 自己绑定 exchange / queue / topic | `[CapSubscribe]` 声明订阅 |
| 消息状态表 | 自己建表 | CAP 内置 Published / Received 表 |
| Dashboard | 自己开发 | CAP 提供 Dashboard |
| 适合场景 | 简单异步任务、日志、低一致性通知 | 订单、支付、库存、审批、工单等可靠业务事件 |
| 复杂度 | MQ 细节多，可靠性自己做 | 业务代码更简单，但要理解 CAP 的最终一致性模型 |

---

## 3.2 直接用 MQ 的问题

普通 MQ 写法：

```csharp
await _db.Orders.AddAsync(order);
await _db.SaveChangesAsync();

await _rabbitMq.PublishAsync("order.created", new OrderCreatedEvent
{
    OrderId = order.Id
});
```

问题：

```text
SaveChangesAsync 成功后，PublishAsync 失败怎么办？
```

你需要自己补齐：

```text
1. 建 outbox_message 表
2. 写业务数据时同时写 outbox_message
3. 后台任务扫描 outbox_message
4. 发送 MQ
5. 成功后标记已发送
6. 失败后重试
7. 超过次数后进入死信或人工处理
8. 消费端做幂等
9. 做监控和告警
```

CAP 的价值，就是把这些通用能力中的大部分封装起来。

---

# 4. CAP 的核心机制：Outbox 本地消息表

CAP 的核心是 **Outbox Pattern**，也就是本地消息表模式。

核心流程：

```text
begin transaction

1. 写业务表 Orders
2. 写 CAP 本地消息表 Published

commit transaction

3. CAP 后台任务扫描 Published
4. 发送到 RabbitMQ / Kafka / Redis Streams
5. 发送成功后更新消息状态
6. 发送失败则重试
```

CAP 还会维护接收消息的记录，通常包括：

```text
Published：待发布 / 已发布 / 发布失败的消息
Received：已接收 / 已消费 / 消费失败的消息
Lock：多实例重试时的存储锁，开启 UseStorageLock 后会使用
```

注意：

```text
具体表名、字段、schema 会随 CAP 版本和数据库配置变化。
不要在业务代码里强依赖 CAP 内部表结构。
```

---

# 5. CAP 的最小落地示例

下面以：

```text
ASP.NET Core
EF Core
PostgreSQL
RabbitMQ
DotNetCore.CAP
```

为例。

## 5.1 安装包

```bash
dotnet add package DotNetCore.CAP
dotnet add package DotNetCore.CAP.EntityFrameworkCore
dotnet add package DotNetCore.CAP.PostgreSql
dotnet add package DotNetCore.CAP.RabbitMQ
dotnet add package DotNetCore.CAP.Dashboard
```

如果要接 OpenTelemetry：

```bash
dotnet add package DotNetCore.CAP.OpenTelemetry
```

---

## 5.2 Program.cs 配置

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
{
    options.UseNpgsql(builder.Configuration.GetConnectionString("Postgres"));
});

builder.Services.AddCap(options =>
{
    // 使用 EF Core 事务集成
    options.UseEntityFramework<AppDbContext>();

    // 使用 PostgreSQL 作为 CAP 消息存储
    options.UsePostgreSql(builder.Configuration.GetConnectionString("Postgres"));

    // 使用 RabbitMQ 作为消息传输
    options.UseRabbitMQ(rabbit =>
    {
        rabbit.HostName = builder.Configuration["RabbitMQ:HostName"];
        rabbit.UserName = builder.Configuration["RabbitMQ:UserName"];
        rabbit.Password = builder.Configuration["RabbitMQ:Password"];
        rabbit.Port = int.Parse(builder.Configuration["RabbitMQ:Port"] ?? "5672");
    });

    // Dashboard
    options.UseDashboard();

    // 生产建议根据业务调整
    options.FailedRetryCount = 10;
    options.FailedRetryInterval = 60;
    options.UseStorageLock = true;
    options.SucceedMessageExpiredAfter = 3600;
    options.FailedMessageExpiredAfter = 15 * 24 * 3600;
    options.CollectorCleaningInterval = 300;
});
```

---

# 6. 发布事件：业务数据和消息同事务

## 6.1 正确写法

关键点：

```text
业务数据写入
和
CAP 消息写入

必须在同一个本地数据库事务里。
```

示例：

```csharp
[ApiController]
[Route("api/orders")]
public class OrderController : ControllerBase
{
    private readonly AppDbContext _db;
    private readonly ICapPublisher _capPublisher;

    public OrderController(AppDbContext db, ICapPublisher capPublisher)
    {
        _db = db;
        _capPublisher = capPublisher;
    }

    [HttpPost]
    public async Task<IActionResult> CreateOrder(CreateOrderRequest request)
    {
        await using var transaction =
            await _db.Database.BeginTransactionAsync(_capPublisher, autoCommit: false);

        var order = new Order
        {
            Id = Guid.NewGuid(),
            UserId = request.UserId,
            Amount = request.Amount,
            Status = OrderStatus.Created,
            CreatedAt = DateTimeOffset.UtcNow
        };

        _db.Orders.Add(order);
        await _db.SaveChangesAsync();

        await _capPublisher.PublishAsync("order.created.v1", new OrderCreatedIntegrationEvent
        {
            EventId = Guid.NewGuid(),
            OrderId = order.Id,
            UserId = order.UserId,
            Amount = order.Amount,
            Currency = "CNY",
            OccurredAt = DateTimeOffset.UtcNow
        });

        await transaction.CommitAsync();

        return Ok(new
        {
            orderId = order.Id,
            status = "Created",
            message = "订单已创建，后续库存、通知等流程异步处理中"
        });
    }
}
```

---

## 6.2 错误写法

```csharp
_db.Orders.Add(order);
await _db.SaveChangesAsync();

// 错误：这时业务事务已经结束
await _capPublisher.PublishAsync("order.created.v1", evt);
```

这种写法不能发挥 CAP 的核心价值。因为业务数据和消息不是同一个本地事务。

---

# 7. 订阅事件：消费者如何处理消息

## 7.1 基础消费者

```csharp
public class OrderCreatedConsumer : ICapSubscribe
{
    private readonly AppDbContext _db;

    public OrderCreatedConsumer(AppDbContext db)
    {
        _db = db;
    }

    [CapSubscribe("order.created.v1")]
    public async Task HandleAsync(OrderCreatedIntegrationEvent evt)
    {
        var log = new OrderEventLog
        {
            Id = Guid.NewGuid(),
            EventId = evt.EventId,
            OrderId = evt.OrderId,
            EventName = "order.created.v1",
            CreatedAt = DateTimeOffset.UtcNow
        };

        _db.OrderEventLogs.Add(log);
        await _db.SaveChangesAsync();
    }
}
```

---

## 7.2 消费者不要吞异常

错误写法：

```csharp
[CapSubscribe("order.created.v1")]
public async Task HandleAsync(OrderCreatedIntegrationEvent evt)
{
    try
    {
        await _inventoryService.ReserveAsync(evt.OrderId);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "库存预占失败");
        // 错误：异常被吞掉，CAP 会认为消费成功
    }
}
```

正确做法：

```csharp
[CapSubscribe("order.created.v1")]
public async Task HandleAsync(OrderCreatedIntegrationEvent evt)
{
    try
    {
        await _inventoryService.ReserveAsync(evt.OrderId);
    }
    catch (TimeoutException ex)
    {
        _logger.LogWarning(ex, "库存服务临时超时，交给 CAP 重试");
        throw;
    }
}
```

原则：

```text
临时失败：throw，让 CAP 重试。
永久失败：记录业务失败，不要无限重试。
异常被吞：CAP 会认为消费成功。
```

---

# 8. CAP 不是什么

## 8.1 CAP 不是强一致性分布式事务

CAP 不等于：

```text
订单服务成功
库存服务成功
支付服务成功
积分服务成功

所有服务同时成功或同时失败
```

CAP 是：

```text
当前服务本地事务成功
消息可靠发布
下游最终消费
跨服务最终一致
```

---

## 8.2 CAP 不是 2PC / DTC

CAP 不提供开箱即用的 MS DTC 或 Two-Phase Commit 强分布式事务。它采用的是：

```text
最终一致性
+
补偿
+
可靠消息
```

---

## 8.3 CAP 不是 Saga / Workflow

CAP 能做：

```text
可靠发布事件
可靠消费事件
失败重试
消息状态记录
服务解耦
```

CAP 不能直接做：

```text
复杂流程编排
多步骤状态机
人工审核等待
超时取消
补偿步骤编排
流程恢复
流程可视化
```

复杂业务流程需要：

```text
状态机 / Workflow / Saga 编排
+
CAP 可靠事件投递
+
数据库持久化流程状态
```

---

## 8.4 CAP 不是流处理框架

CAP 不适合：

```text
高频日志采集
大数据实时计算
复杂窗口聚合
海量埋点流
复杂 Kafka topic join
```

这些更适合：

```text
Kafka
Flink
Spark Streaming
ClickHouse
日志管道
```

---

# 9. CAP 的主要问题和生产坑

## 9.1 问题一：CAP 不是强一致性

用了 CAP 后，系统会变成：

```text
本地事务强一致
跨服务最终一致
```

这意味着：

```text
订单已创建
但库存可能稍后才扣减
积分可能稍后才到账
通知可能稍后才发送
```

因此业务上必须设计：

```text
中间状态
补偿逻辑
对账逻辑
人工处理
失败状态
```

---

## 9.2 问题二：会重复消费

CAP 采用 At Least Once 投递语义。也就是说：

```text
消息尽量保证不丢
但可能重复投递
```

所以消费者必须幂等。

危险代码：

```csharp
[CapSubscribe("points.add.v1")]
public async Task HandleAsync(PointsAddEvent evt)
{
    var user = await _db.Users.FirstAsync(x => x.Id == evt.UserId);
    user.Points += evt.Points;

    await _db.SaveChangesAsync();
}
```

如果消息重复，积分会重复增加。

---

## 9.3 问题三：顺序不是默认保证的

如果你希望同一个订单严格按顺序：

```text
order.created
order.paid
order.shipped
order.completed
```

不能简单依赖 CAP 默认保证顺序。

尤其当：

```text
ConsumerThreadCount > 1
EnableSubscriberParallelExecute = true
多个服务实例并发消费
```

顺序可能被破坏。

---

## 9.4 问题四：CAP 表会带来数据库压力

CAP 的本地消息表会产生：

```text
消息插入
消息状态更新
消息扫描
失败重试查询
历史消息清理
索引维护
```

高并发场景下可能出现：

```text
Published 表膨胀
Received 表膨胀
索引变大
查询变慢
数据库 CPU 升高
IO 升高
```

---

## 9.5 问题五：失败重试可能形成重试风暴

当下游服务挂掉：

```text
大量消息消费失败
↓
CAP 不断重试
↓
数据库、MQ、下游服务压力持续增加
↓
下游恢复后又被积压消息打爆
```

所以必须设计：

```text
重试上限
重试间隔
熔断
限流
失败告警
人工补偿
死信处理
```

---

## 9.6 问题六：多实例部署要注意并发重试

多个实例：

```text
order-service-1
order-service-2
order-service-3
```

都可能扫描 CAP 表进行重试。

生产建议：

```csharp
options.UseStorageLock = true;
```

否则容易出现多个实例同时捞取失败消息的并发问题。

---

## 9.7 问题七：消费者里直接调用外部系统容易出错

例如：

```csharp
[CapSubscribe("order.paid.v1")]
public async Task HandleAsync(OrderPaidEvent evt)
{
    await _sms.SendAsync(evt.Phone, "支付成功");

    var order = await _db.Orders.FirstAsync(x => x.Id == evt.OrderId);
    order.SmsSent = true;

    await _db.SaveChangesAsync();
}
```

如果短信发送成功，但数据库保存失败，CAP 会重试，用户可能收到多条短信。

更好的方式：

```text
消费者先落本地任务表
后台 Worker 再发送短信
短信任务自己做幂等和重试
```

---

## 9.8 问题八：事件版本长期维护成本高

一开始事件可能很简单：

```csharp
public class OrderCreatedEvent
{
    public Guid OrderId { get; set; }
    public decimal Amount { get; set; }
}
```

后续加字段、改字段、改语义，如果没有版本管理，下游服务很容易挂。

建议一开始命名：

```text
order.created.v1
order.created.v2
payment.succeeded.v1
return.quality_checked.v1
```

---

## 9.9 问题九：Dashboard 不能代替业务后台

CAP Dashboard 主要看技术消息状态：

```text
消息是否成功
是否失败
失败原因
是否可以手动重试
```

但业务后台还需要：

```text
订单号
业务事件 ID
业务状态
失败类型
补偿状态
人工处理记录
对账状态
```

---

# 10. 初次接触 CAP 容易遗漏的关键点

## 10.1 CAP 可靠性的前提是“同一个本地事务”

错误理解：

```text
项目用了 CAP，所以一定可靠。
```

正确理解：

```text
只有业务数据写入和 CAP 消息写入处于同一个本地事务中，CAP 才能解决“业务成功但消息丢失”的问题。
```

---

## 10.2 CAP 不能跨多个数据库做原子事务

不要这样设计：

```text
业务库 A 提交
CAP 消息表在库 B 提交
```

因为这样仍然可能：

```text
业务库 A 成功
CAP 库 B 失败
消息丢失
```

建议：

```text
每个服务拥有自己的业务库
CAP Published / Received 表放在该服务自己的数据库里
服务之间通过 MQ + 事件通信
```

---

## 10.3 要区分 Domain Event 和 Integration Event

### Domain Event

服务内部的领域事实：

```text
OrderCreatedDomainEvent
OrderPaidDomainEvent
```

### Integration Event

跨服务契约：

```text
order.created.v1
order.paid.v1
payment.succeeded.v1
```

CAP 发布的通常应该是 Integration Event，而不是 EF Entity。

---

## 10.4 不要直接发布 EF 实体

错误：

```csharp
await _capPublisher.PublishAsync("order.created", orderEntity);
```

问题：

```text
下游依赖你的数据库模型
字段一改，下游可能挂
实体里可能有敏感字段
序列化可能循环引用
事件契约不可控
```

正确：

```csharp
await _capPublisher.PublishAsync("order.created.v1", new OrderCreatedIntegrationEvent
{
    EventId = Guid.NewGuid(),
    OrderId = order.Id,
    UserId = order.UserId,
    Amount = order.Amount,
    Currency = order.Currency,
    OccurredAt = DateTimeOffset.UtcNow
});
```

---

## 10.5 Consumer Group / Fan-out 配错会出大问题

同一个服务多实例：

```text
inventory-service-1
inventory-service-2
inventory-service-3
```

应该通常属于同一个 group，竞争消费。

不同业务服务：

```text
inventory-service
notification-service
coupon-service
audit-service
```

应该属于不同 group，各自都收到事件。

错误配置会导致：

```text
本来只想处理一次，结果多个实例重复处理
本来希望多个系统都收到，结果只有一个系统收到
```

---

## 10.6 吞异常会让 CAP 误判成功

CAP 判断消费成功的基础是：

```text
消费者方法是否正常返回
```

如果你 catch 了异常但不 throw，CAP 会认为消费成功。

---

## 10.7 重试要区分临时失败和毒消息

### 临时失败

适合重试：

```text
MQ 短暂断开
数据库连接超时
下游 HTTP 502 / 503
网络抖动
```

### 毒消息 / 永久失败

不适合一直重试：

```text
事件格式不兼容
必填字段为空
业务状态不允许
用户不存在
订单不存在
金额非法
```

---

## 10.8 CAP 不是内部方法调用替代品

不要为了“显得微服务化”，把所有内部调用都改成 CAP 消息。

优先级通常是：

```text
同进程同步调用
领域事件内存分发
本地事务
后台任务
CAP / MQ
```

CAP 更适合跨进程、跨服务、需要可靠异步的场景。

---

# 11. 生产级事件设计规范

## 11.1 事件命名

推荐：

```text
order.created.v1
order.paid.v1
order.cancelled.v1
inventory.reserved.v1
payment.succeeded.v1
return.quality_checked.v1
ai.analysis.completed.v1
```

不推荐：

```text
sendMessage
test
order
doSomething
```

---

## 11.2 事件基础字段

建议所有集成事件继承基础类：

```csharp
public abstract class IntegrationEventBase
{
    public Guid EventId { get; init; } = Guid.NewGuid();

    public string TraceId { get; init; } = default!;

    public string? CorrelationId { get; init; }

    public string? CausationId { get; init; }

    public DateTimeOffset OccurredAt { get; init; } = DateTimeOffset.UtcNow;
}
```

字段含义：

| 字段 | 含义 |
|---|---|
| EventId | 当前事件唯一 ID，用于幂等 |
| TraceId | 分布式链路追踪 ID |
| CorrelationId | 同一个业务流程的关联 ID |
| CausationId | 当前事件由哪个事件或命令触发 |
| OccurredAt | 事件发生时间 |

---

## 11.3 事件内容不要太大

推荐事件里放：

```text
业务 ID
必要快照
状态变化
发生时间
追踪信息
```

不推荐放：

```text
完整 EF 实体
完整用户信息
大文本
图片二进制
完整 AI 分析过程
敏感信息
```

示例：

```csharp
public sealed class ReturnQualityCheckedEvent : IntegrationEventBase
{
    public Guid ReturnOrderId { get; init; }

    public Guid ProductId { get; init; }

    public string QualityResult { get; init; } = default!;

    public DateTimeOffset CheckedAt { get; init; }
}
```

---

## 11.4 事件版本治理

原则：

```text
只增字段，少改字段
不要删除字段
不要改变字段语义
新增版本时保留旧 topic 一段时间
不要把数据库实体作为事件契约
```

示例：

```csharp
public sealed class OrderCreatedV1 : IntegrationEventBase
{
    public Guid OrderId { get; init; }
    public decimal Amount { get; init; }
}

public sealed class OrderCreatedV2 : IntegrationEventBase
{
    public Guid OrderId { get; init; }
    public decimal Amount { get; init; }
    public string Currency { get; init; } = "CNY";
}
```

---

# 12. 幂等设计方案

## 12.1 为什么必须幂等

CAP 是 At Least Once 投递：

```text
消息不会轻易丢
但可能重复到达
```

所以所有关键消费者都要幂等。

---

## 12.2 幂等表方案

表结构：

```sql
CREATE TABLE processed_messages (
    id uuid PRIMARY KEY,
    event_id uuid NOT NULL,
    consumer varchar(100) NOT NULL,
    processed_at timestamptz NOT NULL
);

CREATE UNIQUE INDEX ux_processed_messages_event_consumer
ON processed_messages(event_id, consumer);
```

消费者：

```csharp
[CapSubscribe("points.add.v1")]
public async Task HandleAsync(PointsAddEvent evt)
{
    var consumerName = nameof(PointsAddConsumer);

    var exists = await _db.ProcessedMessages
        .AnyAsync(x => x.EventId == evt.EventId && x.Consumer == consumerName);

    if (exists)
    {
        return;
    }

    var user = await _db.Users.FirstAsync(x => x.Id == evt.UserId);
    user.Points += evt.Points;

    _db.ProcessedMessages.Add(new ProcessedMessage
    {
        Id = Guid.NewGuid(),
        EventId = evt.EventId,
        Consumer = consumerName,
        ProcessedAt = DateTimeOffset.UtcNow
    });

    await _db.SaveChangesAsync();
}
```

---

## 12.3 状态机天然幂等

如果业务操作是设置状态，而不是累加，就更容易幂等。

例如：

```csharp
order.MarkAsPaid(evt.PaymentId);
```

重复执行时，只要状态已经是 Paid，就直接返回。

```csharp
public void MarkAsPaid(Guid paymentId)
{
    if (Status == OrderStatus.Paid)
    {
        return;
    }

    if (Status != OrderStatus.Created)
    {
        throw new InvalidOperationException($"当前状态 {Status} 不允许支付成功");
    }

    Status = OrderStatus.Paid;
    PaymentId = paymentId;
    PaidAt = DateTimeOffset.UtcNow;
}
```

---

## 12.4 数据库唯一索引幂等

例如支付记录：

```sql
CREATE UNIQUE INDEX ux_payment_order_id
ON payments(order_id);
```

这样即使重复消息进来，数据库也能防止重复插入。

---

# 13. 顺序性设计方案

## 13.1 不要默认依赖 CAP 保证业务顺序

如果 `ConsumerThreadCount > 1`，消息执行顺序不能保证。

所以关键业务需要自己控制。

---

## 13.2 状态机防乱序

示例：

```csharp
[CapSubscribe("order.status.changed.v1")]
public async Task HandleAsync(OrderStatusChangedEvent evt)
{
    var order = await _db.Orders.FirstAsync(x => x.Id == evt.OrderId);

    if (evt.Version <= order.Version)
    {
        return; // 旧消息或重复消息
    }

    if (!CanTransit(order.Status, evt.NewStatus))
    {
        throw new InvalidOperationException(
            $"非法状态流转：{order.Status} -> {evt.NewStatus}");
    }

    order.Status = evt.NewStatus;
    order.Version = evt.Version;

    await _db.SaveChangesAsync();
}
```

---

## 13.3 业务版本号

事件中带版本号：

```csharp
public sealed class OrderStatusChangedEvent : IntegrationEventBase
{
    public Guid OrderId { get; init; }
    public int Version { get; init; }
    public string OldStatus { get; init; } = default!;
    public string NewStatus { get; init; } = default!;
}
```

消费端只处理比当前版本新的事件。

---

## 13.4 Kafka 场景的业务 Key

如果使用 Kafka，要考虑同一个业务 key 进入同一个 partition：

```csharp
var headers = new Dictionary<string, string?>
{
    ["cap-kafka-key"] = order.Id.ToString()
};

await _capPublisher.PublishAsync("order.status.changed.v1", evt, headers);
```

这样有利于同一个 orderId 的事件按分区顺序处理。

---

# 14. 重试、毒消息、死信和补偿

## 14.1 CAP 的重试适合临时失败

适合重试：

```text
网络超时
MQ 暂时不可用
数据库连接池暂时耗尽
下游服务临时不可用
```

消费者可以 throw：

```csharp
catch (TimeoutException)
{
    throw;
}
```

---

## 14.2 永久失败不应该无限重试

例如：

```text
订单不存在
金额非法
事件字段缺失
状态不允许
版本不兼容
```

应该记录业务失败：

```csharp
[CapSubscribe("payment.succeeded.v1")]
public async Task HandleAsync(PaymentSucceededEvent evt)
{
    if (evt.Amount <= 0)
    {
        await _deadLetterService.SaveAsync(evt, "金额非法");
        return;
    }

    var order = await _db.Orders.FindAsync(evt.OrderId);
    if (order is null)
    {
        await _deadLetterService.SaveAsync(evt, "订单不存在");
        return;
    }

    await _orderService.MarkPaidAsync(evt);
}
```

---

## 14.3 业务死信表

```sql
CREATE TABLE business_dead_letters (
    id uuid PRIMARY KEY,
    event_id uuid NOT NULL,
    event_name varchar(200) NOT NULL,
    aggregate_id varchar(100) NULL,
    payload jsonb NOT NULL,
    reason text NOT NULL,
    status varchar(50) NOT NULL,
    created_at timestamptz NOT NULL,
    handled_at timestamptz NULL,
    handled_by varchar(100) NULL,
    handle_comment text NULL
);
```

状态：

```text
Pending
Resolved
Ignored
Retried
```

---

## 14.4 补偿不是简单重试

重试是：

```text
同一个动作再执行一次
```

补偿是：

```text
执行另一个动作，让系统回到业务可接受状态
```

例如：

```text
订单创建成功
库存扣减失败
```

补偿可能是：

```text
取消订单
释放支付
通知用户
人工介入
```

而不是无限重试库存扣减。

---

# 15. Consumer Group 与 Fan-out

## 15.1 同组竞争消费

场景：

```text
inventory-service 部署 3 个实例
```

它们应属于同一个 group：

```text
inventory-service
```

一条消息只由其中一个实例处理。

---

## 15.2 不同组广播消费

场景：

```text
order.created.v1
├── inventory-service
├── notification-service
├── coupon-service
└── audit-service
```

这些服务应该是不同 group，因此每个服务都能收到事件。

---

## 15.3 配置示例

```csharp
public class InventoryConsumer : ICapSubscribe
{
    [CapSubscribe("order.created.v1", Group = "inventory-service")]
    public async Task HandleAsync(OrderCreatedIntegrationEvent evt)
    {
        // 库存处理
    }
}

public class NotificationConsumer : ICapSubscribe
{
    [CapSubscribe("order.created.v1", Group = "notification-service")]
    public async Task HandleAsync(OrderCreatedIntegrationEvent evt)
    {
        // 通知处理
    }
}
```

---

# 16. CAP 与 Saga / Workflow / 状态机的关系

## 16.1 CAP 是事件通道，不是流程大脑

CAP 负责：

```text
事件可靠发布
事件可靠消费
消息失败重试
服务解耦
```

Workflow / Saga / 状态机负责：

```text
流程步骤编排
状态持久化
超时处理
人工审核
补偿决策
流程恢复
```

---

## 16.2 复杂业务不要只靠 topic 串起来

例如退货质检流程：

```text
提交退货
AI 质检
人工审核
入库
重新上架
退款
通知用户
```

不建议只靠一堆 CAP topic 随意串联。

更合理：

```text
ReturnWorkflow 负责流程状态和编排
CAP 负责跨服务事件投递
数据库保存每一步状态
人工审核表保存审核请求
```

---

# 17. 数据库表、监控与运维治理

## 17.1 业务事件日志表

CAP 表是技术表，建议额外建业务事件表：

```sql
CREATE TABLE business_event_logs (
    id uuid PRIMARY KEY,
    event_id uuid NOT NULL,
    event_name varchar(200) NOT NULL,
    aggregate_type varchar(100) NOT NULL,
    aggregate_id varchar(100) NOT NULL,
    status varchar(50) NOT NULL,
    retry_count int NOT NULL DEFAULT 0,
    last_error text NULL,
    created_at timestamptz NOT NULL,
    updated_at timestamptz NOT NULL
);

CREATE UNIQUE INDEX ux_business_event_logs_event_id
ON business_event_logs(event_id);
```

---

## 17.2 需要监控的指标

```text
CAP Published 表数量
CAP Received 表数量
失败消息数量
重试次数
最长积压时间
最早未成功消息时间
消费者平均耗时
消费者失败率
MQ 队列积压
数据库 CPU / IO
```

---

## 17.3 告警规则示例

```text
失败消息数量 > 0 持续 5 分钟：警告
失败消息数量 > 100：严重
最早失败消息超过 30 分钟：严重
消息积压超过 10000：严重
消费者失败率超过 5%：警告
```

---

## 17.4 Dashboard 安全

CAP Dashboard 能看到：

```text
消息体
失败原因
异常信息
手动重试入口
```

生产建议：

```text
只在内网开放
接入认证和权限
限制生产写操作
敏感字段脱敏
异常信息不要泄露密钥、连接串、内部路径
```

---

# 18. OpenTelemetry 与可观测性

## 18.1 接入方式

```bash
dotnet add package DotNetCore.CAP.OpenTelemetry
```

示例：

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddEntityFrameworkCoreInstrumentation()
            .AddCapInstrumentation()
            .AddOtlpExporter();
    });
```

---

## 18.2 为什么需要 TraceId

没有 TraceId 时，排查问题会变成：

```text
用户说订单失败
不知道请求在哪个服务失败
不知道事件有没有发
不知道哪个消费者处理过
不知道重试了几次
```

有 TraceId 后，可以串起来：

```text
HTTP 请求
↓
订单服务 SaveChanges
↓
CAP Published
↓
MQ
↓
库存消费者
↓
库存数据库
↓
通知消费者
```

---

# 19. 前端状态设计

用了 CAP 后，很多业务不是同步完成，而是异步最终一致。

错误返回：

```json
{
  "success": true,
  "message": "全部处理完成"
}
```

更合理：

```json
{
  "requestId": "7d9d7a6b-0b2a-45fa-8ec1-0bdc7788e9a9",
  "businessId": "ORD-20260426-001",
  "status": "Processing",
  "message": "订单已创建，库存与通知处理中"
}
```

前端显示：

```text
订单已创建
库存处理中
通知处理中
积分待发放
```

而不是简单显示“成功”。

---

# 20. 完整业务案例：订单创建与库存扣减

## 20.1 流程

```text
用户创建订单
↓
OrderService 本地事务：
    写 Orders
    写 CAP Published(order.created.v1)
↓
CAP 发布 MQ
↓
InventoryService 消费 order.created.v1
↓
库存预占成功：
    写库存记录
    发布 inventory.reserved.v1
↓
OrderService 消费 inventory.reserved.v1
↓
订单状态变为 InventoryReserved
```

---

## 20.2 订单状态

```csharp
public enum OrderStatus
{
    Created,
    InventoryReserving,
    InventoryReserved,
    InventoryFailed,
    Paid,
    Cancelled
}
```

---

## 20.3 订单创建事件

```csharp
public sealed class OrderCreatedIntegrationEvent : IntegrationEventBase
{
    public Guid OrderId { get; init; }
    public Guid UserId { get; init; }
    public decimal Amount { get; init; }
    public string Currency { get; init; } = "CNY";
}
```

---

## 20.4 库存消费者

```csharp
public class InventoryConsumer : ICapSubscribe
{
    private readonly AppDbContext _db;
    private readonly ICapPublisher _capPublisher;

    public InventoryConsumer(AppDbContext db, ICapPublisher capPublisher)
    {
        _db = db;
        _capPublisher = capPublisher;
    }

    [CapSubscribe("order.created.v1", Group = "inventory-service")]
    public async Task HandleAsync(OrderCreatedIntegrationEvent evt)
    {
        var exists = await _db.ProcessedMessages
            .AnyAsync(x => x.EventId == evt.EventId && x.Consumer == "InventoryConsumer");

        if (exists)
        {
            return;
        }

        await using var tx = await _db.Database.BeginTransactionAsync(_capPublisher, autoCommit: false);

        var reservation = new InventoryReservation
        {
            Id = Guid.NewGuid(),
            OrderId = evt.OrderId,
            Status = "Reserved",
            CreatedAt = DateTimeOffset.UtcNow
        };

        _db.InventoryReservations.Add(reservation);

        _db.ProcessedMessages.Add(new ProcessedMessage
        {
            Id = Guid.NewGuid(),
            EventId = evt.EventId,
            Consumer = "InventoryConsumer",
            ProcessedAt = DateTimeOffset.UtcNow
        });

        await _capPublisher.PublishAsync("inventory.reserved.v1", new InventoryReservedEvent
        {
            EventId = Guid.NewGuid(),
            OrderId = evt.OrderId,
            ReservationId = reservation.Id,
            OccurredAt = DateTimeOffset.UtcNow
        });

        await _db.SaveChangesAsync();
        await tx.CommitAsync();
    }
}
```

---

# 21. 完整业务案例：AI 分析完成后进入人工审核

这个场景贴近 AI 应用。

## 21.1 流程

```text
用户提交 SQL 优化请求
↓
AI 分析服务执行分析
↓
AIAnalysisService 本地事务：
    写 AnalysisResult
    发布 ai.analysis.completed.v1
↓
ReviewService 消费事件
↓
创建人工审核任务
↓
前端显示“待审核”
```

---

## 21.2 事件定义

```csharp
public sealed class AiAnalysisCompletedEvent : IntegrationEventBase
{
    public Guid AnalysisId { get; init; }
    public Guid ProjectId { get; init; }
    public string BusinessType { get; init; } = default!;
    public decimal ConfidenceScore { get; init; }
}
```

---

## 21.3 AI 分析完成后发布事件

```csharp
await using var tx = await _db.Database.BeginTransactionAsync(_capPublisher, autoCommit: false);

var result = new AnalysisResult
{
    Id = Guid.NewGuid(),
    ProjectId = request.ProjectId,
    Content = analysisContent,
    ConfidenceScore = confidence,
    Status = "WaitingReview",
    CreatedAt = DateTimeOffset.UtcNow
};

_db.AnalysisResults.Add(result);
await _db.SaveChangesAsync();

await _capPublisher.PublishAsync("ai.analysis.completed.v1", new AiAnalysisCompletedEvent
{
    EventId = Guid.NewGuid(),
    AnalysisId = result.Id,
    ProjectId = result.ProjectId,
    BusinessType = "SqlOptimization",
    ConfidenceScore = result.ConfidenceScore,
    OccurredAt = DateTimeOffset.UtcNow
});

await tx.CommitAsync();
```

---

## 21.4 审核服务消费事件

```csharp
public class AiAnalysisReviewConsumer : ICapSubscribe
{
    private readonly AppDbContext _db;

    public AiAnalysisReviewConsumer(AppDbContext db)
    {
        _db = db;
    }

    [CapSubscribe("ai.analysis.completed.v1", Group = "review-service")]
    public async Task HandleAsync(AiAnalysisCompletedEvent evt)
    {
        var exists = await _db.ReviewTasks.AnyAsync(x => x.EventId == evt.EventId);
        if (exists)
        {
            return;
        }

        var task = new ReviewTask
        {
            Id = Guid.NewGuid(),
            EventId = evt.EventId,
            BusinessId = evt.AnalysisId,
            BusinessType = evt.BusinessType,
            Status = "Pending",
            CreatedAt = DateTimeOffset.UtcNow
        };

        _db.ReviewTasks.Add(task);
        await _db.SaveChangesAsync();
    }
}
```

---

# 22. 生产级 CAP 配置建议

```csharp
builder.Services.AddCap(options =>
{
    options.UseEntityFramework<AppDbContext>();
    options.UsePostgreSql(builder.Configuration.GetConnectionString("Postgres"));

    options.UseRabbitMQ(rabbit =>
    {
        rabbit.HostName = builder.Configuration["RabbitMQ:HostName"];
        rabbit.UserName = builder.Configuration["RabbitMQ:UserName"];
        rabbit.Password = builder.Configuration["RabbitMQ:Password"];
        rabbit.Port = int.Parse(builder.Configuration["RabbitMQ:Port"] ?? "5672");
    });

    options.UseDashboard();

    // 环境隔离
    options.TopicNamePrefix = "prod.";
    options.GroupNamePrefix = "prod.";
    options.Version = "v1";

    // 多实例下建议开启
    options.UseStorageLock = true;

    // 失败重试
    options.FailedRetryCount = 10;
    options.FailedRetryInterval = 60;

    // 清理策略
    options.SucceedMessageExpiredAfter = 3600;
    options.FailedMessageExpiredAfter = 15 * 24 * 3600;
    options.CollectorCleaningInterval = 300;

    // 顺序敏感场景谨慎调大
    options.ConsumerThreadCount = 1;

    options.FailedThresholdCallback = failed =>
    {
        // 接入日志、告警、飞书、企业微信、Prometheus Alertmanager 等
        Console.WriteLine($"CAP failed: {failed.MessageType}, {failed.Exception}");
    };
});
```

---

# 23. CAP 设计检查清单

## 23.1 事务边界

```text
[ ] CAP Publish 是否和业务 SaveChanges 在同一个本地事务？
[ ] CAP 存储表是否和业务库在同一个事务范围内？
[ ] 是否存在跨多个数据库的伪事务误用？
```

---

## 23.2 事件建模

```text
[ ] 是否区分 Domain Event 和 Integration Event？
[ ] 是否禁止直接发布 EF Entity？
[ ] 事件是否有 EventId / OccurredAt / TraceId？
[ ] topic 是否带业务语义和版本号？
[ ] 是否考虑事件字段兼容性？
```

---

## 23.3 消费端

```text
[ ] 消费者是否幂等？
[ ] 是否有 processed_message 表或业务唯一索引？
[ ] 是否区分临时失败和永久失败？
[ ] 是否避免吞异常导致 CAP 误判成功？
[ ] 是否避免消费者里直接做长耗时任务？
```

---

## 23.4 顺序与并发

```text
[ ] 是否真的需要顺序？
[ ] 如果需要顺序，是否有业务版本号 / 状态机？
[ ] ConsumerThreadCount 是否被谨慎配置？
[ ] 是否了解并行消费会破坏顺序？
```

---

## 23.5 重试与补偿

```text
[ ] FailedRetryCount 是否合理？
[ ] FailedRetryInterval 是否合理？
[ ] 多实例是否开启 UseStorageLock？
[ ] 是否有死信 / 毒消息处理？
[ ] 是否有人工补偿后台？
```

---

## 23.6 存储治理

```text
[ ] CAP 表是否有清理策略？
[ ] 成功消息保留多久？
[ ] 失败消息保留多久？
[ ] CAP 表大小是否监控？
[ ] 是否监控失败消息数量？
```

---

## 23.7 可观测性

```text
[ ] 是否接入日志？
[ ] 是否接入 OpenTelemetry？
[ ] 是否能通过 TraceId 找到完整链路？
[ ] 是否有 CAP Dashboard？
[ ] 是否有业务事件后台？
```

---

## 23.8 安全

```text
[ ] Dashboard 是否有权限控制？
[ ] 消息体是否避免敏感信息？
[ ] 异常信息是否避免泄露密钥？
[ ] 生产环境是否限制手动重试权限？
```

---

## 23.9 架构边界

```text
[ ] 是否把 CAP 当成 MQ 可靠封装，而不是 Workflow？
[ ] 复杂流程是否有状态机 / Workflow？
[ ] 是否避免同进程内部滥用 CAP？
[ ] 是否明确哪些服务订阅哪些事件？
```

---

# 24. 面试回答模板

## 24.1 CAP 是什么？

> DotNetCore.CAP 是 .NET 生态中的事件总线和最终一致性框架。它不是 MQ 的替代品，而是基于 RabbitMQ、Kafka、Redis Streams 等消息队列之上的可靠消息框架。CAP 的核心是 Outbox 本地消息表模式，把业务数据变更和待发布消息放在同一个本地数据库事务中提交，然后由 CAP 后台任务把消息发送到 MQ，从而解决“数据库提交成功但消息发送失败”的问题。

---

## 24.2 CAP 和直接用 MQ 有什么区别？

> 直接用 MQ 只是发送消息，业务数据库事务和消息发送事务之间没有天然原子性，可能出现数据库成功但消息没发出去，或者消息发出去了但数据库回滚。CAP 在 MQ 之上封装了本地消息表、发布重试、消费重试、消息状态、订阅模型和 Dashboard。它的核心价值不是替代 MQ，而是补齐直接使用 MQ 时的可靠事件发布能力。

---

## 24.3 CAP 能保证强一致性吗？

> 不能。CAP 不是 2PC，也不是 DTC，它不保证多个服务同时成功或同时失败。CAP 保证的是本地事务和消息发布的可靠绑定，跨服务采用最终一致性和补偿。所以生产上仍然要设计状态机、幂等、重试、补偿、对账和人工处理。

---

## 24.4 CAP 最大的问题是什么？

> CAP 最大的问题是它引入了最终一致性模型，消费者必须自己处理幂等、顺序、重复消费、失败补偿和毒消息。CAP 的投递语义是 At Least Once，所以消息可能重复。并且 CAP 表会给数据库带来额外压力，高并发下要做清理、监控、告警和多实例重试控制。

---

## 24.5 什么场景适合 CAP？

适合：

```text
订单创建后通知库存服务
支付成功后通知订单服务
审批完成后通知工单服务
AI 分析完成后通知审核服务
退货质检完成后通知上架服务
```

前提：

```text
业务接受最终一致性
需要可靠事件发布
需要跨服务异步解耦
```

---

## 24.6 什么场景不适合 CAP？

不适合：

```text
强实时强一致扣款
证券撮合
高频日志流
复杂流处理
大文件传输
同进程内部普通方法调用
```

这些场景要分别考虑：

```text
本地事务
TCC
Saga
Kafka / Flink
任务调度系统
同步 RPC
```

---

# 25. 参考资料

> 以下资料用于核对 CAP 的官方定位、Outbox、事务模型、幂等、配置、重试、Dashboard、OpenTelemetry 等能力。

1. DotNetCore.CAP 官方文档首页：<https://cap.dotnetcore.xyz/>
2. DotNetCore.CAP GitHub：<https://github.com/dotnetcore/CAP>
3. CAP Transactions 文档：<https://cap.dotnetcore.xyz/user-guide/en/cap/transactions/>
4. CAP Idempotence 文档：<https://cap.dotnetcore.xyz/user-guide/en/cap/idempotence/>
5. CAP Configuration 文档：<https://cap.dotnetcore.xyz/user-guide/en/cap/configuration/>
6. CAP Messaging 文档：<https://cap.dotnetcore.xyz/user-guide/en/cap/messaging/>
7. CAP OpenTelemetry 文档：<https://cap.dotnetcore.xyz/user-guide/en/monitoring/opentelemetry/>

---

# 结论

CAP 的核心价值：

```text
解决“业务数据库提交成功，但 MQ 消息可能丢失”的问题。
```

CAP 的本质：

```text
Outbox 本地消息表
+
EventBus
+
最终一致性
+
失败重试
+
消息状态管理
```

CAP 的边界：

```text
它不是强一致分布式事务。
它不是 Saga。
它不是 Workflow。
它不是流处理框架。
它不会自动保证幂等。
它不会自动保证业务顺序。
```

生产级使用 CAP 的关键：

```text
同事务发布
事件建模
消费者幂等
状态机防乱序
重试治理
毒消息处理
业务补偿
可观测性
Dashboard 安全
事件版本治理
```

最重要的一句话：

> **CAP 是可靠事件投递基础设施，不是完整分布式业务解决方案。它解决消息可靠发布，但业务一致性、状态流转、幂等、补偿、权限、监控和版本治理，仍然必须由系统架构自己设计。**
