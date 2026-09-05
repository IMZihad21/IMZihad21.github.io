# Implementing Event-Driven Order Processing with Kafka using KafkaFlow in .NET

- Canonical URL: https://imzihad21.github.io/articles/a/implementing-event-driven-order-processing-with-kafka-using-kafkaflow-in-net-1m5a/
- Source URL: https://dev.to/imzihad21/implementing-event-driven-order-processing-with-kafka-using-kafkaflow-in-net-1m5a
- Web View: https://imzihad21.github.io/articles/a/implementing-event-driven-order-processing-with-kafka-using-kafkaflow-in-net-1m5a/
- Published: 2026-01-30T20:34:10.000Z
- Modified: 2026-01-30T20:34:10.000Z
- Reading time: 3 minutes
- Tags: dotnet, kafka, eventdriven, pubsub

## Implementing Event-Driven Order Processing with Kafka Using KafkaFlow in .NET

In real systems, producer and consumer workloads usually scale differently. API services publish events quickly, while background processors handle heavier business work. Mixing both in one deployment unit becomes messy.

This guide shows a clean KafkaFlow setup with producer-only and consumer-only registration, sharing one config contract.

### Why It Matters

- Producer and consumer can scale independently.
- Keeps API processes lightweight and focused.
- Isolates heavy event processing in worker services.
- Uses one shared configuration shape across projects.

### Core Concepts

#### 1. Shared Configuration Model

Define one options model used by both producer and consumer projects.

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

#### 2. Producer-Only Registration

Use this in API/command services.

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

#### 3. Consumer-Only Registration

Use this in worker/processor services.

```csharp
public static class KafkaFlowConsumerExtensions
{
    public static IServiceCollection AddKafkaFlowConsumer(
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

#### 4. Shared Event Contracts

```csharp
public record OrderCreated(Guid OrderId, Guid CustomerId, decimal TotalAmount, DateTime CreatedAt);
public record OrderPaid(Guid OrderId, string PaymentId, DateTime PaidAt);
public record OrderShipped(Guid OrderId, string TrackingNumber, DateTime ShippedAt);
```

#### 5. Consumer Typed Handlers

```csharp
public sealed class OrderCreatedHandler : IMessageHandler<OrderCreated>
{
    private readonly IInventoryService _inventory;

    public OrderCreatedHandler(IInventoryService inventory)
    {
        _inventory = inventory;
    }

    public Task Handle(OrderCreated message, CancellationToken cancellationToken)
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

    public Task Handle(OrderPaid message, CancellationToken cancellationToken)
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

    public Task Handle(OrderShipped message, CancellationToken cancellationToken)
    {
        return _notification.SendOrderUpdateAsync(message.OrderId, "shipped");
    }
}
```

#### 6. Producer Publisher Service

Use strongly typed key selector to avoid reflection-heavy magic.

```csharp
public sealed class OrderEventPublisher
{
    private readonly IMessageProducer _producer;

    public OrderEventPublisher(IMessageProducer producer)
    {
        _producer = producer;
    }

    public Task PublishAsync<T>(string key, T eventPayload) where T : class
    {
        var message = new Message<string, T>
        {
            Key = key,
            Value = eventPayload
        };

        return _producer.ProduceAsync(message);
    }
}
```

### Practical Example

`Program.cs` in API/command project:

```csharp
builder.Services.AddKafkaFlowProducer(builder.Configuration);
```

`Program.cs` in worker project:

```csharp
builder.Services.AddKafkaFlowConsumer(builder.Configuration);
```

Shared config in `appsettings.json`:

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

This separation keeps architecture clean. Producers push fast, consumers process heavy, everyone stays sane.

### Common Mistakes

- Registering consumers accidentally inside API projects.
- Hardcoding broker list separately across services.
- Using inconsistent topic names between producer and consumer.
- Ignoring consumer group strategy during horizontal scaling.
- Publishing events without deterministic keys when ordering matters.

### Quick Recap

- Use one shared config model, two separate registrations.
- Producers and consumers should live in different deployment units.
- Typed handlers keep event processing explicit.
- KafkaFlow channel balancing helps under variable lag.
- This pattern scales better and is easier to operate.

### Next Steps

1. Add serializer strategy and schema evolution contract.
2. Add retry/dead-letter handling for failed events.
3. Add idempotency safeguards in consumers.
4. Add end-to-end tests for producer-to-consumer flow.