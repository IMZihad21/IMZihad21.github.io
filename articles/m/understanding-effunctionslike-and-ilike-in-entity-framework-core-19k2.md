# Understanding EF.Functions.Like and ILike in Entity Framework Core

- Canonical URL: https://imzihad21.github.io/articles/a/understanding-effunctionslike-and-ilike-in-entity-framework-core-19k2/
- Source URL: https://dev.to/imzihad21/understanding-effunctionslike-and-ilike-in-entity-framework-core-19k2
- Web View: https://imzihad21.github.io/articles/a/understanding-effunctionslike-and-ilike-in-entity-framework-core-19k2/
- Published: 2026-02-28T05:21:15.000Z
- Modified: 2026-02-28T05:21:15.000Z
- Reading time: 4 minutes
- Tags: dotnet, efcore, postgres, productivity

Pattern-based string search in Entity Framework Core often leads developers to apply standard .NET methods like `string.ToLower()` or LINQ string operations that wrap table columns in SQL scalar functions. This suppresses database index utilization, forces sequential table scans, or accidentally triggers client-side evaluation that transfers entire tables across the network.

Using `EF.Functions.Like` and PostgreSQL-specific `EF.Functions.ILike` translates pattern matching directly into native SQL relational operators, preserving index sargability and ensuring filtering executes entirely within the database engine.

### The problem and production context

Writing string search filters in LINQ without understanding relational expression translation degrades database engine throughput.

- **Failure scenario**: A search query executes `.Where(u => u.Email.ToLower().Contains(input.ToLower()))`; the database applies the `LOWER()` function to every row in a multi-million row table, causing a full table scan that consumes all database I/O bandwidth and times out API endpoints.
- **Why default approaches fall short**: Wrapping column references in functions renders queries non-sargable (unable to utilize standard B-tree indexes), while using `EF.Functions.ILike` on non-PostgreSQL database providers throws runtime translation exceptions.
- **Production impact**: Unindexed wildcard queries trigger database CPU spikes, API response latencies surge from milliseconds to seconds, and unescaped wildcard characters allow arbitrary wildcards in user search inputs.

Ensuring string filtering executes inside the database engine, preventing accidental client evaluation, providing explicit control over case sensitivity across providers, and keeping wildcard patterns maintainable within LINQ establishes scalable query patterns.

### Mental model and core concepts

Expression trees in EF Core map high-level C# expressions to provider-specific SQL abstract syntax tree nodes.

#### 1. SQL LIKE translation mechanics

`EF.Functions.Like(column, pattern)` translates directly to the SQL `LIKE` operator across all relational providers. The pattern supports standard SQL wildcard operators:
- `%`: Matches any sequence of zero or more characters.
- `_`: Matches exactly one character.

#### 2. PostgreSQL ILIKE operator

`EF.Functions.ILike(column, pattern)` is an extension provided by `Npgsql.EntityFrameworkCore.PostgreSQL`. It compiles directly to PostgreSQL's native `ILIKE` operator, providing case-insensitive pattern matching natively without wrapping columns in `LOWER()` and without modifying database-level collations.

#### 3. Comparison with standard string methods

Standard LINQ string methods map to specific SQL patterns:
- `Contains("term")` translates to `LIKE '%term%'`
- `StartsWith("term")` translates to `LIKE 'term%'`
- `EndsWith("term")` translates to `LIKE '%term'`

While standard methods handle fixed positions, `EF.Functions.Like` allows custom, composite wildcard combinations (such as `_a%` or `%prefix%suffix%`) inside a single parameterized expression.

#### 4. Provider and collation sensitivity

- `Like` conforms to the underlying column collation. On Microsoft SQL Server with default case-insensitive collations (e.g., `SQL_Latin1_General_CP1_CI_AS`), `Like` is case-insensitive. On PostgreSQL, default `LIKE` is strictly case-sensitive.
- `ILike` is strictly PostgreSQL-specific. Invoking `ILike` against SQL Server, SQLite, MySQL, or Oracle throws an `InvalidOperationException` during query translation.

### Production implementation

The following implementation demonstrates case-sensitive and case-insensitive pattern matching, wildcard queries, and safe repository search routines with input sanitization.

Standard wildcard queries using `EF.Functions.Like`:

