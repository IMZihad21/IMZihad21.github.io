# Cursor vs Offset Pagination in EF Core: What Actually Works in Production

- Canonical URL: https://imzihad21.github.io/articles/a/cursor-vs-offset-pagination-in-ef-core-what-actually-works-in-production-448m/
- Source URL: https://dev.to/imzihad21/cursor-vs-offset-pagination-in-ef-core-what-actually-works-in-production-448m
- Web View: https://imzihad21.github.io/articles/a/cursor-vs-offset-pagination-in-ef-core-what-actually-works-in-production-448m/
- Published: 2026-03-19T10:18:33.000Z
- Modified: 2026-03-19T10:18:33.000Z
- Reading time: 5 minutes
- Tags: dotnet, efcore, pagination, performance

## Cursor vs offset pagination in EF Core: what works in production

Pagination is simple to implement at small scales, but query performance and data consistency degrade as tables grow into millions of rows. Under heavy read workloads, naive pagination patterns create database bottlenecks that exhaust database I/O, increase latency, and return inconsistent page windows to clients.

Keyset cursor pagination replaces arbitrary offset scans with index-seek lookups based on deterministic sort keys, guaranteeing constant-time page traversal and stable read windows under concurrent write traffic.

### The problem and production context

Most applications begin with `Skip` and `Take`. While this pattern functions adequately for basic administrative pages with small datasets, it introduces structural performance degradation and data drift when datasets scale.

- **Failure scenario**: A client queries deep pages such as page 5,000 with a page size of 20. Under offset pagination, the database engine must scan, sort, and discard the first 100,000 matching rows before streaming the requested 20 rows, which produces linear degradation in execution time. Concurrently, if a new record is inserted or deleted while a user navigates from page 1 to page 2, the window shifts by one row, surfacing duplicate records or skipping records across page boundaries.
- **Why default approaches fall short**: Offset pagination relies on row offsets rather than immutable record identifiers. The database engine cannot jump directly to an arbitrary row number within a non-clustered index; it must traverse all preceding index entries up to the offset count.
- **Production impact**: As query depth grows, database engines suffer excessive memory pressure, CPU saturation, and disk I/O exhaustion. Concurrent write activity causes data drift, corrupting user-facing feeds and downstream data export pipelines.

### Mental model and core concepts

Selecting between offset and keyset pagination requires understanding how the database engine traverses ordered data and handles concurrent mutations.

#### 1. Offset traversal mechanics

Offset pagination defines a query window using the current page number and page size:

```csharp
var items = await context.Posts
    .AsNoTracking()
    .OrderByDescending(x => x.CreatedAt)
    .ThenByDescending(x => x.Id)
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();
```

The database executes this by ordering the entire matched result set, counting up to `(pageNumber - 1) * pageSize` rows, discarding them, and returning the requested count. It accepts simple `page` and `pageSize` parameters, supports direct random-access jumps (such as jumping directly to page 5 or 20), and integrates into frontend grid components. Its execution time degrades linearly as the offset increases.

#### 2. Keyset cursor seek mechanics

Cursor pagination (also called keyset pagination) tracks the specific column values of the last retrieved item instead of a numeric offset:

```csharp
var items = await context.Posts
    .AsNoTracking()
    .OrderByDescending(x => x.CreatedAt)
    .ThenByDescending(x => x.Id)
    .Where(x => x.CreatedAt < lastCreatedAt
        || (x.CreatedAt == lastCreatedAt && x.Id < lastId))
    .Take(pageSize)
    .ToListAsync();
```

Instead of counting and discarding rows, the database engine uses the cursor values to seek directly into the B-tree index. Query latency remains constant between page 1 and page 10,000. Because the seek points directly to immutable keys, insertions or deletions ahead of the cursor do not shift the current result set window.

#### 3. Deterministic sorting and tiebreakers

Both offset and keyset pagination require deterministic sorting. If the primary sort column contains duplicate values (such as identical timestamps), the database ordering becomes non-deterministic:

```csharp
.OrderBy(x => x.CreatedAt)
```

To establish a stable order, a guaranteed-unique column such as the primary key must be appended to the sort clause:

```csharp
.OrderBy(x => x.CreatedAt)
.ThenBy(x => x.Id)
```

Appending a unique secondary column prevents records with identical timestamps from shifting erratically across page boundaries.

#### 4. Index alignment

Keyset queries require composite index alignment matching both the `ORDER BY` and `WHERE` filter predicates. Without an index covering the exact sort and tiebreaker columns, the database engine falls back to a full table scan and in-memory sort, eliminating the performance advantage of keyset seek logic.

### Production implementation

Configure the composite index in Entity Framework Core and execute deterministic keyset pagination against the dataset.

