# EF Core 10's ExecuteUpdateAsync: Finally, Delegates That Don't Hate Developers

- Canonical URL: https://imzihad21.github.io/articles/a/ef-core-10s-executeupdateasync-finally-delegates-that-dont-hate-developers-kij/
- Source URL: https://dev.to/imzihad21/ef-core-10s-executeupdateasync-finally-delegates-that-dont-hate-developers-kij
- Web View: https://imzihad21.github.io/articles/a/ef-core-10s-executeupdateasync-finally-delegates-that-dont-hate-developers-kij/
- Published: 2025-08-27T05:59:08.000Z
- Modified: 2025-08-27T05:59:08.000Z
- Reading time: 4 minutes
- Tags: dotnet, csharp, efcore, lambda

## EF Core 10's ExecuteUpdateAsync supports delegate-based setters

Executing conditional batch updates in earlier versions of Entity Framework Core required complex, error-prone expression-tree composition. When updates involved dynamic columns or runtime branching, developers had to construct AST nodes manually or fall back to multiple round trips and raw SQL strings.

EF Core 10 introduces delegate-based setter composition in `ExecuteUpdateAsync`, allowing developers to express dynamic and conditional batch updates using standard C# control flow while generating atomic, set-based database statements.

### The problem and production context

Relational data pipelines frequently require updating subsets of table columns based on conditional application state (such as patch requests or state transitions).

- **Failure scenario**: An application receives a partial update request where only specific properties (such as blog titles or view counters) require mutation. In EF Core 7 through 9, building dynamic setters conditionally required manually composing nested `Expression` trees. A missing parameter mapping or malformed expression node caused runtime expression compilation failures (`InvalidOperationException`) or generated invalid SQL statements.
- **Why default approaches fall short**: While `ExecuteUpdateAsync` provided set-based updates without loading entities into memory, its parameter required an `Expression<Func<SetPropertyCalls<T>, SetPropertyCalls<T>>>`. Composing this expression dynamically required verbose Reflection calls and low-level expression parameter rebinding. Developers frequently reverted to loading entities into memory with the change tracker, degrading throughput and causing transaction locks under heavy load.
- **Production impact**: Complex expression-tree helper classes increased maintenance burden, slowed down feature delivery, and elevated the risk of runtime query translation exceptions in production environments.

### Mental model and core concepts

Understanding delegate-based setters requires contrasting manual expression-tree manipulation with compiler-driven delegate composition.

#### 1. Manual expression tree composition in earlier versions

Before delegate-based updates, building dynamic setters conditionally required low-level expression trees:

```csharp
Expression<Func<SetPropertyCalls<Blog>, SetPropertyCalls<Blog>>> setters =
    s => s.SetProperty(b => b.Views, 8);

if (nameChanged)
{
    var blogParameter = Expression.Parameter(typeof(Blog), "b");
    setters = Expression.Lambda<Func<SetPropertyCalls<Blog>, SetPropertyCalls<Blog>>>(
        Expression.Call(
            instance: setters.Body,
            methodName: nameof(SetPropertyCalls<Blog>.SetProperty),
            typeArguments: new[] { typeof(string) },
            arguments: new Expression[]
            {
                Expression.Lambda<Func<Blog, string>>(
                    Expression.Property(blogParameter, nameof(Blog.Name)),
                    blogParameter),
                Expression.Constant("foo")
            }),
        setters.Parameters);
}
```

This manual AST construction bypassed standard compiler safety checks, making refactoring brittle and code reviews error-prone.

#### 2. Delegate-based update syntax in EF Core 10

With delegate setters, conditional update logic uses standard C# control flow directly within a block-bodied lambda:

```csharp
await context.Blogs.ExecuteUpdateAsync(s =>
{
    s.SetProperty(b => b.Views, 8);

    if (nameChanged)
    {
        s.SetProperty(b => b.Name, "foo");
    }

    return s;
});
```

The delegate approach supports native branching constructs (`if`, `switch`), retains full compile-time type safety, and eliminates custom expression-manipulation utilities.

#### 3. Migration patterns