```csharp
using System.Collections.Generic;
using System.Threading.Tasks;
using Microsoft.EntityFrameworkCore;

public static async Task<List<User>> QueryUsersWithLikeAsync(AppDbContext context)
{
    var users = await context.Users
        .Where(u => EF.Functions.Like(u.Name, "%John%"))
        .ToListAsync();

    var usersByPrefix = await context.Users
        .Where(u => EF.Functions.Like(u.Name, "J%"))
        .ToListAsync();

    var usersByPattern = await context.Users
        .Where(u => EF.Functions.Like(u.Name, "_a%"))
        .ToListAsync();

    return users;
}
```

PostgreSQL queries using `EF.Functions.ILike`:

```csharp
public static async Task<List<Product>> QueryProductsWithILikeAsync(AppDbContext context)
{
    var products = await context.Products
        .Where(p => EF.Functions.ILike(p.Description, "%widget%"))
        .ToListAsync();

    var productsByCategory = await context.Products
        .Where(p => EF.Functions.ILike(p.Category, "electronics"))
        .ToListAsync();

    return products;
}
```

Repository search method implementing wildcard escaping and case-insensitive search:

```csharp
using System;
using System.Collections.Generic;
using System.Text.RegularExpressions;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.EntityFrameworkCore;

public sealed class UserRepository(AppDbContext context)
{
    private static string EscapeLikePattern(string input)
    {
        return Regex.Replace(input, @"([%_\[\]])", @"\$1");
    }

    public async Task<List<User>> SearchUsersAsync(string query, CancellationToken cancellationToken)
    {
        if (string.IsNullOrWhiteSpace(query))
            return new List<User>();

        var sanitized = EscapeLikePattern(query.Trim());
        var pattern = $"%{sanitized}%";

        return await context.Users
            .Where(u => EF.Functions.ILike(u.Name, pattern) || EF.Functions.ILike(u.Email, pattern))
            .ToListAsync(cancellationToken);
    }
}
```

### Architectural trade-offs and edge cases

Choosing between native pattern matching, standard string methods, and full-text search introduces distinct architectural trade-offs.

* **Latency versus consistency**: `EF.Functions.Like` executes in-database with strong transactional read consistency, but leading wildcards (`%term`) force full table or index scans unless specialized indexes are created.
* **Failure recovery**: Queries executed against unsupported database providers fail at translation time. Applications supporting multiple database backends must abstract pattern generation or guard provider-specific calls via `context.Database.IsNpgsql()`.
* **Scale limitations**: For tables exceeding hundreds of thousands of rows, leading wildcard queries (`%term%`) degrade significantly on standard B-tree indexes. Scaling requires PostgreSQL `pg_trgm` GIN/GiST indexes or full-text search engines.

### Common anti-patterns and gotchas

* **Calling ToLower or ToUpper in LINQ queries**: Wrapping columns in `.ToLower()` causes EF Core to emit `LOWER(column)`, invalidating standard B-tree indexes and forcing full table scans. Use `EF.Functions.ILike` on PostgreSQL or provider collations.
* **Invoking ILike on non-PostgreSQL providers**: Calling `EF.Functions.ILike` on SQL Server, SQLite, or Oracle throws runtime translation exceptions because the operator is unique to the Npgsql provider. Use `EF.Functions.Like` with appropriate collations on non-PostgreSQL engines.
* **Unindexed leading wildcards on large tables**: Searching with `"%term%"` cannot use standard B-tree index prefixes and triggers sequential scans across large tables. Create trigram indexes (`pg_trgm`) or dedicated full-text catalogs.
* **Failing to escape wildcard characters in user input**: Passing raw user input directly into pattern interpolations allows users to enter `%` or `_` to execute arbitrary wildcard scans. Always sanitize or escape reserved wildcard symbols.

### Implementation checklist

1. Inspect existing LINQ queries for `.ToLower()` and `.ToUpper()` invocations and replace with `EF.Functions.Like` or `EF.Functions.ILike`.
2. Confirm the target database provider supports `EF.Functions.ILike` before deploying Npgsql-specific calls.
3. Profile search query performance using `EXPLAIN ANALYZE` on PostgreSQL or SQL Server execution plans.
4. Implement wildcard escaping routines to neutralize literal `%` and `_` characters in user input strings.
5. Create PostgreSQL `pg_trgm` GIN indexes on columns requiring leading wildcard searches (`LIKE '%term%'`).
6. Evaluate full-text search engines if requirements expand beyond substring and wildcard pattern matching.