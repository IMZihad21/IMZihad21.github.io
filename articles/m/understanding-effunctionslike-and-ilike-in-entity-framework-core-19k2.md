# Understanding EF.Functions.Like and ILike in Entity Framework Core

- Canonical URL: https://imzihad21.github.io/articles/a/understanding-effunctionslike-and-ilike-in-entity-framework-core-19k2/
- Source URL: https://dev.to/imzihad21/understanding-effunctionslike-and-ilike-in-entity-framework-core-19k2
- Web View: https://imzihad21.github.io/articles/a/understanding-effunctionslike-and-ilike-in-entity-framework-core-19k2/
- Published: 2026-02-28T05:21:15.000Z
- Modified: 2026-02-28T05:21:15.000Z
- Reading time: 2 minutes
- Tags: dotnet, efcore, postgres, productivity

Pattern-based string search in EF Core should run in the database, not in application memory. `EF.Functions.Like` and `EF.Functions.ILike` help translate wildcard filtering into SQL-friendly expressions.

This guide explains when to use each, what they replace, and how they affect performance.

### Why It Matters

- Keeps filtering server-side for scalability.
- Avoids heavy client-side string processing.
- Improves control over case-sensitive/insensitive matching.
- Makes wildcard search behavior explicit in LINQ.

### Core Concepts

#### 1. What `Like` Does

`EF.Functions.Like` maps to SQL `LIKE` pattern matching.

- `%` matches any sequence of characters.
- `_` matches a single character.

#### 2. What `ILike` Does

`EF.Functions.ILike` performs case-insensitive pattern matching where supported (commonly PostgreSQL via Npgsql).

#### 3. Replacing Common String Methods

- `Contains("foo")` -> `Like(column, "%foo%")`
- `StartsWith("foo")` -> `Like(column, "foo%")`
- `EndsWith("foo")` -> `Like(column, "%foo")`

#### 4. Example with `Like`

```csharp
var users = await context.Users
    .Where(u => EF.Functions.Like(u.Name, "%John%"))
    .ToListAsync();

var usersByPrefix = await context.Users
    .Where(u => EF.Functions.Like(u.Name, "J%"))
    .ToListAsync();

var usersByPattern = await context.Users
    .Where(u => EF.Functions.Like(u.Name, "_a%"))
    .ToListAsync();
```

#### 5. Example with `ILike`

```csharp
var products = await context.Products
    .Where(p => EF.Functions.ILike(p.Description, "%widget%"))
    .ToListAsync();

var productsByCategory = await context.Products
    .Where(p => EF.Functions.ILike(p.Category, "electronics"))
    .ToListAsync();
```

#### 6. Provider and Collation Behavior

- `Like` behavior depends on database collation.
- `ILike` is provider-specific; check your EF provider support.
- Always validate generated SQL and execution plan.

### Practical Example

Search endpoint pattern:

```csharp
public Task<List<User>> SearchUsersAsync(string query)
{
    var pattern = $"%{query}%";

    return context.Users
        .Where(u => EF.Functions.ILike(u.Name, pattern) || EF.Functions.ILike(u.Email, pattern))
        .ToListAsync();
}
```

This keeps search logic readable and database-executed. Good for user-facing search boxes where case-insensitive behavior is expected.

### Common Mistakes

- Using `ToLower()`/`ToUpper()` in LINQ and damaging index usage.
- Assuming `ILike` is universally supported by all providers.
- Overusing `%term%` patterns on huge tables without strategy.
- Not checking query plan after adding wildcard searches.
- Falling back to client-side filtering for large datasets.

### Quick Recap

- Use `Like` for SQL wildcard matching.
- Use `ILike` for case-insensitive patterns where provider supports it.
- Keep matching on server side for performance.
- Prefix patterns (`term%`) are more index-friendly than `%term%`.
- Always test with real data volume and query plans.

### Next Steps

1. Benchmark `Like`/`ILike` queries with realistic table size.
2. Add proper indexes for frequent search fields.
3. Consider full-text search for heavy text-search workloads.
4. Standardize search pattern strategy across repositories.