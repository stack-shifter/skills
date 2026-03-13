# Repository Pattern (Postgres)

Use this reference when you need a reusable Postgres repository pattern for an Express API.

For DynamoDB projects, use `references/repository-pattern.md` instead.

## Stack

- **ORM**: Drizzle ORM (`drizzle-orm`) — default for new projects
- **Driver**: use the repository's existing Drizzle driver first (`pg` with `drizzle-orm/node-postgres` or `postgres` with `drizzle-orm/postgres-js`)
- **Migrations**: Drizzle Kit (`drizzle-kit`)

**ORM flexibility**: Drizzle is the default for new projects. If the target repository already uses a different ORM (Prisma, TypeORM, MikroORM, etc.), follow that instead — adapt the repository class shape and query style to match what is already there.

## Goal

Define a Drizzle schema, create a typed repository class implementing a shared repository contract, and wire the Postgres connection through one shared access pattern.

## Schema Definition

```ts
// src/data/db/schema/items.schema.ts
import { pgTable, timestamp, uuid, varchar } from 'drizzle-orm/pg-core';

export const items = pgTable('items', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 128 }).notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow().notNull(),
});

export type ItemRow = typeof items.$inferSelect;
export type NewItemRow = typeof items.$inferInsert;
```

## Repository Interface

```ts
// src/data/repositories/repository.interface.ts
import { Pagination } from '../pagination.interface';

export interface IRepository<TEntity, TFilter> {
  findById(id: string): Promise<TEntity | undefined>;
  find(filter?: TFilter): Promise<Pagination<TEntity>>;
  create(item: TEntity): Promise<TEntity>;
  update(item: TEntity): Promise<void>;
  remove(id: string): Promise<void>;
}
```

## Baseline Example

```ts
// src/data/repositories/item.repository.ts
import { asc, desc, eq, sql } from 'drizzle-orm';
import { Db } from '../db/client';
import { Pagination, decodeCursor, encodeCursor } from '../pagination.interface';
import { IRepository } from './repository.interface';
import { IItem } from '../models/entities/item.model';
import { IItemQuery } from '../models/query/item.query';
import { items } from '../db/schema';
import { NotFoundError } from './errors/not-found.error';

export class ItemRepository implements IRepository<IItem, IItemQuery> {
  constructor(private readonly db: Db) {
    // Store the shared database client for item queries.
  }

  async findById(id: string): Promise<IItem | undefined> {
    // Fetch a single item row by primary key.
    const [item] = await this.db.select().from(items).where(eq(items.id, id)).limit(1);

    return item ? this.mapItemRecord(item) : undefined;
  }

  async find(filter: IItemQuery = {}): Promise<Pagination<IItem>> {
    // Build a cursor-based item page query with deterministic ordering.
    const limit = filter.limit ?? 20;
    const sort = filter.sort ?? 'asc';
    const cursorData = filter.cursor ? decodeCursor<{ name: string; id: string }>(filter.cursor) : undefined;
    const orderBy = sort === 'desc' ? [desc(items.name), desc(items.id)] : [asc(items.name), asc(items.id)];
    const cursorFilter = cursorData
      ? sort === 'desc'
        ? sql<boolean>`(${items.name} < ${cursorData.name}) OR (${items.name} = ${cursorData.name} AND ${items.id} < ${cursorData.id})`
        : sql<boolean>`(${items.name} > ${cursorData.name}) OR (${items.name} = ${cursorData.name} AND ${items.id} > ${cursorData.id})`
      : undefined;
    const rows = await this.db
      .select()
      .from(items)
      .where(cursorFilter)
      .orderBy(...orderBy)
      .limit(limit + 1);
    const hasNextPage = rows.length > limit;
    const pageRows = hasNextPage ? rows.slice(0, limit) : rows;

    return {
      limit,
      nextCursor: hasNextPage
        ? encodeCursor({
            name: pageRows[pageRows.length - 1].name,
            id: pageRows[pageRows.length - 1].id,
          })
        : null,
      items: pageRows.map((row) => this.mapItemRecord(row)),
    };
  }

  async create(item: IItem): Promise<IItem> {
    // Insert an item row and return the hydrated record.
    const [created] = await this.db.insert(items).values({ name: item.name }).returning({ id: items.id });

    return (await this.findById(created.id))!;
  }

  async update(item: IItem): Promise<void> {
    // Update item fields and fail when the target row does not exist.
    const updated = await this.db
      .update(items)
      .set({
        name: item.name,
        updatedAt: item.modifiedAt ? new Date(item.modifiedAt) : new Date(),
      })
      .where(eq(items.id, item.id!))
      .returning({ id: items.id });

    if (updated.length === 0) {
      throw new NotFoundError('Item not found.');
    }
  }

  async remove(id: string): Promise<void> {
    // Delete an item row by id.
    await this.db.delete(items).where(eq(items.id, id));
  }

  private mapItemRecord(record: typeof items.$inferSelect): IItem {
    // Convert a database item row into the domain entity shape.
    return {
      id: record.id,
      name: record.name,
      createdAt: record.createdAt.toISOString(),
      modifiedAt: record.updatedAt.toISOString(),
    };
  }
}
```

