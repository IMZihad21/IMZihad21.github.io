# Implementing Event-Driven Order Processing with Kafka using KafkaFlow in .NET

- Canonical URL: https://imzihad21.github.io/articles/a/implementing-event-driven-order-processing-with-kafka-using-kafkaflow-in-net-1m5a/
- Source URL: https://dev.to/imzihad21/implementing-event-driven-order-processing-with-kafka-using-kafkaflow-in-net-1m5a
- Web View: https://imzihad21.github.io/articles/a/implementing-event-driven-order-processing-with-kafka-using-kafkaflow-in-net-1m5a/
- Published: 2026-01-30T20:34:10.000Z
- Modified: 2026-01-30T20:34:10.000Z
- Reading time: 6 minutes
- Tags: dotnet, kafka, eventdriven, pubsub

## Implementing event-driven order processing with Kafka using KafkaFlow in .NET

In distributed systems, producer and consumer workloads scale along completely different resource profiles. Web APIs require sub-millisecond dispatch times with minimal memory footprints, while background event consumers execute long-running I/O, database writes, and third-party integrations. Combining producer and consumer lifecycles within the same application process creates resource contention and prevents independent horizontal scaling.

Structuring KafkaFlow with decoupled producer and consumer registrations backed by a shared configuration contract guarantees isolated resource allocation, high API responsiveness, and scalable background message processing.

### The problem and production context

Event-driven architectures often begin by registering Kafka producers and consumers in the same application host. As transaction volume scales, this monolithic configuration leads to operational failure.

- **Failure scenario**: A surge in client order submissions causes consumer threads within an API process to execute heavy database queries and external payment gateway calls. Thread pool starvation ensues, leading to API request timeouts, high consumer lag, and inability to auto-scale API web pods without redundantly spinning up consumer partitions.
- **Why default approaches fall short**: Monolithic registrations bind consumer group rebalancing, thread distribution, and broker connection pools to web application lifecycles. Scaling the web layer to absorb HTTP traffic multiplies consumer instances beyond the partition count of the topic, causing idle consumer threads and rebalance storms.
- **Production impact**: API latency spikes under consumer load, background processing backs up across partitions, and deployment rollouts take longer due to coordinated consumer rebalances.

### Mental model and core concepts

Separating event publishing from message processing relies on modular dependency injection, typed event routing, and consumer worker pools.

#### 1. Shared configuration model

Centralizing Kafka cluster endpoints, topic names, consumer groups, and client identifiers in a strongly typed configuration model ensures configuration parity between publishing APIs and processing workers:

```csharp
public sealed class KafkaFlowConfiguration
{
    public string ServerUrl { get; set; } = string.Empty;
    public OrderConfig Orders { get; set; } = new();
}

public sealed class OrderConfig
{
    public string TopicName { get; set; } = string.Empty;
    public string ConsumerGroup { get; set; } = string.Empty;
    public string ConsumerName { get; set; } = string.Empty;
}
```

#### 2. Producer-only registration pattern

API hosts register only Kafka producer instances and topic definitions, avoiding the overhead of background worker pools, partition listeners, and message deserializers:

```csharp
public static class KafkaFlowProducerExtensions
{
    public static IServiceCollection AddKafkaFlowProducer(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        var config = configuration
            .GetSection("KafkaConfiguration")
            .Get<KafkaFlowConfiguration>()
            ?? throw new InvalidOperationException("KafkaConfiguration is missing.");

        services.AddKafka(kafka => kafka
            .AddCluster(cluster => cluster
                .WithBrokers(config.ServerUrl.Split(',', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries))
                .CreateTopicIfNotExists(config.Orders.TopicName, 10, 1)
                .AddProducer("order-producer", producer =>
                    producer.DefaultTopic(config.Orders.TopicName))));

        return services;
    }
}
```

#### 3. Consumer worker balancing and typed handlers

Worker services register consumers configured with explicit worker pools, lag balancers, and typed message handlers. Using `FreeWorkerDistributionStrategy` alongside `WithConsumerLagWorkerBalancer` allows the worker process to dynamically scale concurrent message processing threads between defined minimum and maximum thresholds based on consumer lag.

#### 4. Shared event contracts

Business events represent immutable facts within the domain and are defined as pure record types shared between producers and consumers:

