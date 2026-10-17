# SQL-First-in-Migrations: Governing Every Database Artifact Through EF Core Migrations

- Canonical URL: https://imzihad21.github.io/articles/a/sql-first-in-migrations-governing-every-database-artifact-through-ef-core-migrations-i7p/
- Source URL: https://dev.to/imzihad21/sql-first-in-migrations-governing-every-database-artifact-through-ef-core-migrations-i7p
- Web View: https://imzihad21.github.io/articles/a/sql-first-in-migrations-governing-every-database-artifact-through-ef-core-migrations-i7p/
- Published: 2026-05-12T11:12:11.000Z
- Modified: 2026-05-12T11:12:11.000Z
- Reading time: 6 minutes
- Tags: dotnet, efcore, codefirst, sql

## SQL-first in migrations: governing every database artifact through EF Core migrations

Entity Framework Core code-first workflows frequently treat migrations as if they only cover entity classes and relational table mappings. In practice, production databases rely heavily on views, stored procedures, user-defined functions, triggers, vendor-specific indexes, and operational baseline data that cannot be modeled cleanly through the Fluent API. Managing these non-entity artifacts outside the migration pipeline creates configuration drift, untracked manual schema mutations, and broken deployment pipelines across environments.

Governing every database artifact through `MigrationBuilder.Sql` with embedded, versioned SQL files establishes the EF Core migration timeline as the single source of truth for all schema structures, procedural logic, and baseline reference data.

### The problem and production context

Treating database artifacts as secondary scripts executed manually or via out-of-band tooling disrupts deterministic schema deployment.

- **Failure scenario**: An application deployment runs automated EF Core table migrations, but an unmigrated dependent stored procedure or view is missing in the production database, causing runtime relational query failures when procedures reference non-existent columns.
- **Why default approaches fall short**: Embedding multiline raw SQL strings directly inside C# migration classes causes escaping syntax bugs, impairs code reviews, lacks SQL syntax highlighting, and frequently results in omitted or untested `Down()` rollback logic.
- **Production impact**: Rollbacks fail during automated CI/CD deployment aborts, environment provisioning requires manual database administrator interventions, and database logic drifts across staging and production instances.

Extending the `MigrationBuilder` timeline guarantees that views, stored procedures, functions, triggers, custom indexes, and baseline seed records execute in an auditable, deterministic sequence across every environment.

### Mental model and core concepts

A SQL-first migration pattern separates C# migration orchestration from raw SQL scripts while binding their lifecycles to the unified migration history.

#### 1. Unified migration timeline

EF Core tracks applied migrations in the `__EFMigrationsHistory` table. By wrapping raw DDL scripts inside `MigrationBuilder.Sql()`, custom SQL artifacts share the exact transactional and ordering guarantees as table alterations. Spinning up fresh environments from source control applies entity schemas and custom database logic in lockstep.

#### 2. Immutable versioned script hierarchy

SQL scripts reside in dedicated subdirectories organized by artifact category and version. Once merged and deployed, a version directory (such as `v1`) remains strictly immutable. Subsequent alterations introduce new version folders (`v2`), preserving historical progression and preventing retroactive history tampering:

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

#### 3. Embedded resource extraction

Embedding SQL files as assembly resources ensures scripts are compiled directly into the persistence assembly. At runtime during migration execution, `SqlFileLoader` resolves scripts from the manifest resource stream using predictable dot-delimited namespaces, avoiding external file path dependencies in containerized runners.

#### 4. Deterministic bidirectional execution

Every migration pairing an `Up.sql` with a matching `Down.sql` ensures full reversibility. Rollback logic is codified and verified in continuous integration pipelines, eliminating manual cleanup steps when reversing a release.

### Production implementation

The following implementation defines the embedded resource loader and coordinates migrations for stored procedures, functions, views, concurrent indexes, and baseline reference data.

The resource loader extracts SQL text from embedded assembly resources:

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

The C# migration class orchestrates deployment by loading versioned scripts:

```csharp
using Microsoft.EntityFrameworkCore.Migrations;
using YourApp.Persistence.Helpers;

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

The corresponding stored procedure scripts calculate line item totals and update order records:

`StoredProcedures/CalculateOrderTotals/v1/Up.sql`:

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

`StoredProcedures/CalculateOrderTotals/v1/Down.sql`:

```sql
DROP PROCEDURE IF EXISTS calculate_order_totals();
```

Scalar function scripts for reporting expressions that LINQ cannot translate efficiently:

`Functions/GetUserFullName/v1/Up.sql`:

```sql
CREATE OR REPLACE FUNCTION get_user_full_name(first_name text, last_name text)
RETURNS text
LANGUAGE sql
IMMUTABLE
AS $$
    SELECT TRIM(first_name || ' ' || last_name);
