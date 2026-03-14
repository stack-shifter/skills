# Repository Pattern

Use this reference when an Express API needs persistence access and the repository needs a coherent boundary between controllers/services and the datastore.

## Goal

Keep controllers and services unaware of datastore mechanics by putting persistence access behind repositories and aggregating repositories behind one shared context when needed.

## Rules

- Controllers and services should ask for domain operations, not construct datastore requests.
- Repositories own query construction, key construction, transactions, and persistence-specific error handling.
- When multiple repositories are needed together, expose them through one aggregate context such as `DatabaseContext`, `DataContext`, or `Context`.
- Route handlers should depend on controllers or services, not on datastore clients directly.

## Repository Interface

```ts
// src/data/repository.interface.ts
import { CursorPagination } from './pagination';

export interface Repository<T, TQuery = { cursor?: string; limit?: number }> {
  findById(id: string): Promise<T | null>;
  findAll(query?: TQuery): Promise<CursorPagination<T>>;
  create(entity: T): Promise<T>;
  update(id: string, entity: Partial<T>): Promise<T>;
  delete(id: string): Promise<void>;
}
```

## Baseline Repository Shape

```ts
// src/data/repositories/item.repository.ts
import { Repository } from '../repository.interface';
import { CursorPagination } from '../pagination';
import { Item } from '../../models/item.model';
import { DatabaseError } from '../../utilities/errors';

export class ItemRepository implements Repository<Item> {
  constructor(private readonly db: RepositoryDependencies) {}

  async findById(id: string): Promise<Item | null> {
    try {
      const record = await this.fetchItemRecord(id);
      if (!record) return null;
      return this.mapToDomain(record);
    } catch (error) {
      throw new DatabaseError('Error retrieving item');
    }
  }

  async findAll(query?: { cursor?: string; limit?: number }): Promise<CursorPagination<Item>> {
    try {
      return await this.fetchItemPage(query);
    } catch (error) {
      throw new DatabaseError('Error listing items');
    }
  }

  async create(entity: Item): Promise<Item> {
    try {
      await this.insertItem(entity);
      return entity;
    } catch (error) {
      throw new DatabaseError('Error creating item');
    }
  }

  async update(id: string, entity: Partial<Item>): Promise<Item> {
    try {
      const updated = await this.updateItem(id, entity);
      if (!updated) throw new DatabaseError('Item not found');
      return updated;
    } catch (error) {
      throw new DatabaseError('Error updating item');
    }
  }

  async delete(id: string): Promise<void> {
    try {
      await this.deleteItem(id);
    } catch (error) {
      throw new DatabaseError('Error deleting item');
    }
  }
}
```

## Aggregate Context Pattern

When multiple repositories are used together, aggregate them behind one context object instead of exporting ad hoc repository singletons.

```ts
// src/data/context.ts
import { ItemRepository } from './repositories/item.repository';
import { UserRepository } from './repositories/user.repository';

export class Context {
  readonly items: ItemRepository;
  readonly users: UserRepository;

  constructor(db: RepositoryDependencies) {
    this.items = new ItemRepository(db);
    this.users = new UserRepository(db);
  }
}
```

## Composition Pattern

```ts
// src/dependencies/app.dependencies.ts
import { Context } from '../data/context';

const db = createRepositoryDependencies();

export const context = new Context(db);
export const itemService = new ItemService(context.items);
```

## Guidance

- If the repository already has a repository contract or dependency composition pattern, adapt this shape to it instead of introducing a second one.
- If it does not, generate one reusable repository and composition pattern rather than calling datastore clients directly from controllers.
- Repository methods should be named after domain operations or access patterns, not after raw persistence APIs.
- Keep mapping between persistence records and domain objects inside repositories or dedicated mappers, not in controllers.
- Keep pagination helpers and cursor encoding inside the data-access layer or shared utilities rather than assembling cursors in controllers.
- Translate persistence-specific errors into shared application errors at the repository boundary so controllers can stay HTTP-focused.
