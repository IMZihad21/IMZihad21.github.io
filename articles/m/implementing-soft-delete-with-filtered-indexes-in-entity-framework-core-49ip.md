# Implementing Soft Delete with Filtered Indexes in Entity Framework Core

- Canonical URL: https://imzihad21.github.io/articles/a/implementing-soft-delete-with-filtered-indexes-in-entity-framework-core-49ip/
- Source URL: https://dev.to/imzihad21/implementing-soft-delete-with-filtered-indexes-in-entity-framework-core-49ip
- Web View: https://imzihad21.github.io/articles/a/implementing-soft-delete-with-filtered-indexes-in-entity-framework-core-49ip/
- Published: 2026-04-22T16:32:18.000Z
- Modified: 2026-04-22T16:32:18.000Z
- Reading time: 4 minutes
- Tags: dotnet, database, softdelete, efcore

### Soft Delete with Global Query Filters and Filtered Indexes in Entity Framework Core

Soft delete sounds simple until it reaches production.

If you only add an `IsDeleted` flag, you have not finished the job. You still need:

- Default queries that hide deleted rows
- A write path that converts deletes into updates
- Unique constraints that ignore soft-deleted rows
- A restore path that does not break the model

This article shows a clean EF Core pattern that keeps the implementation explicit and maintainable.

### The Core Idea

Soft delete is a domain concern, not just a database trick. The model should express that an entity can be deleted and restored.

An interface keeps the contract small and avoids forcing every entity into the same inheritance tree:

```csharp
public interface ISoftDeletable
{
    bool IsDeleted { get; }
    DateTimeOffset? DeletedAt { get; }

    void Delete();
    void Restore();
}
```

A simple base entity can implement the behavior:

```csharp
public abstract class BaseEntity : ISoftDeletable
{
    public Guid Id { get; private set; } = Guid.NewGuid();
    public bool IsDeleted { get; private set; }
    public DateTimeOffset? DeletedAt { get; private set; }

    public void Delete()
    {
        if (IsDeleted)
        {
            return;
        }

        IsDeleted = true;
        DeletedAt = DateTimeOffset.UtcNow;
    }

    public void Restore()
    {
        if (!IsDeleted)
        {
            return;
        }

        IsDeleted = false;
        DeletedAt = null;
    }
}
```

That gives the application one place to define the lifecycle of a deleted record.

### Turn Deletes Into Updates

The easiest way to preserve EF Core's normal API is to intercept `Deleted` entities before they hit the database.

For most applications, a `SaveChanges` override is enough:

```csharp
using System.Linq;
using Microsoft.EntityFrameworkCore;

public class ApplicationDbContext : DbContext
{
    private void ApplySoftDelete()
    {
        foreach (var entry in ChangeTracker.Entries<ISoftDeletable>()
                     .Where(e => e.State == EntityState.Deleted))
        {
            entry.State = EntityState.Modified;
            entry.Entity.Delete();
        }
    }

    public override int SaveChanges()
    {
        ApplySoftDelete();
        return base.SaveChanges();
    }

    public override Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        ApplySoftDelete();
        return base.SaveChangesAsync(cancellationToken);
    }
}
```

This keeps your service layer clean:

```csharp
context.Users.Remove(user);
await context.SaveChangesAsync();
```

From the caller's perspective, it is still a delete. Internally, the row is retained and marked as deleted.

If your project already uses EF Core interceptors for auditing or multi-tenant behavior, you can move this logic there instead. The important part is consistency, not the specific hook.

### Filter Rows By Default

Global query filters ensure deleted rows stay out of normal reads.

For one or two entities, a per-entity filter works. In a real system, that becomes repetition. The better approach is to let `OnModelCreating` run one convention pass across the model after loading your normal entity configurations:

```csharp
using System.Linq;
using System.Linq.Expressions;

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(SomeEntityConfiguration).Assembly);

    foreach (var entityType in modelBuilder.Model.GetEntityTypes())
    {
        if (typeof(ISoftDeletable).IsAssignableFrom(entityType.ClrType))
        {
            var parameter = Expression.Parameter(entityType.ClrType, "e");
            var isDeleted = Expression.Property(parameter, nameof(ISoftDeletable.IsDeleted));
            var isActive = Expression.Equal(isDeleted, Expression.Constant(false));
            var filter = Expression.Lambda(isActive, parameter);

            modelBuilder.Entity(entityType.ClrType).HasQueryFilter(filter);

            foreach (var index in entityType.GetIndexes().Where(i => i.IsUnique && i.GetFilter() is null))
            {
                index.SetFilter($"[{nameof(ISoftDeletable.IsDeleted)}] = 0");
            }
        }

        foreach (var foreignKey in entityType.GetForeignKeys())
        {
            foreignKey.DeleteBehavior = DeleteBehavior.NoAction;
        }
    }

    base.OnModelCreating(modelBuilder);
}
```

This is the important shift: `OnModelCreating` becomes the place where persistence conventions are enforced, not an area where each entity gets hand-tuned individually.

That gives you the default behavior you want without repeating `Where(u => !u.IsDeleted)` everywhere.

When you intentionally need deleted rows, use `IgnoreQueryFilters()`:

```csharp
var deletedUser = await context.Users
    .IgnoreQueryFilters()
    .FirstOrDefaultAsync(u => u.Id == userId && u.IsDeleted);
```

That is the right escape hatch for restore flows, admin views, and audits.

### Protect Unique Constraints

This is where many soft-delete implementations break down.

If `Email` is unique, then a deleted user should not block another user from using the same email later. In the convention above, every unique index that does not already define its own filter gets the soft-delete predicate automatically.

The sample above uses SQL Server syntax:

```csharp
index.SetFilter($"[{nameof(ISoftDeletable.IsDeleted)}] = 0");
```

For PostgreSQL, the equivalent would look like:

```csharp
index.SetFilter($"\"{nameof(ISoftDeletable.IsDeleted)}\" = false");
```

The exact filter expression is provider-specific, but the architectural rule is not.

The architectural point is more important than the syntax:

- Filter unique indexes on active rows only
- Do not rely on `IsDeleted` alone to preserve uniqueness
- Verify the generated migration for your provider

That same model pass is also a good place to set foreign keys to `DeleteBehavior.NoAction` so soft delete does not accidentally fight with cascade delete rules.

### Restore Safely

Restoring a row should be a first-class operation, not a manual flag flip hidden in business code.

```csharp
var user = await context.Users
    .IgnoreQueryFilters()
    .FirstOrDefaultAsync(u => u.Id == userId);

if (user is null)
{
    return;
}

user.Restore();
await context.SaveChangesAsync();
```

This works because the model already knows how deletion and restoration behave.

### Design Notes

- Prefer a global query filter over ad hoc `Where` clauses. It is harder to forget and easier to reason about.
- Use filtered or partial indexes for business keys that must remain unique among active rows.
- Do not add an index on `IsDeleted` by default. The column is usually low-selectivity and rarely useful on its own.
- Test `Include` queries and required relationships. Global filters apply everywhere, and that can affect query shape.
- If you use bulk operations such as `ExecuteDelete`, remember they bypass the change tracker and will not trigger your soft-delete pipeline.

### When This Pattern Fits

This approach works well when you need:

- Auditability
- Recovery from accidental deletes
- Safer admin tooling
- Stable referential history

It is less useful when data must be physically removed for legal, privacy, or retention reasons. In those cases, hard delete is the correct answer.

### Conclusion

Soft delete is a small feature that becomes a large architectural problem if you treat it as a single boolean column.

The production-ready version has three parts:

1. Convert deletes into updates
2. Filter deleted rows out of normal queries
3. Make unique constraints ignore deleted rows

Once those are in place, the rest of the application can use EF Core naturally without repeating soft-delete rules everywhere.

### References

- [Global Query Filters - EF Core](https://learn.microsoft.com/en-us/ef/core/querying/filters)
- [Indexes - EF Core](https://learn.microsoft.com/en-us/ef/core/modeling/indexes)
- [Interceptors - EF Core](https://learn.microsoft.com/en-us/ef/core/logging-events-diagnostics/interceptors)