# A Generic Repository Pattern for NestJS with Mongoose for MongoDB

- Canonical URL: https://imzihad21.github.io/articles/a/a-generic-repository-pattern-for-nestjs-with-mongoose-for-mongodb-gp3/
- Source URL: https://dev.to/imzihad21/a-generic-repository-pattern-for-nestjs-with-mongoose-for-mongodb-gp3
- Web View: https://imzihad21.github.io/articles/a/a-generic-repository-pattern-for-nestjs-with-mongoose-for-mongodb-gp3/
- Published: 2024-11-03T18:29:54.000Z
- Modified: 2024-11-03T18:29:54.000Z
- Reading time: 3 minutes
- Tags: nestjs, mongodb, generic, repository

As applications grow, direct model usage across services quickly becomes messy. A generic repository gives one consistent place for CRUD behavior, error mapping, and query defaults.

This pattern keeps NestJS service logic cleaner while still using full Mongoose capability under the hood.

### Why It Matters

- Centralizes database access patterns for all collections.
- Improves consistency in error handling and logging.
- Keeps business services focused on domain logic.
- Provides reusable query behavior with strong typing.

### Core Concepts

#### 1. Generic Type-Safe Repository

The repository uses `<T extends Document>` so each model keeps proper typing.

#### 2. Duplicate Key Error Mapping

MongoDB duplicate key errors (`11000`) are mapped to `ConflictException` for predictable API responses.

#### 3. Lean Query Strategy

Read-heavy methods use `.lean()` where possible to reduce memory overhead.

#### 4. Centralized CRUD Operations

Create, read, update, delete, count, and ID validation are grouped in one abstraction.

#### 5. Deterministic Read Ordering

`getAll` applies default `createdAt` descending sort when caller does not provide one.

#### 6. Batch ObjectId Validation

`validateObjectIds` validates multiple IDs efficiently with one `$in` query.

### Practical Example

Generic repository implementation:

```typescript
import { ConflictException, Logger, NotFoundException } from "@nestjs/common";
import { ObjectId } from "mongodb";
import {
  Document,
  FilterQuery,
  FlattenMaps,
  Model,
  QueryOptions,
  SaveOptions,
  UpdateQuery,
  UpdateWithAggregationPipeline,
} from "mongoose";

export class GenericRepository<T extends Document> {
  private readonly internalLogger: Logger;
  private readonly internalModel: Model<T>;

  constructor(model: Model<T>, logger?: Logger) {
    this.internalModel = model;
    this.internalLogger = logger ?? new Logger(this.constructor.name);
  }

  async create(doc: Partial<T>, saveOptions: SaveOptions = {}): Promise<T> {
    try {
      const createdEntity = new this.internalModel(doc);
      return await createdEntity.save(saveOptions);
    } catch (error) {
      if (this.isDuplicateKeyError(error)) {
        this.internalLogger.error("Duplicate key error while creating", error as Error);
        throw new ConflictException("Document already exists with provided inputs");
      }

      throw error;
    }
  }

  async getAll(
    filter: FilterQuery<T> = {},
    options: QueryOptions = {}
  ): Promise<FlattenMaps<T>[]> {
    try {
      const queryOptions: QueryOptions = {
        ...options,
        sort: options.sort ?? { createdAt: -1 },
      };

      return await this.internalModel.find(filter, null, queryOptions).lean().exec();
    } catch (error) {
      this.internalLogger.error("Error finding entities", error as Error);
      return [];
    }
  }

  async getOneWhere(filter: FilterQuery<T>, options: QueryOptions = {}): Promise<T | null> {
    try {
      return await this.internalModel.findOne(filter, null, options).exec();
    } catch (error) {
      this.internalLogger.error("Error finding entity by filter", error as Error);
      return null;
    }
  }

  async getOneById(id: string, options: QueryOptions = {}): Promise<T | null> {
    try {
      return await this.internalModel.findById(id, null, options).exec();
    } catch (error) {
      this.internalLogger.error("Error finding entity by ID", error as Error);
      return null;
    }
  }

  async updateOneById(
    documentId: string,
    updated: UpdateWithAggregationPipeline | UpdateQuery<T>,
    options: QueryOptions = {}
  ): Promise<T> {
    try {
      const result = await this.internalModel
        .findOneAndUpdate(
          { _id: documentId },
          { ...updated, updatedAt: new Date() },
          { ...options, new: true }
        )
        .exec();

      if (!result) {
        throw new NotFoundException("Document not found with provided ID");
      }

      return result;
    } catch (error) {
      if (this.isDuplicateKeyError(error)) {
        this.internalLogger.error("Duplicate key error while updating", error as Error);
        throw new ConflictException("Document already exists with provided inputs");
      }

      this.internalLogger.error("Error updating one entity", error as Error);
      throw error;
    }
  }

  async removeOneById(id: string): Promise<boolean> {
    try {
      const { acknowledged } = await this.internalModel.deleteOne({ _id: id }).exec();
      return acknowledged;
    } catch (error) {
      this.internalLogger.error("Error removing entity", error as Error);
      throw error;
    }
  }

  async count(filter: FilterQuery<T> = {}): Promise<number> {
    try {
      return await this.internalModel.countDocuments(filter).exec();
    } catch (error) {
      this.internalLogger.error("Error counting documents", error as Error);
      throw error;
    }
  }

  async validateObjectIds(listOfIds: string[] = []): Promise<boolean> {
    try {
      if (!Array.isArray(listOfIds) || listOfIds.length === 0) {
        return false;
      }

      const objectIds = listOfIds.map((id) => new ObjectId(String(id)));

      const result = await this.internalModel
        .find({ _id: { $in: objectIds } })
        .select("_id")
        .lean()
        .exec();

      return listOfIds.length === result.length;
    } catch (error) {
      this.internalLogger.error("Error during ObjectId validation", error as Error);
      return false;
    }
  }

  private isDuplicateKeyError(error: unknown): boolean {
    return (
      typeof error === "object" &&
      error !== null &&
      "name" in error &&
      "code" in error &&
      (error as { name?: string; code?: number }).name === "MongoServerError" &&
      (error as { code?: number }).code === 11000
    );
  }
}
```

This gives you reusable persistence behavior and less repeated code across modules. Your services can stay calm and focused on business rules.

### Common Mistakes

- Duplicating repository logic separately for each model.
- Ignoring Mongo duplicate key mapping and leaking raw DB errors.
- Returning full Mongoose documents for all read flows when lean output is enough.
- Mixing business validation and persistence concerns in one method.
- Skipping ID batch validation before bulk operations.

### Quick Recap

- Generic repository pattern improves consistency and maintainability.
- Duplicate-key handling should map to explicit API exceptions.
- Lean reads reduce overhead in list/query endpoints.
- Update and delete paths should stay atomic and predictable.
- Batch ID validation helps protect downstream operations.

### Next Steps

1. Add pagination helpers (`limit`, `skip`, cursor strategy) for list endpoints.
2. Add soft-delete support with shared filters.
3. Add transaction-aware overloads using Mongo sessions.
4. Add repository-level metrics for latency and error rates.