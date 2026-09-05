# EF Core 10's ExecuteUpdateAsync: Finally, Delegates That Don't Hate Developers

- Canonical URL: https://imzihad21.github.io/articles/a/ef-core-10s-executeupdateasync-finally-delegates-that-dont-hate-developers-kij/
- Source URL: https://dev.to/imzihad21/ef-core-10s-executeupdateasync-finally-delegates-that-dont-hate-developers-kij
- Web View: https://imzihad21.github.io/articles/a/ef-core-10s-executeupdateasync-finally-delegates-that-dont-hate-developers-kij/
- Published: 2025-08-27T05:59:08.000Z
- Modified: 2025-08-27T05:59:08.000Z
- Reading time: 2 minutes
- Tags: dotnet, csharp, efcore, lambda

## EF Core 10's ExecuteUpdateAsync Finally Uses Delegates Developers Can Actually Live With

Conditional batch updates in older EF versions often pushed developers into expression-tree gymnastics. It worked, but readability and maintenance were painful.

EF Core 10 improved this by allowing delegate-based setters in `ExecuteUpdateAsync`, which makes update logic much more natural.

### Why It Matters

- Makes conditional batch updates readable again.
- Removes most manual expression tree composition.
- Keeps compile-time safety with cleaner syntax.
- Reduces maintenance overhead in data access code.

### Core Concepts

#### 1. Old Pain: Expression Tree Composition

Before delegate-friendly update composition, conditional update logic often looked like this:

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

#### 2. EF Core 10 Delegate-Based Update Style

With delegate-friendly setters, logic becomes standard C#.

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

#### 3. Why This Is Better

- You can use normal control flow.
- Update logic is easier to review.
- Less custom helper code needed.

#### 4. Migration Shape

Main migration direction:

- Old: expression-tree composition-heavy update builders
- New: delegate-based setter composition in ordinary code

#### 5. SQL and Performance Expectations

`ExecuteUpdateAsync` still performs set-based SQL updates and avoids entity materialization.

#### 6. Refactor Opportunity

If your codebase built complex update-expression utilities, many can now be simplified or removed.

### Practical Example

```csharp
public async Task<int> UpdateBlogAsync(AppDbContext context, bool nameChanged)
{
    return await context.Blogs
        .Where(b => b.IsActive)
        .ExecuteUpdateAsync(s =>
        {
            s.SetProperty(b => b.Views, b => b.Views + 1);

            if (nameChanged)
            {
                s.SetProperty(b => b.Name, "foo");
            }

            return s;
        });
}
```

Cleaner updates with fewer expression hacks means fewer headaches during code reviews and fewer "who wrote this" moments.

### Common Mistakes

- Keeping old expression-tree boilerplate after upgrade.
- Forgetting `return s;` in block-bodied setter delegates.
- Assuming `ExecuteUpdateAsync` tracks entities like normal updates.
- Mixing batch updates with stale tracked entity assumptions.
- Skipping generated SQL verification during migration.

### Quick Recap

- EF Core 10 makes `ExecuteUpdateAsync` setter composition easier.
- Delegate-based style enables readable conditional logic.
- You can remove a lot of expression-tree plumbing.
- Batch updates remain set-based and efficient.
- Migration is mostly syntax and helper cleanup.

### Next Steps

1. Refactor existing batch-update helpers to delegate-based style.
2. Add tests that verify conditional setter behavior.
3. Validate generated SQL for critical updates.
4. Document team conventions for `ExecuteUpdateAsync` usage.

### References

- https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-10.0/breaking-changes
- https://learn.microsoft.com/en-us/ef/core/saving/execute-insert-update-delete
- https://github.com/dotnet/efcore/issues/32018