## Client Setup Pattern

The Drizzle client can live alongside the schema and repositories or behind a composition module. Keep one coherent pattern for how repositories receive database access.

```ts
// src/data/db/client.ts
import { Pool } from 'pg';
import { drizzle } from 'drizzle-orm/node-postgres';
import * as schema from './schema';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 10,
});

export const db = drizzle({
  client: pool,
  schema,
});

export type Db = typeof db;
```

This repository also uses a shared database context to wire resource repositories once:

```ts
// src/data/context.ts
import { Db } from './db/client';
import { ItemRepository } from './repositories/item.repository';

export class DatabaseContext {
  readonly items: ItemRepository;

  constructor(db: Db) {
    this.items = new ItemRepository(db);
  }
}
```

## Drizzle Config

```ts
// drizzle.config.ts
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  schema: './src/data/db/schema/index.ts',
  out: './drizzle',
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
});
```

## Required Packages

```bash
npm install drizzle-orm pg
npm install -D drizzle-kit
```

## Environment Variables

```dotenv
# .env.development
DATABASE_URL=postgres://postgres:password@localhost:5432/myapp_dev

# .env.production (or injected directly)
DATABASE_URL=postgres://user:pass@your-host:5432/myapp_prod
```

## Guidance

- For UUID primary keys, do not paginate by `id` alone when using random UUID generation; order by a stable deterministic key such as `name + id` or `createdAt + id`
- If the repository already uses another ORM, adapt repository shape and transaction boundaries to that ORM rather than forcing Drizzle-specific patterns.
- If it does not yet have a repository layer, generate one reusable repository and shared DB access pattern instead of putting SQL in controllers.
- Keep the cursor payload aligned with the sort order so pagination remains deterministic
- If the repository already uses a `DatabaseContext` or equivalent composition object, extend it instead of exporting ad hoc repository singletons

## Migration Commands

```bash
# Generate migration files from schema changes
npx drizzle-kit generate

# Apply migrations to the database
npx drizzle-kit migrate

# Open Drizzle Studio (local DB browser)
npx drizzle-kit studio
```

## Docker Compose for Local Postgres

```yaml
# compose.yaml
services:
  postgres:
    image: postgres:17
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: myapp_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

## Guidance

- `DATABASE_URL` is the only required connection variable — use the standard postgres connection string format
- Always wrap Drizzle calls in `try/catch` and rethrow as `DatabaseError`; controllers do the HTTP mapping
- Use `.returning()` on `insert` and `update` to get the persisted row back without a second query
- The cursor pagination example uses `id`-based keyset pagination — adjust the cursor column to match the primary sort order of your query
- For transactions, use `db.transaction(async (tx) => { ... })`
- Run `drizzle-kit generate` after every schema change and commit the generated migration files
