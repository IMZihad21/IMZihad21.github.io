# Implementing Soft Delete with Filtered Indexes in Entity Framework Core

- Canonical URL: https://imzihad21.github.io/articles/a/implementing-soft-delete-with-filtered-indexes-in-entity-framework-core-49ip/
- Source URL: https://dev.to/imzihad21/implementing-soft-delete-with-filtered-indexes-in-entity-framework-core-49ip
- Web View: https://imzihad21.github.io/articles/a/implementing-soft-delete-with-filtered-indexes-in-entity-framework-core-49ip/
- Published: 2026-04-22T16:32:18.000Z
- Modified: 2026-04-22T16:32:18.000Z
- Reading time: 6 minutes
- Tags: dotnet, database, softdelete, efcore

## Implementing soft delete with filtered indexes in Entity Framework Core

Soft delete implementations often fail in production because adding a simple `IsDeleted` boolean flag introduces unintended side effects across queries, unique constraints, and foreign key cascades. When applications execute standard delete operations, downstream relations can break unexpectedly, or unique index violations can block new active records from reusing identifying values like email addresses or usernames.

This pattern couples explicit domain-level entity contracts with Entity Framework Core change tracking interception, automated global query filters, and database-level filtered unique indexes. The architecture guarantees that standard deletion APIs safely convert into state updates, active reads systematically exclude soft-deleted rows, and unique constraints remain enforceable on active data without sacrificing historical associations.

### The problem and production context

In production databases, naive soft delete implementations create friction between domain models, persistence layers, and schema constraints. When a system sets a boolean flag without infrastructure support, developers must remember to filter out deleted rows in every single query. Standard unique database constraints also reject valid insert operations when new entities share unique keys with inactive, soft-deleted records.

- **Failure scenario**: An application attempts to register a new user using an email address that previously belonged to a deleted account. The database raises a unique constraint violation error because the naive unique index includes soft-deleted rows. Alternatively, a developer writes an entity query and forgets to append `.Where(x => !x.IsDeleted)`, exposing logically deleted data in production user interfaces and reports.
- **Why default approaches fall short**: Default unique indexes enforce uniqueness across all rows regardless of deletion status. In addition, manual filtering across ad-hoc queries is prone to human error, and relying on database cascading hard-deletes strips historical associations and breaks soft-delete audit trails.
- **Production impact**: Preventable unique constraint violations block customer registrations and data entry. Accidental data leaks occur when omitted query filters expose sensitive soft-deleted rows, and untracked database cascades physically purge records intended for regulatory audit and operational recovery.

A complete soft delete implementation requires:
- Default queries that systematically hide deleted rows across all application read operations.
- A write path that transparently converts deletes into updates without breaking service-layer abstractions.
- Database-level unique constraints that ignore soft-deleted rows.
- An explicit restore path that maintains internal entity state consistency.

### Mental model and core concepts

#### 1. Domain encapsulation via explicit contracts

Soft delete is a domain concern rather than a database-only trick. The domain model should explicitly express that an entity can be deleted and restored. An interface defines the minimal contract without forcing entities into a rigid inheritance tree:

```csharp
public interface ISoftDeletable
{
    bool IsDeleted { get; }
    DateTimeOffset? DeletedAt { get; }

    void Delete();
    void Restore();
}
```

A base entity provides a default implementation that encapsulates state transitions and prevents invalid mutations:

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

This encapsulates deletion and restoration logic within the entity itself.

#### 2. Transparent delete interception via change tracking

To preserve the standard EF Core delete API, intercept entities marked as `Deleted` before changes persist to the database. Overriding `SaveChanges` and `SaveChangesAsync` converts entries into `Modified` state and invokes the domain `Delete()` method:

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

The service layer continues using the standard deletion call:

```csharp
context.Users.Remove(user);
await context.SaveChangesAsync();
```

From the perspective of calling code, the entity is removed. In the database, the row remains with its deletion flags updated. If a project already uses EF Core interceptors for auditing or multi-tenancy, this logic can run inside an interceptor instead.

#### 3. Global query filters and automated model conventions

Global query filters keep deleted rows out of standard queries. Configuring filters manually on individual entities becomes repetitive and error-prone across large models. A reflection-driven convention pass in `OnModelCreating` automatically binds query filters, configures filtered unique indexes, and sets foreign key delete behaviors to `DeleteBehavior.NoAction` across all `ISoftDeletable` entities:

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

This centralizes persistence conventions in `OnModelCreating` instead of configuring each entity manually. Normal reads exclude deleted records without manual `.Where(u => !u.IsDeleted)` conditions.

When deleted records are required, disable the filter using `IgnoreQueryFilters()`:

