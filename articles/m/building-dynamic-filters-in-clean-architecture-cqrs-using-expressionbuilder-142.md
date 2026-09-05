# Building Dynamic Filters in Clean Architecture (CQRS) using ExpressionBuilder

- Canonical URL: https://imzihad21.github.io/articles/a/building-dynamic-filters-in-clean-architecture-cqrs-using-expressionbuilder-142/
- Source URL: https://dev.to/imzihad21/building-dynamic-filters-in-clean-architecture-cqrs-using-expressionbuilder-142
- Web View: https://imzihad21.github.io/articles/a/building-dynamic-filters-in-clean-architecture-cqrs-using-expressionbuilder-142/
- Published: 2026-03-25T16:01:12.000Z
- Modified: 2026-03-25T16:01:12.000Z
- Reading time: 4 minutes
- Tags: dotnet, cleanarchitecture, genericrepository, cqrs

## Building Dynamic Filters in Clean Architecture with ExpressionBuilder

Dynamic filtering becomes messy when the application layer must describe query rules without touching `IQueryable`.
This is common in Clean Architecture and CQRS, where query execution belongs to infrastructure.
In this article, you will see how `ExpressionBuilder<T>` helps compose safe dynamic predicates without breaking layer boundaries.

### Why It Matters

- Keeps filtering logic inside the application layer without leaking EF Core concerns.
- Lets repositories receive one predicate instead of many optional filter parameters.
- Reduces repetitive conditional code in handlers and services.
- Produces expressions that LINQ providers can translate correctly.

### Core Concepts

#### 1. Pass Expressions Instead of Queries

The application layer should describe filtering logic, not build database queries directly.

```csharp
Task<IReadOnlyList<TEntity>> ListAsync(
    Expression<Func<TEntity, bool>>? filter = null,
    CancellationToken cancellationToken = default);
```

This keeps the contract simple. Application builds the predicate. Infrastructure executes it.

#### 2. Reassigning Filters Does Not Scale

A naive approach usually overwrites previous conditions.

```csharp
Expression<Func<User, bool>>? filter = null;

if (request.IsActive.HasValue)
    filter = x => x.IsActive == request.IsActive.Value;

if (!string.IsNullOrWhiteSpace(request.Name))
    filter = x => x.Name.Contains(request.Name);
```

Only the last assignment survives. Once multiple optional filters appear, this pattern starts fighting you.

#### 3. ExpressionBuilder Collects Predicate Blocks

`ExpressionBuilder<T>` stores independent expression blocks and combines them with `AND`.

```csharp
var builder = new ExpressionBuilder<User>()
    .And(x => !x.IsDeleted)
    .And(x => x.Age >= 18 && x.Age <= 60)
    .And(x => x.Role == "Admin" || x.Role == "Manager");
```

The final predicate keeps the inner logic grouped while joining each block with `Expression.AndAlso`.

#### 4. Parameter Rewriting Makes the Final Expression Valid

Each lambda expression has its own parameter instance.

```csharp
x => x.IsActive
y => y.Age >= 18
```

These cannot be merged safely unless both bodies are rewritten to share the same parameter. That is why the builder uses a custom `ExpressionVisitor`.

### Practical Example

The main thing should stay the main thing, so start with the builder only.

#### `ExpressionBuilder<T>`

```csharp
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
            And(condition);

        return this;
    }

    public Expression<Func<T, bool>>? Build()
    {
        if (_conditions.Count == 0)
            return null;

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

    private sealed class ReplaceParameterVisitor(ParameterExpression source, ParameterExpression target) : ExpressionVisitor
    {
        private readonly ParameterExpression _source = source;
        private readonly ParameterExpression _target = target;

        protected override Expression VisitParameter(ParameterExpression node)
            => node == _source ? _target : base.VisitParameter(node);
    }
}
```

This builder has one job. It collects predicates, keeps them as separate blocks, then combines them into one final expression. That makes it easy to reason about and easy to reuse.

#### Minimal Example: One Required Condition

This is the smallest useful case.

```csharp
var filter = new ExpressionBuilder<User>()
    .And(x => !x.IsDeleted)
    .Build();
```

The final result is equivalent to:

```csharp
x => !x.IsDeleted
```

#### Minimal Example: Required and Optional Conditions

This is the common CQRS handler scenario. Some filters always apply, while others depend on request values.

```csharp
var filter = new ExpressionBuilder<User>()
    .And(x => !x.IsDeleted)
    .AndIf(request.IsActive.HasValue, x => x.IsActive == request.IsActive!.Value)
    .AndIf(!string.IsNullOrWhiteSpace(request.Name), x => x.Name.Contains(request.Name!))
    .Build();
```

This stays readable even when the request grows. The handler reads like business rules instead of if-statement plumbing.

#### Minimal Example: Grouped Logic Inside One Block

You can keep more complex logic inside a single predicate block.

```csharp
var filter = new ExpressionBuilder<User>()
    .And(x => !x.IsDeleted)
    .And(x => x.Role == "Admin" || x.Role == "Manager")
    .And(x => x.Age >= 18 && x.Age <= 60)
    .Build();
```

The builder joins blocks with `AND`, but it does not break the logic inside each block. That detail is important because real filters are rarely one simple comparison.

#### Using It in a Query Handler

In the application layer, the handler builds the predicate and sends it to the repository.

```csharp
public sealed record SearchUsersQuery(bool? IsActive, string? Name);

public sealed class SearchUsersQueryHandler(IUserRepository userRepository)
{
    private readonly IUserRepository _userRepository = userRepository;

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
```

This keeps the application layer focused on what should be filtered, not how the database query is constructed.

#### Repository Side

The repository receives the composed expression and applies it only if it exists.

```csharp
public interface IUserRepository
{
    Task<IReadOnlyList<User>> ListAsync(
        Expression<Func<User, bool>>? filter = null,
        CancellationToken cancellationToken = default);
}

public sealed class UserRepository(AppDbContext dbContext) : IUserRepository
{
    private readonly AppDbContext _dbContext = dbContext;

    public async Task<IReadOnlyList<User>> ListAsync(
        Expression<Func<User, bool>>? filter = null,
        CancellationToken cancellationToken = default)
    {
        IQueryable<User> query = _dbContext.Users;

        if (filter is not null)
            query = query.Where(filter);

        return await query.ToListAsync(cancellationToken);
    }
}
```

If `Build()` returns `null`, the repository skips `.Where(...)`. If it returns a predicate, EF Core receives one clean expression tree instead of scattered condition logic.

### Common Mistakes

- Reassigning the filter variable and losing previously added conditions.
- Returning `x => true` everywhere instead of allowing `null` for no filter.
- Combining expression bodies without rewriting parameters first.
- Exposing `IQueryable` to the application layer just to support dynamic filters.
- Mixing repository execution details into handlers.

### Quick Recap

- Clean Architecture works better when application passes predicates, not queries.
- `ExpressionBuilder<T>` composes optional filters into one expression.
- `AndIf` keeps conditional filtering compact and readable.
- Inner `AND` and `OR` logic stays grouped inside each predicate block.
- Parameter rewriting is what makes the final expression safe for EF Core translation.

### Next Steps

1. Add unit tests for zero, one, and multiple conditions.
2. Extend the builder with `Or` support if your use case needs it.
3. Check generated SQL for your heavier predicates in EF Core.
4. Review the official docs for more expression tree details: https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/