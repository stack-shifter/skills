# Repository Pattern

Keep persistence behind repositories, with one shared context when several repositories work together. Preserve the existing repository
contracts, classes, and `dependencies/` composition. Controllers orchestrate, services implement domain workflows and integrations, and
repositories own queries, transactions, record mapping, and persistence error translation.

## Repository Class

Reuse an existing repository interface when present; this concrete class needs no generic base contract. `ItemPersistence` stands in for the
project's datastore implementation so the example stays database-neutral. Adapt it to the actual driver/query code rather than introducing
a second CRUD layer. It returns mapped records, creates IDs/timestamps, uses stable pagination, and returns `null` for a missing update.

```ts
// src/data/repositories/item.repository.ts
import type { Item } from '../../models/entities/item.model.ts';
import type { ItemPostDto, ItemPutDto, ItemQuery } from '../../models/validation/item.validation.ts';
import type { Pagination } from '../pagination.interface.ts';
import { ConflictError, DatabaseError, NotFoundError, ValidationError } from '../../utilities/errors.ts';

export interface ItemPersistence {
  findById(id: string): Promise<Item | null>;
  findAll(query: ItemQuery): Promise<Pagination<Item>>;
  create(input: ItemPostDto): Promise<Item>;
  update(id: string, input: ItemPutDto): Promise<Item | null>;
  delete(id: string): Promise<void>;
}

export class ItemRepository {
  constructor(private readonly db: ItemPersistence) {}

  async findById(id: string): Promise<Item | null> {
    try {
      const item = await this.db.findById(id);
      return item;
    } catch {
      throw new DatabaseError('Error retrieving item');
    }
  }

  async findAll(query: ItemQuery): Promise<Pagination<Item>> {
    try {
      const page = await this.db.findAll(query);
      return page;
    } catch (error) {
      if (error instanceof ValidationError) throw error;
      throw new DatabaseError('Error listing items');
    }
  }

  async create(input: ItemPostDto): Promise<Item> {
    try {
      const item = await this.db.create(input);
      return item;
    } catch (error) {
      if (error instanceof ConflictError) throw error;
      throw new DatabaseError('Error creating item');
    }
  }

  async update(id: string, input: ItemPutDto): Promise<Item> {
    let item: Item | null;
    try {
      item = await this.db.update(id, input);
    } catch (error) {
      if (error instanceof ConflictError) throw error;
      throw new DatabaseError('Error updating item');
    }
    if (!item) throw new NotFoundError('Item not found');
    return item;
  }

  async delete(id: string): Promise<void> {
    try {
      await this.db.delete(id);
    } catch {
      throw new DatabaseError('Error deleting item');
    }
  }
}
```

This example treats an already-missing delete as complete. Preserve another established contract when required. Translate known driver
constraint failures to `ConflictError` where the real query runs; do not indiscriminately convert domain failures to generic database errors.
Preserve diagnostic causes through existing logging/error conventions without exposing driver details to clients.

## Shared Context and Composition

```ts
// src/data/context.ts
import { ItemRepository } from './repositories/item.repository.ts';
import type { ItemPersistence } from './repositories/item.repository.ts';

export class RepositoryContext {
  readonly items: ItemRepository;

  constructor(database: ItemPersistence) {
    this.items = new ItemRepository(database);
    // Add existing resource repositories here when the flow needs them.
  }
}
```

```ts
// src/dependencies/app.dependencies.ts
import { RepositoryContext } from '../data/context.ts';
import { database } from '../data/db/client.ts';

export const context = new RepositoryContext(database);
```

`data/db/client.ts` exports the configured database client that satisfies `ItemPersistence` in this example. Reuse the
project's existing database module when present. Initialize the client and `context` once after configuration loads. Provide working
queries and composition in generated projects. Compose services that use `context` in the same dependency module when a workflow needs them.

Use a service for workflows or external storage rather than moving datastore calls into a controller. Multi-row changes that form one
invariant belong in a transaction; cross-system work needs explicit failure/retry semantics. Consult
[storage-cleanup-pattern.md](storage-cleanup-pattern.md) only when the requested flow needs it.