The upgrade path involves replacing manual expression trees:
- Earlier approaches: expression-tree composition and dynamic lambda builders.
- EF Core 10: delegate-based setter composition directly in application code.

If a codebase relies on custom expression-tree builders to support conditional updates, they can be refactored into standard delegates.

#### 4. SQL execution and performance expectations

`ExecuteUpdateAsync` continues to generate set-based SQL `UPDATE` statements executed directly on the database server. It avoids entity materialization, change tracking, and unnecessary read roundtrips.

### Production implementation

Execute conditional batch updates using EF Core 10 delegate syntax within a data access service.

```csharp
public sealed class Blog
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public int Views { get; set; }
    public bool IsActive { get; set; }
}

public sealed class BlogService
{
    private readonly AppDbContext _context;

    public BlogService(AppDbContext context)
    {
        _context = context;
    }

    public async Task<int> UpdateBlogMetricsAsync(
        int blogId,
        string? newName,
        bool incrementViews,
        CancellationToken cancellationToken)
    {
        if (blogId <= 0)
        {
            throw new ArgumentOutOfRangeException(nameof(blogId), "Blog ID must be a positive integer.");
        }

        return await _context.Blogs
            .Where(b => b.Id == blogId && b.IsActive)
            .ExecuteUpdateAsync(s =>
            {
                if (incrementViews)
                {
                    s.SetProperty(b => b.Views, b => b.Views + 1);
                }

                if (!string.IsNullOrWhiteSpace(newName))
                {
                    s.SetProperty(b => b.Name, newName);
                }

                return s;
            }, cancellationToken);
    }
}
```

### Architectural trade-offs and edge cases

Employing `ExecuteUpdateAsync` involves architectural trade-offs between raw execution speed and ORM lifecycle guarantees.

* **Latency versus consistency**: Set-based batch updates execute directly in the database, offering sub-millisecond execution times without the memory overhead of loading entities. However, `ExecuteUpdateAsync` bypasses the EF Core Change Tracker. Any tracked entities already loaded in the current DbContext instance will not reflect these database updates, risking stale-in-memory state within the same unit of work.
* **Failure recovery**: If a database connection fails during execution, standard database transaction boundaries apply. When chaining multiple batch operations, wrap calls inside an explicit `IDbContextTransaction` to ensure atomicity.
* **Scale limitations**: Batch updates excel at high-volume bulk modifications. However, because they bypass the Change Tracker, domain events, optimistic concurrency tokens (`RowVersion`), and client-side cascade rules defined in EF Core are not automatically triggered. If business logic requires domain event dispatching upon entity mutation, explicit event generation must be coordinated outside the query.

### Common anti-patterns and gotchas

* **Retaining expression tree boilerplate**: Upgrading to EF Core 10 while preserving legacy manual expression tree builders adds unnecessary cognitive overhead. Refactor to native delegate syntax.
* **Forgetting return statement in delegate blocks**: In block-bodied setter delegates, omitting `return s;` causes compile errors. Always return the setter instance.
* **Assuming Change Tracker synchronization**: Developers run `ExecuteUpdateAsync` and subsequently read modified entity instances from the local DbContext cache without reloading or detaching them, reading stale data. Always discard or refresh local context instances after batch updates.
* **Mixing batch updates with stale tracked entities**: Modifying an entity via `ExecuteUpdateAsync` while simultaneously calling `SaveChanges` on a tracked instance of the same entity overwrites the batch update or triggers concurrency conflicts.
* **Skipping SQL verification**: Failing to inspect the generated SQL during migration can lead to undetected query generation issues or suboptimal execution plans.

### Implementation checklist

1. Upgrade project dependencies to .NET 10 and EF Core 10 packages.
2. Identify existing manual expression tree builders used for dynamic `SetProperty` operations.
3. Refactor conditional update calls to block-bodied delegates with native `if` and `switch` statements.
4. Ensure block-bodied lambdas conclude with `return s;`.
5. Verify that `ExecuteUpdateAsync` invocations do not conflict with active entities in the local Change Tracker.
6. Verify generated SQL execution plans through EF Core logging or SQL profilers.
7. Add automated integration tests verifying both true and false conditional update branches against a live database.