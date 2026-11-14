# Building Dynamic Filters in Clean Architecture (CQRS) using ExpressionBuilder

- Canonical URL: https://imzihad21.github.io/articles/a/building-dynamic-filters-in-clean-architecture-cqrs-using-expressionbuilder-142/
- Source URL: https://dev.to/imzihad21/building-dynamic-filters-in-clean-architecture-cqrs-using-expressionbuilder-142
- Web View: https://imzihad21.github.io/articles/a/building-dynamic-filters-in-clean-architecture-cqrs-using-expressionbuilder-142/
- Published: 2026-03-25T16:01:12.000Z
- Modified: 2026-03-25T16:01:12.000Z
- Reading time: 5 minutes
- Tags: dotnet, cleanarchitecture, genericrepository, cqrs

## Building dynamic filters in Clean Architecture with ExpressionBuilder

Dynamic filtering in CQRS applications frequently leads to architectural leaks when handlers accept or return `IQueryable` interfaces. In Clean Architecture, query compilation and execution belong inside the infrastructure layer, while filter definitions belong in application contracts.

Using an `ExpressionBuilder<T>` utility composes dynamic predicate expressions into unified expression trees using parameter-rewriting visitors, ensuring complete architectural isolation without leaking database dependencies into domain handlers.

### The problem and production context

Constructing optional query filters across multiple API parameters introduces condition branching and parameter coupling.

- **Failure scenario**: An application handler naive-chains lambda expressions by combining expressions without rewriting parameter references, or by passing an `IQueryable` query across layer boundaries. EF Core throws an unhandled `InvalidOperationException` at runtime when the LINQ expression provider attempts to translate mismatched lambda parameter nodes into SQL queries.
- **Why default approaches fall short**: Direct variable reassignment overwrites previously evaluated filters instead of composing them. Alternatively, passing `IQueryable<T>` through the application layer exposes database engine abstractions, prevents unit testing handlers in isolation, and allows domain layers to trigger arbitrary deferred queries.
- **Production impact**: Unhandled runtime query translation failures crash API search endpoints, while leaky abstractions force EF Core package references into domain and application projects, violating dependency inversion boundaries.

Applying dynamic expression composition provides key structural benefits:
- Keeps query filtering logic inside the application layer without referencing EF Core packages.
- Allows repository methods to accept a single predicate expression instead of numerous optional filter parameters.
- Eliminates repetitive conditional statements across query handlers and services.
- Produces valid expression trees that LINQ providers can translate into parameterized SQL.

### Mental model and core concepts

Building composable predicate expressions across architectural layers requires parameter normalization and expression tree rewriting.

#### 1. Passing expression predicates across layer boundaries

The application layer defines what data is requested via strongly typed lambda expressions without dictating how the infrastructure layer executes or indexes the query:
```csharp
Task<IReadOnlyList<TEntity>> ListAsync(
    Expression<Func<TEntity, bool>>? filter = null,
    CancellationToken cancellationToken = default);
```
This contract establishes a strict boundary: the application layer constructs the expression tree, and the infrastructure repository evaluates it against the database provider.

#### 2. Failure modes of naive expression assignment

Naive optional filtering typically reassigns expression variables in sequence:
```csharp
Expression<Func<User, bool>>? filter = null;
if (request.IsActive.HasValue)
    filter = x => x.IsActive == request.IsActive.Value;
if (!string.IsNullOrWhiteSpace(request.Name))
    filter = x => x.Name.Contains(request.Name);
```
In this pattern, subsequent evaluations overwrite earlier criteria rather than joining them with logical `AND` operators.

#### 3. Expression collection and composition

`ExpressionBuilder<T>` maintains a sequence of independent predicate expressions and merges them using logical `AND` (`Expression.AndAlso`):
```csharp
var builder = new ExpressionBuilder<User>()
    .And(x => !x.IsDeleted)
    .And(x => x.Age >= 18 && x.Age <= 60)
    .And(x => x.Role == "Admin" || x.Role == "Manager");
```
Internal grouped precedence is preserved within each clause while separate clauses are joined sequentially at the root.

#### 4. Parameter rewriting via ExpressionVisitor

Each compiled lambda expression instantiates its own parameter symbol (`x => ...`, `y => ...`). Merging distinct lambda bodies directly fails because the expressions evaluate different parameter instances. `ExpressionBuilder<T>` uses an `ExpressionVisitor` subclass to rewrite all parameter nodes to reference a single unified `ParameterExpression`.

### Production implementation

The following implementation provides the complete `ExpressionBuilder<T>`, parameter rewriting visitor, CQRS query handler, and EF Core repository.

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Linq.Expressions;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.EntityFrameworkCore;

public sealed class ExpressionBuilder<T>
{
    private readonly List<Expression<Func<T, bool>>> _conditions = new();

    public ExpressionBuilder<T> And(Expression<Func<T, bool>> condition)
    {
        ArgumentNullException.ThrowIfNull(condition);

        _conditions.Add(condition);
        return this;
    }

    public ExpressionBuilder<T> AndIf(bool shouldAdd, Expression<Func<T, bool>> condition)
    {
        if (shouldAdd)
        {
            And(condition);
        }

        return this;
    }

    public Expression<Func<T, bool>>? Build()
    {
        if (_conditions.Count == 0)
        {
            return null;
        }

        var parameter = Expression.Parameter(typeof(T), "x");
        Expression combinedBody = RewriteBody(_conditions[0], parameter);

        for (var i = 1; i < _conditions.Count; i++)
        {
            var nextBody = RewriteBody(_conditions[i], parameter);
            combinedBody = Expression.AndAlso(combinedBody, nextBody);
        }

        return Expression.Lambda<Func<T, bool>>(combinedBody, parameter);
    }

