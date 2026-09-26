# A Generic Repository Pattern for NestJS with Mongoose for MongoDB

- Canonical URL: https://imzihad21.github.io/articles/a/a-generic-repository-pattern-for-nestjs-with-mongoose-for-mongodb-gp3/
- Source URL: https://dev.to/imzihad21/a-generic-repository-pattern-for-nestjs-with-mongoose-for-mongodb-gp3
- Web View: https://imzihad21.github.io/articles/a/a-generic-repository-pattern-for-nestjs-with-mongoose-for-mongodb-gp3/
- Published: 2024-11-03T18:29:54.000Z
- Modified: 2024-11-03T18:29:54.000Z
- Reading time: 5 minutes
- Tags: nestjs, mongodb, generic, repository

## Generic repository pattern for NestJS and Mongoose

When every service in a NestJS application injects Mongoose models directly, database conventions scatter across modules. One developer calls `.lean()`, another forgets, and error handling fragments throughout the service layer.

A generic repository centralizes persistence logic, query defaults, and database error mapping into a reusable base class without removing Mongoose query flexibility.

### The problem and production context

Direct model injection throughout application services couples business logic to Mongoose query syntax and database-specific error signatures. When services bypass a centralized persistence contract, database failure handling differs across endpoints.

- **Failure scenario**: A service attempts an insert or update that violates a unique index constraint. Without centralized error translation, a raw `MongoServerError: E11000` bubbles up to the framework layer and triggers an unhandled HTTP 500 Internal Server Error instead of a structured client error.
- **Why default approaches fall short**: Individual services must catch database-specific exceptions, apply performance optimizations like `.lean()`, and validate ID formats before sending queries to MongoDB. Missing these checks causes scattered logic and inconsistent query behaviors.
- **Production impact**: Unhandled BSON cast exceptions and database constraint violations cause unpredictable 500 errors for callers, increase heap allocations during large read operations, and leak database infrastructure details across module boundaries.

Using a repository layer between NestJS services and Mongoose provides several operational advantages:
- Consistent error translation: Database engines use specific error codes. Catching MongoDB error `11000` in one place and rethrowing NestJS `ConflictException` guarantees uniform HTTP 409 responses across all modules.
- Predictable query performance: Applying `.lean()` by default on read queries avoids instantiating full Mongoose document instances when plain JavaScript objects are all the caller needs.
- Cleaner unit tests: Mocking a repository interface with methods like `findById` and `create` avoids mocking Mongoose multi-step chained query builders.
- Decoupled domain logic: Services focus on business rules, validation, and orchestration rather than query syntax and BSON type conversions.

### Mental model and core concepts

Centralizing persistence through a typed generic abstraction requires balancing type safety, input validation, and transparent error handling.

#### 1. Type safety with generics

The base class accepts a type parameter `T extends Document` and pairs it with Mongoose utility types such as `FilterQuery<T>` and `UpdateQuery<T>`. This preserves IDE autocomplete, property checking, and schema constraint enforcement on collection filters and updates.

#### 2. Guarding against BSON cast errors

Calling Mongoose query methods with malformed ObjectId strings throws runtime BSON cast errors. Validating inputs with `isValidObjectId()` before sending them to the database prevents unnecessary query execution and distinguishes bad input from missing documents.

#### 3. Avoiding swallowed infrastructure errors

Read queries must not hide infrastructure failures. Wrapping read queries in try-catch blocks that silently return `null` or `[]` causes database disconnections, network timeouts, or syntax errors to look like missing data. A repository lets unexpected infrastructure errors surface while cleanly translating expected database errors.

### Production implementation

The following generic repository provides type-safe CRUD operations, duplicate key error handling, and batch ID validation.

