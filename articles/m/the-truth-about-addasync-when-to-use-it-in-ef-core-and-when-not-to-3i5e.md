# The Truth About AddAsync: When to Use It in EF Core (and When Not To)

- Canonical URL: https://imzihad21.github.io/articles/a/the-truth-about-addasync-when-to-use-it-in-ef-core-and-when-not-to-3i5e/
- Source URL: https://dev.to/imzihad21/the-truth-about-addasync-when-to-use-it-in-ef-core-and-when-not-to-3i5e
- Web View: https://imzihad21.github.io/articles/a/the-truth-about-addasync-when-to-use-it-in-ef-core-and-when-not-to-3i5e/
- Published: 2025-05-28T18:05:08.000Z
- Modified: 2025-05-28T18:05:08.000Z
- Reading time: 2 minutes
- Tags: dotnet, efcore, valuegeneration, asyncawait

## The Truth About AddAsync in EF Core: When to Use It and When Not To

`AddAsync()` often confuses developers because EF Core has no `UpdateAsync()` or `RemoveAsync()`. It looks inconsistent, but the reason is technical and very specific.

This guide explains exactly what `AddAsync()` does, why it exists, and when it actually matters.

### Why It Matters

- Prevents incorrect assumptions about EF Core async behavior.
- Avoids blocking threads in async ID generation scenarios.
- Improves performance decisions in high-concurrency apps.
- Helps you choose the right API with confidence.

### Core Concepts

#### 1. `AddAsync()` Does Not Insert into Database

`AddAsync()` only starts entity tracking in `Added` state. Database insert happens at `SaveChangesAsync()`.

```csharp
await dbContext.Users.AddAsync(user);
await dbContext.SaveChangesAsync();
```

#### 2. Why `Async` Exists at All

The async part is for asynchronous value generation before save (for example custom ID generators).

#### 3. Custom Async Value Generator Example

```csharp
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
```

#### 4. `Add()` vs `AddAsync()` Behavior

- `Add()` may force sync path.
- `AddAsync()` allows non-blocking async generator path.

#### 5. Why No `UpdateAsync()` or `RemoveAsync()`

`Update()` and `Remove()` only change entity state to `Modified` or `Deleted`. They do not require async value generation.

#### 6. Real Rule of Thumb

Use `AddAsync()` when async value generation exists or may exist. Otherwise `Add()` is usually fine.

### Practical Example

```csharp
var order = new Order
{
    CustomerName = "Alex"
};

await dbContext.Orders.AddAsync(order, cancellationToken);
await dbContext.SaveChangesAsync(cancellationToken);
```

In async-first APIs, this keeps the write path consistent and future-friendly. If later you plug in distributed ID generation, your code is already ready.

### Common Mistakes

- Assuming `AddAsync()` performs immediate database I/O.
- Using `Add()` with async value generation in high-load scenarios.
- Looking for `UpdateAsync()` and `RemoveAsync()` equivalents.
- Mixing sync and async persistence patterns inconsistently.
- Forgetting that real DB work starts at `SaveChangesAsync()`.

### Quick Recap

- `AddAsync()` is about async pre-save value generation.
- It does not execute SQL insert by itself.
- `Update()`/`Remove()` are state changes, so no async variant.
- `SaveChangesAsync()` is where database write happens.
- In async services, `AddAsync()` is a safe default for adds.

### Next Steps

1. Audit your repositories for mixed `Add()`/`AddAsync()` usage.
2. Confirm whether your model uses async value generators.
3. Standardize persistence style for better team consistency.
4. Add performance tests for high-throughput write endpoints.