```csharp
public record OrderCreated(Guid OrderId, Guid CustomerId, decimal TotalAmount, DateTime CreatedAt);
public record OrderPaid(Guid OrderId, string PaymentId, DateTime PaidAt);
public record OrderShipped(Guid OrderId, string TrackingNumber, DateTime ShippedAt);
```

#### 5. Strongly typed event publisher

Encapsulating KafkaFlow's `IMessageProducer` inside a domain publisher service enforces partition key usage to guarantee message ordering across topic partitions without reflection overhead.

#### 6. Architecture configuration topology

The producer service (Web API) and consumer service (Background Worker) consume the same configuration structure while remaining isolated processes:

```json
{
  "KafkaConfiguration": {
    "ServerUrl": "kafka-1:9092,kafka-2:9092",
    "Orders": {
      "TopicName": "orders",
      "ConsumerGroup": "order-processing",
      "ConsumerName": "order-events-handler"
    }
  }
}
```

### Production implementation

Implement the configuration extensions, typed domain handlers, publisher service, and application startup configuration.

```csharp
using KafkaFlow;
using KafkaFlow.Consumers.DistributionStrategies;
using KafkaFlow.Producers;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;

public sealed class KafkaFlowConfiguration
{
    public string ServerUrl { get; set; } = string.Empty;
    public OrderConfig Orders { get; set; } = new();
}

public sealed class OrderConfig
{
    public string TopicName { get; set; } = string.Empty;
    public string ConsumerGroup { get; set; } = string.Empty;
    public string ConsumerName { get; set; } = string.Empty;
}

public record OrderCreated(Guid OrderId, Guid CustomerId, decimal TotalAmount, DateTime CreatedAt);
public record OrderPaid(Guid OrderId, string PaymentId, DateTime PaidAt);
public record OrderShipped(Guid OrderId, string TrackingNumber, DateTime ShippedAt);

public interface IInventoryService
{
    Task ReserveStockAsync(Guid orderId);
}

public interface IShippingService
{
    Task InitiateAsync(Guid orderId);
}

public interface INotificationService
{
    Task SendOrderUpdateAsync(Guid orderId, string status);
}

public sealed class OrderCreatedHandler : IMessageHandler<OrderCreated>
{
    private readonly IInventoryService _inventory;

    public OrderCreatedHandler(IInventoryService inventory)
    {
        _inventory = inventory;
    }

    public Task Handle(IMessageContext context, OrderCreated message)
    {
        return _inventory.ReserveStockAsync(message.OrderId);
    }
}

public sealed class OrderPaidHandler : IMessageHandler<OrderPaid>
{
    private readonly IShippingService _shipping;

    public OrderPaidHandler(IShippingService shipping)
    {
        _shipping = shipping;
    }

    public Task Handle(IMessageContext context, OrderPaid message)
    {
        return _shipping.InitiateAsync(message.OrderId);
    }
}

public sealed class OrderShippedHandler : IMessageHandler<OrderShipped>
{
    private readonly INotificationService _notification;

    public OrderShippedHandler(INotificationService notification)
    {
        _notification = notification;
    }

    public Task Handle(IMessageContext context, OrderShipped message)
    {
        return _notification.SendOrderUpdateAsync(message.OrderId, "shipped");
    }
}

public sealed class OrderEventPublisher
{
    private readonly IMessageProducer _producer;

    public OrderEventPublisher(IMessageProducer producer)
    {
        _producer = producer;
    }

    public Task PublishAsync<T>(string key, T eventPayload) where T : class
    {
        return _producer.ProduceAsync(
            key,
            eventPayload);
    }
}

public static class KafkaFlowRegistrationExtensions
{
    public static IServiceCollection AddKafkaFlowProducer(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        KafkaFlowConfiguration config = configuration
            .GetSection("KafkaConfiguration")
            .Get<KafkaFlowConfiguration>()
            ?? throw new InvalidOperationException("KafkaConfiguration is missing.");

        string[] brokers = config.ServerUrl.Split(
            ',',
            StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries);

        services.AddKafka(kafka => kafka
            .AddCluster(cluster => cluster
                .WithBrokers(brokers)
                .CreateTopicIfNotExists(config.Orders.TopicName, 10, 1)
                .AddProducer("order-producer", producer =>
                    producer.DefaultTopic(config.Orders.TopicName))));

        services.AddSingleton<OrderEventPublisher>();

        return services;
    }

    public static IServiceCollection AddKafkaFlowConsumer(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        KafkaFlowConfiguration config = configuration
            .GetSection("KafkaConfiguration")
            .Get<KafkaFlowConfiguration>()
            ?? throw new InvalidOperationException("KafkaConfiguration is missing.");

        string[] brokers = config.ServerUrl.Split(
            ',',
            StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries);

        services.AddKafka(kafka => kafka
            .AddCluster(cluster => cluster
                .WithBrokers(brokers)
                .CreateTopicIfNotExists(config.Orders.TopicName, 10, 1)
                .AddConsumer(consumer => consumer
                    .Topic(config.Orders.TopicName)
                    .WithGroupId(config.Orders.ConsumerGroup)
                    .WithName(config.Orders.ConsumerName)
                    .WithBufferSize(10)
                    .WithConsumerLagWorkerBalancer(maxWorkers: 100, minWorkers: 10, lagThreshold: 25)
                    .WithWorkerDistributionStrategy<FreeWorkerDistributionStrategy>()
                    .AddMiddlewares(middlewares => middlewares
                        .AddTypedHandlers(handlers => handlers
                            .AddHandler<OrderCreatedHandler>()
                            .AddHandler<OrderPaidHandler>()
                            .AddHandler<OrderShippedHandler>())))));

        return services;
    }
}
```

