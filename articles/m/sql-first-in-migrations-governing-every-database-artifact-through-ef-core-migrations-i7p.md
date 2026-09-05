# SQL-First-in-Migrations: Governing Every Database Artifact Through EF Core Migrations

- Canonical URL: https://imzihad21.github.io/articles/a/sql-first-in-migrations-governing-every-database-artifact-through-ef-core-migrations-i7p/
- Source URL: https://dev.to/imzihad21/sql-first-in-migrations-governing-every-database-artifact-through-ef-core-migrations-i7p
- Web View: https://imzihad21.github.io/articles/a/sql-first-in-migrations-governing-every-database-artifact-through-ef-core-migrations-i7p/
- Published: 2026-05-12T11:12:11.000Z
- Modified: 2026-05-12T11:12:11.000Z
- Reading time: 6 minutes
- Tags: dotnet, efcore, codefirst, sql

### 1. Introduction

Entity Framework Core's code-first approach is frequently understood as limited to entity classes and their table mappings. A production database contains more than tables: views, functions, stored procedures, triggers, indexes that EF Core does not natively model, and baseline operational data. If these artifacts are created or managed outside the migration stream, the project loses the single source of truth that code-first promises.

The solution is to treat any SQL object on which the system depends as a first-class migration artifact. The pattern uses the EF Core migration timeline as the authoritative history for schema changes, SQL object changes, and baseline data, all driven from versioned SQL files.

### 2. The Core Idea

EF Core's `MigrationBuilder` already records every entity-model change as a C# migration. This pattern extends that single timeline to include every other database concern:

- Stored procedures
- Functions
- Views (including materialized views)
- Triggers
- Custom indexes or constraints not mapped by EF
- Seed or baseline data that the system requires to function

By routing all of these through `MigrationBuilder.Sql`, the migration order remains deterministic, the history auditable, and the database fully reproducible from source control.

### 3. The Pattern

The migration class remains thin, acting as an orchestrator that delegates SQL content to external files. The following example creates a stored procedure that calculates order totals:

```csharp
public partial class AddCalculateOrderTotalsStoredProc : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.Sql(
            SqlFileLoader.Read("StoredProcedures/CalculateOrderTotals/v1/Up.sql"));
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.Sql(
            SqlFileLoader.Read("StoredProcedures/CalculateOrderTotals/v1/Down.sql"));
    }
}
```

The `SqlFileLoader` helper reads raw SQL from embedded resources, decoupling SQL logic from C# string literals entirely:

```csharp
using System.Reflection;

namespace YourApp.Persistence.Helpers;

public static class SqlFileLoader
{
    private const string RootNamespace = "YourApp.Persistence.SqlScripts";

    public static string Read(string relativePath)
    {
        var resourceName = $"{RootNamespace}.{relativePath.Replace('\\', '.').Replace('/', '.')}";

        using var stream = Assembly.GetExecutingAssembly().GetManifestResourceStream(resourceName)
            ?? throw new FileNotFoundException($"Embedded SQL script not found: {resourceName}");

        using var reader = new StreamReader(stream);

        return reader.ReadToEnd();
    }
}
```

The versioned folder convention (`v1`, `v2`, ...) ensures that each deployed SQL file remains immutable; changes spawn a new version folder, preserving the full history.

### 4. Why This Works Better Than Inline SQL

Inline SQL embedded in C# migration classes creates several problems. SQL mixed with string concatenation is difficult to read and review. When a reviewer examines a diff, they must mentally separate C# logic from SQL. Complex procedural logic written in languages such as PL/pgSQL or T-SQL becomes painful to manage inside string literals. Rollback logic must be manually mirrored in the `Down` method, often leading to mistakes.

External SQL files solve these issues. The SQL code lives in its native syntax and benefits from syntax highlighting in editors. The `Up.sql` / `Down.sql` pairing makes reversibility explicit and testable. Database administrators can review pure SQL changes without opening a C# file. The separation also allows the same migration to be applied to different database providers by swapping the SQL file set.

### 5. Real-World Use Cases

