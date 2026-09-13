# The Truth About AddAsync: When to Use It in EF Core (and When Not To)

- Canonical URL: https://imzihad21.github.io/articles/a/the-truth-about-addasync-when-to-use-it-in-ef-core-and-when-not-to-3i5e/
- Source URL: https://dev.to/imzihad21/the-truth-about-addasync-when-to-use-it-in-ef-core-and-when-not-to-3i5e
- Web View: https://imzihad21.github.io/articles/a/the-truth-about-addasync-when-to-use-it-in-ef-core-and-when-not-to-3i5e/
- Published: 2025-05-28T18:05:08.000Z
- Modified: 2025-05-28T18:05:08.000Z
- Reading time: 5 minutes
- Tags: dotnet, efcore, valuegeneration, asyncawait

## The truth about AddAsync in EF Core: when to use it and when not to

Developers frequently assume Entity Framework Core API symmetry mandates using `AddAsync()` for all asynchronous code paths, while searching in vain for nonexistent `UpdateAsync()` or `RemoveAsync()` methods. This misconception leads to unnecessary state-machine allocation overhead on standard entities and reflects a fundamental misunderstanding of change tracking versus SQL execution boundaries.

Calling `AddAsync()` does not execute an immediate database insertion. It attaches an entity to the `ChangeTracker` and only performs asynchronous I/O if a configured `ValueGenerator` requires asynchronous key generation before attaching.

### The problem and production context

Misinterpreting the role of `AddAsync()` introduces state-machine overhead in high-throughput loops and creates erroneous assumptions regarding when database locks and constraints take effect.

- **Failure scenario**: An application loop invokes `await dbContext.Orders.AddAsync(order)` thousands of times assuming each call executes a non-blocking database write, while the actual blocking network payload accumulates uncommitted in memory until `SaveChangesAsync()` is called.
- **Why default approaches fall short**: Developers reflexively replace synchronous repository methods with `AddAsync()` under the impression that synchronous `Add()` blocks database socket I/O, unaware that `Add()` is an entirely in-memory operation taking microseconds unless custom asynchronous value generators are configured.
- **Production impact**: Micro-benchmarks exhibit elevated memory allocations from unnecessary `ValueTask` state machine instantiations, while thread-pool starvation can occur if custom asynchronous value generators are executed synchronously via `Add()`.

Understanding when EF Core executes SQL commands, preventing thread-pool starvation with asynchronous value generators, making informed API choices in high-throughput data layers, and understanding why state tracking methods rarely need asynchronous counterparts establishes efficient persistence patterns.

### Mental model and core concepts

Change tracking in EF Core separates in-memory entity graph mutations from database transaction boundaries.

#### 1. Change tracker attachment versus database execution

Attaching an entity to the `DbContext` change tracker sets its state flag to `EntityState.Added`. No socket I/O, network packets, or SQL `INSERT` statements are dispatched to the database engine during this step. Physical persistence occurs exclusively when calling `SaveChangesAsync()`.

#### 2. The purpose of asynchronous attachment

Because in-memory graph traversal and dictionary insertion complete in microseconds, synchronous `Add()` is standard. The `AddAsync()` method exists specifically to support `ValueGenerator<T>` implementations that must perform asynchronous I/O before assigning a primary key, such as querying a distributed Snowflake ID generator, reading a hi/lo sequence from a remote service, or awaiting an authentication token.

#### 3. Asynchronous value generation mechanics

A custom `ValueGenerator<T>` overrides `NextAsync()` to yield keys asynchronously. When invoked via `AddAsync()`, EF Core awaits the generator without blocking the active thread. If invoked via synchronous `Add()`, EF Core blocks synchronously on the asynchronous task, risking thread-pool starvation.

#### 4. Absence of UpdateAsync and RemoveAsync

Updating or deleting entities never generates new primary key values. Calling `Update()` or `Remove()` alters the entity's internal `EntityEntry.State` to `Modified` or `Deleted` entirely within memory. Because no disk or network I/O is possible during this in-memory flag update, EF Core provides no `UpdateAsync()` or `RemoveAsync()` methods.

#### 5. Practical selection rules

- Use synchronous `Add()` for the vast majority of entities. Entities using client-assigned GUIDs, application-assigned integers, or database-generated columns (such as SQL Server `IDENTITY` or PostgreSQL `SERIAL`/`IDENTITY`) require no pre-save I/O. Synchronous `Add()` avoids `ValueTask` allocation overhead.
- Use `AddAsync()` only when an entity explicitly configures a custom `ValueGenerator` that performs asynchronous operations prior to entity attachment.

