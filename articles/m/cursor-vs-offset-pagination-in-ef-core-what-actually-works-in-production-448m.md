# Cursor vs Offset Pagination in EF Core: What Actually Works in Production

- Canonical URL: https://imzihad21.github.io/articles/a/cursor-vs-offset-pagination-in-ef-core-what-actually-works-in-production-448m/
- Source URL: https://dev.to/imzihad21/cursor-vs-offset-pagination-in-ef-core-what-actually-works-in-production-448m
- Web View: https://imzihad21.github.io/articles/a/cursor-vs-offset-pagination-in-ef-core-what-actually-works-in-production-448m/
- Published: 2026-03-19T10:18:33.000Z
- Modified: 2026-03-19T10:18:33.000Z
- Reading time: 2 minutes
- Tags: dotnet, efcore, pagination, performance

Pagination is easy to implement, but hard to get right when data grows.

Most applications start with `Skip/Take`. It works fine… until it doesn’t.

This article explains where each approach fits, what breaks, and how to choose properly.

---

### Why This Matters

When pagination is wrong, you will see:

* slow queries on large pages
* duplicate or missing rows
* unnecessary database load

The goal is not just pagination — it is **consistent and scalable data access**.

---

### 1. Offset Pagination (Skip/Take)

The most common approach:

```csharp
var items = await context.Posts
    .AsNoTracking()
    .OrderByDescending(x => x.CreatedAt)
    .ThenByDescending(x => x.Id)
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();
```

#### Why it works well

* simple and predictable
* supports page numbers (`page 1, 2, 3...`)
* easy to expose in APIs

#### Where it breaks

Offset pagination becomes inefficient as the offset grows.

Even if you request 20 rows, the database still processes all skipped rows before returning results.

Also, if data changes between requests:

* some rows may appear twice
* some rows may be skipped

Because pagination is based on **position**, not actual data.

---

### 2. Cursor Pagination (Keyset)

Cursor pagination moves through data using a reference point.

Instead of page number, you pass the last item from the previous result.

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

#### Why it works better for large data

* no row skipping
* uses index efficiently
* stable even if data changes

The database jumps directly to the correct position instead of scanning.

---

### 3. Ordering Must Be Stable

This is critical for both approaches.

Bad:

```csharp
.OrderBy(x => x.CreatedAt)
```

Good:

```csharp
.OrderBy(x => x.CreatedAt)
.ThenBy(x => x.Id)
```

Without a unique order, pagination will eventually return inconsistent results.

---

### 4. Performance Comparison

| Scenario                 | Offset       | Cursor |
| ------------------------ | ------------ | ------ |
| small dataset            | good         | good   |
| large page number        | slow         | stable |
| data frequently changing | inconsistent | stable |
| sequential navigation    | ok           | best   |

Cursor pagination avoids unnecessary work, especially at scale.

---

### 5. Limitations of Cursor Pagination

Cursor pagination is not a full replacement.

It does not support:

* jumping to arbitrary page
* clean page numbers
* accurate “page X of Y” navigation

Because it works with **positions**, not pages.

---

### 6. When to Use What

#### Use offset pagination when:

* UI requires page numbers
* users jump between pages
* dataset size is manageable

Example: admin dashboard

---

#### Use cursor pagination when:

* data is large
* users navigate sequentially
* consistency matters

Example: feeds, logs, activity streams

---

### 7. Practical Recommendation

In many systems, the best approach is:

* offset pagination → for page-based navigation
* cursor pagination → for next/previous navigation

Each solves a different problem. Using both is normal.

---

### 8. Indexing Matters

For cursor pagination to perform well:

```csharp
modelBuilder.Entity<Post>()
    .HasIndex(x => new { x.CreatedAt, x.Id });
```

Without proper indexing, both approaches degrade.

---

### Quick Recap

* Offset is simple but slows down with large offsets
* Cursor is efficient and stable but limited in navigation
* Always use a unique ordering
* Choose based on how users interact with data

---

### Final Thought

Offset solves UI navigation.
Cursor solves data traversal.

Pick based on the problem, not the implementation.