#### 5.1 Stored Procedures

A stored procedure that aggregates order line items and updates the `total_amount` column on the `orders` table.

**Up.sql**
```sql
CREATE OR REPLACE PROCEDURE calculate_order_totals()
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE orders o
    SET total_amount = sub.total
    FROM (
        SELECT order_id, SUM(price * quantity * (1 - discount)) AS total
        FROM order_items
        GROUP BY order_id
    ) sub
    WHERE o.id = sub.order_id;
END;
$$;
```

**Down.sql**
```sql
DROP PROCEDURE IF EXISTS calculate_order_totals();
```

The migration references these files using `SqlFileLoader.Read("StoredProcedures/CalculateOrderTotals/v1/Up.sql")`.

#### 5.2 Functions

A scalar function that returns a formatted full name, used in reporting queries where EF Core's LINQ translation would be inefficient.

**Up.sql**
```sql
CREATE OR REPLACE FUNCTION get_user_full_name(first_name text, last_name text)
RETURNS text
LANGUAGE sql
IMMUTABLE
AS $$
    SELECT TRIM(first_name || ' ' || last_name);
$$;
```

**Down.sql**
```sql
DROP FUNCTION IF EXISTS get_user_full_name(text, text);
```

#### 5.3 Views

A view that filters out soft-deleted users, encapsulating the filtering logic so that multiple application queries do not repeat the condition.

**Up.sql**
```sql
CREATE VIEW active_users AS
SELECT id, email, first_name, last_name, created_at
FROM users
WHERE is_deleted = false;
```

**Down.sql**
```sql
DROP VIEW IF EXISTS active_users;
```

For materialized views, the `Up.sql` would include `CREATE MATERIALIZED VIEW` and any required index creation; the `Down.sql` drops the materialized view and its indexes, restoring the pre-migration state.

#### 5.4 Custom Indexes Not Modeled by EF

EF Core can model many indexes through attributes or the Fluent API. However, specialized indexes such as filtered indexes, partial indexes, or PostgreSQL GIN/GiST indexes often require raw SQL. The pattern handles them naturally:

```csharp
// Inside a migration Up method
migrationBuilder.Sql("CREATE INDEX CONCURRENTLY IF NOT EXISTS ix_events_timestamp ON events (timestamp);");
```

Because the C# class owns the migration timestamp, the index creation is sequenced relative to all other schema changes, eliminating the risk of race conditions that arise from running ad-hoc scripts.

#### 5.5 Seed / Baseline Data

Some data is not optional; the application requires a known set of product categories, configuration rows, or lookup values to function correctly. The pattern treats these as versioned SQL inserts.

**Up.sql**
```sql
INSERT INTO product_categories (id, name)
VALUES
    (1, 'Electronics'),
    (2, 'Books'),
    (3, 'Clothing')
ON CONFLICT (id) DO NOTHING;

-- Reset the sequence if using serial/bigserial primary keys
SELECT setval('product_categories_id_seq', (SELECT MAX(id) FROM product_categories));
```

**Down.sql**
```sql
DELETE FROM product_categories WHERE id IN (1, 2, 3);
```

The migration `SeedBaseCategories` places the SQL files in `SeedData/BaseCategories/v1/` and calls `SqlFileLoader.Read` in exactly the same way as any other artifact.

### 6. Recommended Project Structure

All SQL scripts reside under a `SqlScripts` folder inside the persistence project. They are embedded resources and accessed via the `SqlFileLoader`. The folder layout mirrors the artifact type and name.

```plaintext
YourApp.Persistence/
  Migrations/
    20260421062959_SeedBaseCategories.cs
    20260501000000_AddCalculateOrderTotalsStoredProc.cs
    20260505120000_AddGetUserFullNameFunction.cs
    20260510120000_AddActiveUsersView.cs
  SqlScripts/
    StoredProcedures/
      CalculateOrderTotals/
        v1/
          Up.sql
          Down.sql
    Functions/
      GetUserFullName/
        v1/
          Up.sql
          Down.sql
    Views/
      ActiveUsers/
        v1/
          Up.sql
          Down.sql
    SeedData/
      BaseCategories/
        v1/
          Up.sql
          Down.sql
  Helpers/
    SqlFileLoader.cs
```