$$;
```

`Functions/GetUserFullName/v1/Down.sql`:

```sql
DROP FUNCTION IF EXISTS get_user_full_name(text, text);
```

Database view scripts standardizing soft-delete filtering across reporting tools:

`Views/ActiveUsers/v1/Up.sql`:

```sql
CREATE VIEW active_users AS
SELECT id, email, first_name, last_name, created_at
FROM users
WHERE is_deleted = false;
```

`Views/ActiveUsers/v1/Down.sql`:

```sql
DROP VIEW IF EXISTS active_users;
```

For materialized views, `Up.sql` executes `CREATE MATERIALIZED VIEW` alongside supporting indexes, while `Down.sql` drops the view cleanly.

Custom DDL for vendor-specific index creation not supported by Fluent API modeling:

```csharp
migrationBuilder.Sql("CREATE INDEX CONCURRENTLY IF NOT EXISTS ix_events_timestamp ON events (timestamp);");
```

Baseline reference data scripts applying static lookup records:

`SeedData/BaseCategories/v1/Up.sql`:

```sql
INSERT INTO product_categories (id, name)
VALUES
    (1, 'Electronics'),
    (2, 'Books'),
    (3, 'Clothing')
ON CONFLICT (id) DO NOTHING;

SELECT setval('product_categories_id_seq', (SELECT MAX(id) FROM product_categories));
```

`SeedData/BaseCategories/v1/Down.sql`:

```sql
DELETE FROM product_categories WHERE id IN (1, 2, 3);
```

### Architectural trade-offs and edge cases

Governing raw SQL through migrations introduces trade-offs between database-native optimizations and provider portability.

* **Latency versus consistency**: Executing procedural SQL and complex views inside migrations enforces strict transactional consistency during schema deployment, but large table backfills or unindexed view definitions can block schema migration transactions.
* **Failure recovery**: Migrations wrapped in a transaction roll back automatically on DDL failure where supported. Non-transactional statements such as PostgreSQL `CREATE INDEX CONCURRENTLY` cannot run inside an ambient migration transaction and require `suppressTransaction: true` in `migrationBuilder.Sql()`.
* **Scale limitations**: When supporting multiple database engines (such as PostgreSQL and SQL Server), SQL files must be duplicated per dialect or selected dynamically via `migrationBuilder.ActiveProvider` (e.g., `Up.psql.sql` and `Up.sqlserver.sql`).

### Common anti-patterns and gotchas

* **Embedding raw multiline SQL strings in C# classes**: Inlining raw SQL inside C# strings breaks IDE syntax validation, causes string escaping errors, and produces noisy Git pull request diffs. Keep SQL isolated in external `.sql` files.
* **Mutating existing versioned scripts after deployment**: Modifying an already-merged `v1/Up.sql` file causes new environments to deploy different logic than existing production instances. Always create a new version folder (`v2`) for schema modifications.
* **Omitting Down.sql rollback scripts**: Failing to define matching drop or delete logic in `Down()` prevents automated CI/CD rollback recovery. Always pair every `Up.sql` with a tested `Down.sql`.
* **Forgetting the embedded resource build action**: Leaving SQL files configured as content or none results in runtime `FileNotFoundException` during assembly execution. Ensure the project file sets the build action to `EmbeddedResource`.
* **Neglecting primary key sequence synchronization**: Inserting hardcoded primary key identifiers in seed scripts without synchronizing database sequences causes subsequent application inserts to crash on primary key conflict. Always call `setval()` or equivalent sequence reset logic.

### Implementation checklist

1. Scaffold an empty migration using `dotnet ef migrations add AddArtifactName`.
2. Clear auto-generated entity operations inside `Up()` and `Down()` methods.
3. Create versioned directories under `SqlScripts/{Category}/{ArtifactName}/v1/`.
4. Create `Up.sql` containing the forward DDL or DML logic.
5. Create `Down.sql` containing the reverse drop or rollback logic.
6. Set the Build Action of all `.sql` files to `EmbeddedResource` in the project file.
7. Call `SqlFileLoader.Read()` inside `Up()` and `Down()` passing the relative script paths.
8. Execute `dotnet ef database update` against a local test database.
9. Execute a rollback to the previous migration using `dotnet ef database update <PreviousMigrationName>` to verify the `Down()` script.
10. Re-apply the migration and verify that views, functions, procedures, and seed records function correctly.
11. Review references including Microsoft EF Core Custom Migrations Operations, GitHub dotnet/efcore issue 34469, EntityFramework.Docs issue 694, Npgsql EF Core Provider schema documentation, Redgate Flyway and EF Core guides, SSW script versioning rules, and EFCore.MigrationExtensions.PostgreSQL.