```typescript
import { ConflictException, Logger, NotFoundException } from "@nestjs/common";
import {
  Document,
  FilterQuery,
  FlattenMaps,
  isValidObjectId,
  Model,
  QueryOptions,
  SaveOptions,
  Types,
  UpdateQuery,
} from "mongoose";

export class GenericRepository<T extends Document> {
  protected readonly logger: Logger;
  protected readonly model: Model<T>;

  constructor(model: Model<T>, logger?: Logger) {
    this.model = model;
    this.logger = logger ?? new Logger(this.constructor.name);
  }

  async create(doc: Partial<T>, saveOptions?: SaveOptions): Promise<T> {
    try {
      const entity = new this.model(doc);
      return await entity.save(saveOptions);
    } catch (error) {
      this.handleDatabaseError(error, "Failed to create document");
      throw error;
    }
  }

  async find(
    filter: FilterQuery<T> = {},
    options: QueryOptions = {}
  ): Promise<FlattenMaps<T>[]> {
    const queryOptions: QueryOptions = {
      sort: { createdAt: -1 },
      ...options,
    };
    return this.model.find(filter, null, queryOptions).lean().exec() as Promise<FlattenMaps<T>[]>;
  }

  async findById(id: string, options: QueryOptions = {}): Promise<T | null> {
    if (!isValidObjectId(id)) {
      return null;
    }
    return this.model.findById(id, null, options).exec();
  }

  async findOne(filter: FilterQuery<T>, options: QueryOptions = {}): Promise<T | null> {
    return this.model.findOne(filter, null, options).exec();
  }

  async updateById(
    id: string,
    update: UpdateQuery<T>,
    options: QueryOptions = {}
  ): Promise<T> {
    if (!isValidObjectId(id)) {
      throw new NotFoundException(`Invalid document ID: ${id}`);
    }

    try {
      const updated = await this.model
        .findByIdAndUpdate(id, update, { ...options, new: true })
        .exec();

      if (!updated) {
        throw new NotFoundException(`Document with ID ${id} not found`);
      }

      return updated;
    } catch (error) {
      if (error instanceof NotFoundException) {
        throw error;
      }
      this.handleDatabaseError(error, `Failed to update document ${id}`);
      throw error;
    }
  }

  async deleteById(id: string): Promise<boolean> {
    if (!isValidObjectId(id)) {
      return false;
    }

    const result = await this.model.deleteOne({ _id: id }).exec();
    return result.deletedCount > 0;
  }

  async count(filter: FilterQuery<T> = {}): Promise<number> {
    return this.model.countDocuments(filter).exec();
  }

  async validateIds(ids: string[]): Promise<boolean> {
    if (!Array.isArray(ids) || ids.length === 0) {
      return false;
    }

    const validIds = ids.filter((id) => isValidObjectId(id));
    if (validIds.length !== ids.length) {
      return false;
    }

    const objectIds = validIds.map((id) => new Types.ObjectId(id));
    const count = await this.model.countDocuments({ _id: { $in: objectIds } }).exec();
    return count === ids.length;
  }

  private handleDatabaseError(error: unknown, contextMessage: string): void {
    if (
      typeof error === "object" &&
      error !== null &&
      "name" in error &&
      "code" in error &&
      (error as { name?: string; code?: number }).name === "MongoServerError" &&
      (error as { code?: number }).code === 11000
    ) {
      this.logger.warn(`${contextMessage}: Duplicate key constraint violated`);
      throw new ConflictException("A document with these unique fields already exists");
    }

    this.logger.error(`${contextMessage}: ${(error as Error)?.message || error}`);
  }
}
```

### Architectural trade-offs and edge cases

Abstracting Mongoose behind a repository enforces structural discipline while introducing specific architectural constraints.

* **Latency versus consistency**: Using `.lean()` strips document hydration, reducing memory consumption and garbage collection pauses during large queries. Lean documents lack Mongoose lifecycle middleware and virtual fields. When business logic relies on document methods or hooks, callers must execute hydrated queries instead.
* **Failure recovery**: Infrastructure failures such as database disconnections or network partitions propagate to NestJS global exception filters rather than hiding as missing data. Specific database errors like duplicate key violations (code 11000) convert into structured `ConflictException` instances for deterministic HTTP recovery.
* **Scale limitations**: For high-throughput write workloads or complex aggregation pipelines involving multi-collection joins (`$lookup`, `$facet`), generic CRUD signatures become limiting. Specialized queries and batch bulk writes (`bulkWrite`) belong in dedicated domain repositories extending the base class rather than inflating generic method signatures.

### Common anti-patterns and gotchas

* **Silent error suppression**: Catching read errors inside repositories and returning empty arrays turns database connection drops and query syntax failures into false empty states.
* **Corrupting update operators**: Manually attaching timestamp fields to update objects can conflict with Mongoose schema `{ timestamps: true }` options and invalidates MongoDB update operators like `$set`.
* **Neglecting lean queries**: Reading thousands of documents without `.lean()` allocates full Mongoose model instances for every record, substantially increasing heap memory usage.
* **Unvalidated ObjectId casting**: Passing arbitrary strings into `new Types.ObjectId()` throws unhandled BSON errors before your query reaches MongoDB. Always validate with `isValidObjectId()` prior to instantiation.

### Implementation checklist

1. Define concrete entity interfaces extending Mongoose `Document`.
2. Implement the `GenericRepository` base class with explicit type parameters and error handlers.
3. Inject domain-specific models into child repositories extending `GenericRepository`.
4. Validate single and batch identifier inputs using `isValidObjectId` before running queries.
5. Add pagination helpers supporting both page-based (`skip`/`limit`) and cursor-based strategies.
6. Incorporate transaction support by allowing a Mongoose `ClientSession` to pass through options into write operations.
7. Add soft-delete helpers that automatically filter out deleted records across find operations.