Each version folder is immutable after deployment. When the logic changes, a new version folder (`v2`) is added and the corresponding migration references it.

### 7. Implementation Details

#### 7.1 SqlFileLoader Configuration

All `.sql` files must have their **Build Action** set to `Embedded resource`. The `RootNamespace` in `SqlFileLoader` must match the assembly's root namespace plus the folder path to the scripts. The helper converts path separators to dots to form the full resource name.

For multi-provider support, you can extend the helper to select a provider-specific file (for example, `Up.psql.sql` vs. `Up.mssql.sql`) based on `MigrationBuilder.ActiveProvider`.

#### 7.2 Migration Scaffolding Workflow

When adding a new SQL-backed artifact:

1. Create the migration shell: `dotnet ef migrations add AddMyNewView`.
2. Remove the auto-generated body from `Up()` and `Down()` if any.
3. Create the versioned SQL files in the appropriate folder under `SqlScripts`.
4. Call `SqlFileLoader.Read` inside the migration methods.
5. Test both `Up` and `Down` against a development database.

#### 7.3 Deterministic Rollback

Because every `Up.sql` is paired with a corresponding `Down.sql`, rolling back a migration is deterministic. This is difficult to guarantee when SQL statements are interleaved with C# operations. The pairing is critical for CI/CD pipelines that apply migrations in one direction and may need to reverse them in a disaster-recovery scenario.

### 8. When This Pattern Fits

The pattern is suitable when the project requires:

- EF migration ordering combined with native SQL power.
- Full source-controlled database history that includes tables, functions, procedures, views, and baseline data.
- Reliable environment parity across development, staging, and production.
- Code-first governance that extends beyond entity tables to all database objects the application uses.

This pattern does not replace EF's model-driven migrations. It complements them for the portion of the database surface that EF cannot model declaratively. The EF Core documentation itself recommends custom operations via `Sql()` as the extension point when built-in operations are insufficient.

### 9. Conclusion

The value of this pattern lies not in placing seed data inside a migration, but in treating SQL artifacts as first-class migration history. The migration timeline becomes the single source of truth for every database object the system depends on.

By keeping migration classes thin and delegating SQL to versioned files, teams gain:

- Clean, reviewable code diffs
- Native SQL syntax for complex logic
- Reversible, testable deployments
- A single, reproducible timeline of the entire database state

Whether the artifact is a stored procedure, a view, a function, or a set of baseline rows, the pattern scales naturally and keeps the project truly code-first, not just entity-first.

### References

- Microsoft Docs. "Custom Migrations Operations - EF Core." [learn.microsoft.com](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/operations) (see `MigrationBuilder.Sql`).
- EF Core Issues. "Adding Stored Procedures/View/Functions using Function within the migration flow." [GitHub/dotnet/efcore #34469](https://github.com/dotnet/efcore/issues/34469).
- EF Core Docs. "Managing Migrations - Custom DDL Examples." [GitHub/dotnet/EntityFramework.Docs #694](https://github.com/dotnet/EntityFramework.Docs/issues/694).
- Npgsql EF Core Provider. "Migrations and Schema Management." [DeepWiki](https://deepwiki.com/npgsql/efcore.pg/5-migrations-and-schema-management).
- Redgate. "Working with Flyway and Entity Framework Code First: An Overview." [Redgate Blog](https://www.red-gate.com/blog/database-devops/working-with-flyway-and-entity-framework-code-first-an-overview).
- SSW Rules. "Do you save each script as you go?" [SSW Rules](https://www.ssw.com.au/rules/save-each-script-as-you-go/).
- EFCore.MigrationExtensions. "NuGet Package for PostgreSQL SQL Objects in Migrations." [NuGet](https://www.nuget.org/packages/EFCore.MigrationExtensions.PostgreSQL/).