In the Web API `Program.cs`:

```csharp
builder.Services.AddKafkaFlowProducer(builder.Configuration);
```

In the Worker service `Program.cs`:

```csharp
builder.Services.AddKafkaFlowConsumer(builder.Configuration);
```

### Architectural trade-offs and edge cases

Decoupling Kafka producers from consumers introduces operational considerations across partition assignment, ordering semantics, and delivery guarantees.

* **Latency versus consistency**: Publishing asynchronously to Kafka minimizes HTTP API response latency. However, it introduces eventual consistency: downstream database projections and order fulfillment states update milliseconds or seconds after the initial HTTP request finishes.
* **Failure recovery**: In the event of temporary broker disconnections, KafkaFlow producers buffer outbound messages in memory according to configured retry thresholds. For background consumers, transient handler exceptions must trigger exponential backoff retries and route poison-pill messages to a dead-letter queue (DLQ) to prevent consumer loop blockage.
* **Scale limitations**: Maximum parallelism within a single consumer group is bounded by the topic's partition count. If an orders topic has 10 partitions, allocating more than 10 active consumer process instances leaves redundant pods idle. The internal worker balancer mitigates this by distributing messages across multiple worker threads per partition using `FreeWorkerDistributionStrategy`.

### Common anti-patterns and gotchas

* **Registering consumers in API hosts**: Accidentally invoking `AddKafkaFlowConsumer` within public-facing Web API deployments causes background message processing to compete with HTTP request threads for CPU and database connections.
* **Uncoordinated broker lists**: Maintaining separate, out-of-sync broker connection strings across services causes intermittent communication failures during broker cluster rotations. Always share configuration structures across services.
* **Mismatched topic configurations**: Defining differing topic names between producer configurations and consumer subscriptions causes events to be published into unmonitored topics.
* **Neglecting partition keys**: Publishing order events with null or static partition keys distributes all messages to a single partition or breaks ordering. Always provide an explicit entity identifier (such as `OrderId`) as the message key to preserve event ordering per aggregate.
* **Non-idempotent consumer handlers**: Assuming messages will arrive exactly once. Network retries or consumer group rebalances can cause duplicate delivery. All consumer handlers must implement idempotent database updates or maintain deduplication logs.

### Implementation checklist

1. Define a shared `KafkaFlowConfiguration` binding cluster endpoints and order topic parameters.
2. Implement `AddKafkaFlowProducer` in a reusable extension for Web API services.
3. Implement `AddKafkaFlowConsumer` in worker services with typed handlers and worker balancing.
4. Define immutable event contracts using C# record types (`OrderCreated`, `OrderPaid`, `OrderShipped`).
5. Ensure message publishing uses domain entity keys (`OrderId`) for deterministic partition routing.
6. Implement idempotent handling or deduplication verification within each consumer handler.
7. Configure retry middleware and dead-letter queues for unprocessable messages.
8. Validate broker connectivity and consumer group assignment using integration tests.