    private static Expression RewriteBody(
        Expression<Func<T, bool>> sourceExpression,
        ParameterExpression targetParameter)
    {
        return new ReplaceParameterVisitor(sourceExpression.Parameters[0], targetParameter)
            .Visit(sourceExpression.Body)
            ?? throw new InvalidOperationException("Failed to rewrite filter expression.");
    }

    private sealed class ReplaceParameterVisitor : ExpressionVisitor
    {
        private readonly ParameterExpression _source;
        private readonly ParameterExpression _target;

        public ReplaceParameterVisitor(ParameterExpression source, ParameterExpression target)
        {
            _source = source;
            _target = target;
        }

        protected override Expression VisitParameter(ParameterExpression node)
        {
            return node == _source ? _target : base.VisitParameter(node);
        }
    }
}

public sealed record SearchUsersQuery(bool? IsActive, string? Name);

public sealed class SearchUsersQueryHandler
{
    private readonly IUserRepository _userRepository;

    public SearchUsersQueryHandler(IUserRepository userRepository)
    {
        _userRepository = userRepository;
    }

    public Task<IReadOnlyList<User>> Handle(SearchUsersQuery request, CancellationToken cancellationToken)
    {
        var filter = new ExpressionBuilder<User>()
            .And(x => !x.IsDeleted)
            .AndIf(request.IsActive.HasValue, x => x.IsActive == request.IsActive!.Value)
            .AndIf(!string.IsNullOrWhiteSpace(request.Name), x => x.Name.Contains(request.Name!))
            .Build();

        return _userRepository.ListAsync(filter, cancellationToken);
    }
}

public interface IUserRepository
{
    Task<IReadOnlyList<User>> ListAsync(
        Expression<Func<User, bool>>? filter = null,
        CancellationToken cancellationToken = default);
}

public sealed class UserRepository : IUserRepository
{
    private readonly AppDbContext _dbContext;

    public UserRepository(AppDbContext dbContext)
    {
        _dbContext = dbContext;
    }

    public async Task<IReadOnlyList<User>> ListAsync(
        Expression<Func<User, bool>>? filter = null,
        CancellationToken cancellationToken = default)
    {
        IQueryable<User> query = _dbContext.Users;

        if (filter is not null)
        {
            query = query.Where(filter);
        }

        return await query.ToListAsync(cancellationToken);
    }
}
```

The builder cleanly handles various predicate compositions:

A single required condition:
```csharp
var filter = new ExpressionBuilder<User>()
    .And(x => !x.IsDeleted)
    .Build();
```

Combining required and optional conditions:
```csharp
var filter = new ExpressionBuilder<User>()
    .And(x => !x.IsDeleted)
    .AndIf(request.IsActive.HasValue, x => x.IsActive == request.IsActive!.Value)
    .AndIf(!string.IsNullOrWhiteSpace(request.Name), x => x.Name.Contains(request.Name!))
    .Build();
```

Grouped logic inside one block:
```csharp
var filter = new ExpressionBuilder<User>()
    .And(x => !x.IsDeleted)
    .And(x => x.Role == "Admin" || x.Role == "Manager")
    .And(x => x.Age >= 18 && x.Age <= 60)
    .Build();
```

### Architectural trade-offs and edge cases

Composing expressions programmatically involves trade-offs between layer purity and query translation flexibility.

* **Latency versus consistency**: Building and visiting expression trees adds negligible CPU overhead in the application layer compared to the database query latency. Emitting `null` when no conditions exist allows the query planner to omit `WHERE` clauses completely, avoiding suboptimal SQL execution plans.
* **Failure recovery**: If invalid member accesses or unmappable custom functions are inserted into expression trees, EF Core throws a translation exception during SQL generation. Handlers should validate input bounds prior to building expressions to prevent malformed queries from reaching infrastructure.
* **Scale limitations**: When queries require complex multi-table navigations (`Include`), explicit projections (`Select`), or full-text search functions, simple predicate expressions become insufficient. Specification patterns or dedicated query handler queries should be utilized when operations exceed simple filter predicates.

### Common anti-patterns and gotchas

* **Expression reassignment overwrite**: Overwriting an expression variable sequentially instead of combining predicates drops earlier filter parameters.
* **Default true expressions**: Returning a fallback `x => true` expression instead of `null` causes EF Core to append `WHERE 1 = 1`, which can hinder query plan compilation on certain database providers.
* **Mismatched lambda parameters**: Combining lambda bodies without rewriting parameter references to a single `ParameterExpression` causes EF Core expression translation failures at runtime.
* **Leaking IQueryable across boundaries**: Exposing `IQueryable<T>` from repository contracts enables handlers to execute arbitrary, unindexed database queries and couples application layers directly to ORM assemblies.
* **Embedding database mechanics in handlers**: Writing direct SQL operations or EF Core-specific execution calls inside CQRS command or query handlers violates Clean Architecture boundaries.

### Implementation checklist

1. Implement `ExpressionBuilder<T>` and the `ReplaceParameterVisitor` inside an application or core utility project.
2. Update repository interfaces to accept `Expression<Func<TEntity, bool>>? filter = null` with explicit `CancellationToken` support.
3. Write unit tests covering empty filter cases, single predicates, and multi-condition scenarios.
4. Extend `ExpressionBuilder<T>` with an `Or` method if your domain requires root-level disjunction.
5. Inspect the SQL queries generated by EF Core in logging outputs to verify index usage across combined conditions.
6. Consult documentation on expression trees in C# for advanced visitor patterns when extending tree rewriting.