```csharp
var deletedUser = await context.Users
    .IgnoreQueryFilters()
    .FirstOrDefaultAsync(u => u.Id == userId && u.IsDeleted);
```

This provides a clear escape hatch for audit logs, administrative portals, and restore routines.

#### 4. Database filtered unique indexes

Soft-delete designs encounter constraint issues with standard unique database indexes. If an `Email` column has an unconditional unique constraint, a soft-deleted row prevents a new user from registering with that same email address. Applying a partial index filter restricts uniqueness checks to active records.

For SQL Server:

```csharp
index.SetFilter($"[{nameof(ISoftDeletable.IsDeleted)}] = 0");
```

For PostgreSQL, the equivalent filter is:

```csharp
index.SetFilter($"\"{nameof(ISoftDeletable.IsDeleted)}\" = false");
```

Key considerations for unique indexes:
- Restrict uniqueness checks strictly to active records.
- Avoid relying on application-level checks to enforce uniqueness.
- Verify generated migration scripts against your database provider.

Setting foreign key relationships to `DeleteBehavior.NoAction` during the same pass prevents soft deletes from triggering unintended database cascades.

### Production implementation

The following workflow restores soft-deleted records through explicit domain entity methods combined with bypassing global query filters.

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

This guarantees that entity state transitions remain encapsulated within the domain model while EF Core tracks the update.

### Architectural trade-offs and edge cases

* **Latency versus consistency**: Global query filters inject an additional boolean condition into every generated SQL query. In return, the system guarantees that soft-deleted rows never leak into standard queries without requiring manual filter enforcement across thousands of application queries.
* **Failure recovery**: Restoring a soft-deleted record when another active record has since taken the same unique value (such as an email address) causes a database unique constraint violation. Restoration services must catch unique constraint exceptions or perform validation before invoking `Restore()`.
* **Scale limitations**: When tables accumulate millions of soft-deleted records over time, table size and scan performance degrade if queries touch unindexed columns. A dedicated partitioning strategy or background archival job is required to purge or move long-deleted rows into cold storage.
* **Bulk operations**: Bulk operations such as EF Core's `ExecuteDelete` or raw SQL commands bypass the change tracker entirely and do not trigger `SaveChanges` overrides or interceptors. Physical deletes occur unless separate database triggers or explicit update scripts run.
* **Relationship navigation**: Navigating required relationships (`Include`) where the target related record is soft-deleted can cause queries to return null or filter out parent records entirely. Relationships must be tested to verify intended join behavior under global query filters.
* **Hard delete requirements**: Soft delete handles recovery, audit trails, and referential integrity. However, when compliance rules (such as GDPR or CCPA) mandate physical data destruction, a dedicated hard-delete administrative routine remains necessary.

### Common anti-patterns and gotchas

* **Indexing IsDeleted in isolation**: Developers add a standalone index on `IsDeleted`. Because boolean flags possess very low selectivity, database query planners ignore this index during query execution. Filtered indexes on candidate keys should be used instead.
* **Relying on application-level uniqueness checks**: Developers check for duplicate entries in application code before inserting rather than using filtered database indexes. Concurrent write operations bypass application checks, producing duplicate active records and corrupting data.
* **Allowing default cascade behaviors on foreign keys**: Default EF Core cascade rules may attempt to delete child records or throw constraint exceptions when parent records are marked deleted. Setting foreign key relationships to `DeleteBehavior.NoAction` prevents unintended cascading deletes.
* **Assuming bulk operations trigger soft delete**: Developers assume calling `context.Users.Where(...).ExecuteDeleteAsync()` executes soft delete logic. Bulk operations bypass the change tracker and permanently delete records from the physical table.
* **Scattering manual filter clauses**: Writing `.Where(x => !x.IsDeleted)` manually across individual queries inevitably results in omitted filters and production data leaks. Global query filters must be enforced by convention.

### Implementation checklist

1. Define the `ISoftDeletable` interface and implement it on base or domain entities with encapsulated state mutators.
2. Override `SaveChanges` and `SaveChangesAsync` in `DbContext` (or implement an `ISaveChangesInterceptor`) to intercept `EntityState.Deleted` and convert to `EntityState.Modified`.
3. Register automated global query filters in `OnModelCreating` using expression trees across all `ISoftDeletable` entities.
4. Configure filtered partial unique indexes for your target database engine (SQL Server or PostgreSQL) across all candidate keys.
5. Set foreign key delete behaviors to `DeleteBehavior.NoAction` to prevent unintended database cascades.
6. Verify generated database migration scripts to confirm filtered index definitions match the target database syntax.
7. Test queries that use `Include` on required relationships to confirm join behaviors under global query filters.
8. Establish an explicit administrative archival path for GDPR compliance and cold storage data purging.