```csharp
public class PostConfiguration : IEntityTypeConfiguration<Post>
{
    public void Configure(EntityTypeBuilder<Post> builder)
    {
        builder.HasKey(x => x.Id);

        builder.Property(x => x.CreatedAt)
            .IsRequired();

        builder.HasIndex(x => new { x.CreatedAt, x.Id });
    }
}

public sealed class PostService
{
    private readonly AppDbContext _context;

    public PostService(AppDbContext context)
    {
        _context = context;
    }

    public async Task<List<Post>> GetPostsCursorAsync(
        DateTime? lastCreatedAt,
        int? lastId,
        int pageSize,
        CancellationToken cancellationToken)
    {
        if (pageSize <= 0 || pageSize > 100)
        {
            throw new ArgumentOutOfRangeException(nameof(pageSize), "Page size must be between 1 and 100.");
        }

        IQueryable<Post> query = _context.Posts
            .AsNoTracking()
            .OrderByDescending(x => x.CreatedAt)
            .ThenByDescending(x => x.Id);

        if (lastCreatedAt.HasValue && lastId.HasValue)
        {
            DateTime cursorCreatedAt = lastCreatedAt.Value;
            int cursorId = lastId.Value;

            query = query.Where(x => x.CreatedAt < cursorCreatedAt
                || (x.CreatedAt == cursorCreatedAt && x.Id < cursorId));
        }

        return await query
            .Take(pageSize)
            .ToListAsync(cancellationToken);
    }
}
```

### Architectural trade-offs and edge cases

Choosing between offset and cursor pagination involves distinct trade-offs across query complexity, UI requirements, and data stability.

| Scenario | Offset pagination | Cursor pagination |
| :--- | :--- | :--- |
| Small datasets | Fast | Fast |
| Deep page offsets | Degrades linearly | Constant time |
| High write concurrency | Susceptible to duplicate/skipped items | Stable pagination window |
| Sequential navigation | Moderate | Optimal |

* **Latency versus consistency**: Offset pagination provides simple random-access navigation at the expense of linear query degradation and window drift under concurrent writes. Cursor pagination provides constant latency and window consistency across deep traversals, but requires clients to retain state and supply the last record's sort keys for each subsequent page.
* **Failure recovery**: Cursor pagination tolerates client-side disconnects and retries because passing the same cursor token returns the exact subsequent page slice, regardless of how many rows were inserted prior to the cursor point.
* **Scale limitations**: Offset pagination is bounded by table size and read volume; it remains appropriate for back-office admin grids, customer support ticket lists, and small configuration tables where users require explicit numbered page controls and direct random-access jumps. Keyset cursor pagination scales to millions of records and fits infinite-scroll interfaces, message streams, audit logs, and high-throughput public API endpoints.
* **Bi-directional navigation**: Cursor pagination does not support jumping directly to arbitrary page numbers (such as navigating directly to page 14). Bi-directional navigation requires reversing the sort predicates and reordering the result slice in application memory.

### Common anti-patterns and gotchas

* **Missing composite index**: Developers implement keyset `WHERE` clauses matching `CreatedAt` and `Id` without creating a matching composite index `(CreatedAt, Id)`. The database engine performs a full table scan and in-memory sort, eliminating all keyset performance advantages. Always define matching composite indexes in database migrations.
* **Non-deterministic sort ordering**: Developers sort exclusively by non-unique fields like `CreatedAt` or `Status`. When duplicate values span across page boundaries, rows are duplicated or omitted between queries. Always append a unique tiebreaker column such as the primary key.
* **Applying keyset pagination to arbitrary page jumping**: Developers attempt to compute arbitrary page numbers with keyset pagination by chaining multiple offset operations. Keyset pagination inherently requires sequential navigation; if random jumping to page N is an absolute business requirement, use bounded offset pagination with strict maximum page limits.
* **State shift in offset pagination**: Developers rely on `Skip` and `Take` for high-frequency activity feeds. New insertions push older items into subsequent offsets, forcing users to see duplicate items on every page scroll. Replace offset feeds with cursor pagination for all append-heavy event streams.

### Implementation checklist

1. Audit target entity queries to determine whether the UI requires random page access or sequential infinite scrolling.
2. Add a composite index covering both the primary sort column and the unique tiebreaker column in EF Core model configurations.
3. Replace `Skip` with keyset filter expressions using tuple comparison or compound `WHERE` clauses.
4. Append the entity primary key as the final deterministic tiebreaker in all `OrderBy` chains.
5. Validate that API input constraints enforce upper bounds on requested page size.
6. Verify database execution plans via `EXPLAIN` or SQL Server execution plans to confirm an index seek replaces index scans across deep page queries.