### Production implementation

The following implementation demonstrates a custom asynchronous ID generator and the corresponding entity registration and persistence lifecycle.

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.ChangeTracking;
using Microsoft.EntityFrameworkCore.ValueGeneration;

public sealed class SnowflakeIdGenerator : ValueGenerator<long>
{
    public override bool GeneratesTemporaryValues => false;

    public override async ValueTask<long> NextAsync(
        EntityEntry entry,
        CancellationToken cancellationToken = default)
    {
        await Task.Delay(50, cancellationToken);
        return DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
    }

    public override long Next(EntityEntry entry)
    {
        return DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
    }
}

public sealed class Order
{
    public long Id { get; set; }
    public string CustomerName { get; set; } = string.Empty;
}

public sealed class StoreDbContext : DbContext
{
    public StoreDbContext(DbContextOptions<StoreDbContext> options) : base(options)
    {
    }

    public DbSet<Order> Orders => Set<Order>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Order>()
            .Property(e => e.Id)
            .HasValueGenerator<SnowflakeIdGenerator>();
    }
}

public sealed class OrderService(StoreDbContext dbContext)
{
    public async Task CreateOrderAsync(string customerName, CancellationToken cancellationToken)
    {
        if (string.IsNullOrWhiteSpace(customerName))
            throw new ArgumentException("Customer name cannot be empty.", nameof(customerName));

        var order = new Order
        {
            CustomerName = customerName
        };

        await dbContext.Orders.AddAsync(order, cancellationToken);
        await dbContext.SaveChangesAsync(cancellationToken);
    }
}
```

### Architectural trade-offs and edge cases

Choosing between `Add()` and `AddAsync()` involves balancing state-machine allocation overhead against thread-pool starvation risks.

* **Latency versus consistency**: Synchronous `Add()` avoids `ValueTask` state machine allocation overhead and executes immediately in memory, but if an asynchronous value generator is configured, calling `Add()` results in synchronous-over-asynchronous blocking (`Task.Result`), risking thread starvation under high concurrent load.
* **Failure recovery**: Neither `Add()` nor `AddAsync()` communicates with the database. Failures during this step are strictly in-memory validation or generator exceptions. If `SaveChangesAsync()` fails due to database constraints, connection timeouts, or duplicate keys, the entity remains attached in the `Added` state unless discarded or detached.
* **Scale limitations**: When inserting high-volume batches (thousands of entities), using `AddAsync()` in tight loops compounds garbage collection pressure through repeated `ValueTask` generation. For standard key generation strategies, use `Add()` or `AddRange()`.

### Common anti-patterns and gotchas

* **Assuming AddAsync dispatches immediate SQL**: Believing that `AddAsync()` executes an `INSERT` statement leads developers to inspect database records before calling `SaveChangesAsync()`. Remember that database I/O is deferred to `SaveChangesAsync()`.
* **Searching for nonexistent UpdateAsync and RemoveAsync**: Expecting asynchronous update or remove methods indicates a conceptual misunderstanding of change tracking. Modifying entity state in memory is always instantaneous and synchronous.
* **Applying AddAsync uniformly across all entities**: Using `AddAsync()` everywhere under the mistaken belief that it improves general persistence throughput adds micro-overhead. Standardize on `Add()` unless using asynchronous value generators.
* **Forgetting that connection acquisition occurs during SaveChangesAsync**: Connection pooling, physical connection opening, and transaction management happen during `SaveChangesAsync()`, not during entity registration.
* **Omitting cancellation tokens during SaveChangesAsync**: Failing to propagate `CancellationToken` to `SaveChangesAsync()` prevents long-running or stalled database insert statements from aborting during client disconnection.

### Implementation checklist

1. Review data access repositories and entity configurations to identify custom `ValueGenerator<T>` implementations.
2. If custom value generators perform asynchronous I/O, verify that `AddAsync()` is used to prevent thread-pool blocking.
3. Standardize on synchronous `Add()` for standard entity types where keys are database-generated or assigned in memory.
4. Remove any wrappers attempting to emulate `UpdateAsync()` or `RemoveAsync()`.
5. Ensure all database writes call `await dbContext.SaveChangesAsync(cancellationToken)`.
6. Verify that cancellation tokens are consistently passed to both `AddAsync()` and `SaveChangesAsync()`.