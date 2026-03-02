# Repository Pattern (Postgres)

Use this reference when the target repository uses Postgres and does not already have repository classes in `src/data/`.

For DynamoDB projects, use `references/repository-pattern.md` instead.

## Stack

- **ORM**: Drizzle ORM (`drizzle-orm`) — default for new projects
- **Driver**: `postgres` (postgres.js)
- **Migrations**: Drizzle Kit (`drizzle-kit`)

**ORM flexibility**: Drizzle is the default for new projects. If the target repository already uses a different ORM (Prisma, TypeORM, MikroORM, etc.), follow that instead — adapt the repository class shape and query style to match what is already there.

## Goal

Define a Drizzle schema, create a typed repository class implementing `Repository<T>`, and wire the Postgres connection in the dependency file.

## Schema Definition

```ts
// src/data/schema.ts
import { pgTable, uuid, text, timestamp, varchar } from 'drizzle-orm/pg-core';

export const items = pgTable('items', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 128 }).notNull(),
  description: text('description').notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
});

export type ItemRow = typeof items.$inferSelect;
export type NewItemRow = typeof items.$inferInsert;
```

## Repository Interface

```ts
// src/data/repository.interface.ts
import { CursorPagination } from './pagination';

export interface Repository<T> {
  findById(id: string): Promise<T | null>;
  findAll(query?: { cursor?: string; limit?: number }): Promise<CursorPagination<T>>;
  create(entity: T): Promise<T>;
  update(id: string, entity: Partial<T>): Promise<T>;
  delete(id: string): Promise<void>;
}
```

## Baseline Example

```ts
// src/data/item.repository.ts
import { eq, gt, asc } from 'drizzle-orm';
import { PostgresJsDatabase } from 'drizzle-orm/postgres-js';
import { Repository } from './repository.interface';
import { CursorPagination, decodeCursor, encodeCursor } from './pagination';
import { Item } from '../models/item.model';
import { items } from './schema';
import { loggerService } from '../dependencies/project.deps';
import { DatabaseError, NotFoundError } from '../utilities/errors';

export class ItemRepository implements Repository<Item> {
  private db: PostgresJsDatabase;

  constructor(db: PostgresJsDatabase) {
    this.db = db;
  }

  async findById(id: string): Promise<Item | null> {
    try {
      const rows = await this.db.select().from(items).where(eq(items.id, id)).limit(1);
      if (rows.length === 0) return null;
      return mapRowToItem(rows[0]);
    } catch (error: any) {
      loggerService.error('Error retrieving item', error);
      throw new DatabaseError('Error retrieving item');
    }
  }

  async findAll(query?: { cursor?: string; limit?: number }): Promise<CursorPagination<Item>> {
    const limit = query?.limit || 10;

    try {
      const cursorId = query?.cursor ? (decodeCursor(query.cursor).key as string) : undefined;

      const rows = await this.db
        .select()
        .from(items)
        .where(cursorId ? gt(items.id, cursorId) : undefined)
        .orderBy(asc(items.id))
        .limit(limit + 1); // fetch one extra to determine hasNextPage

      const hasNextPage = rows.length > limit;
      const pageRows = hasNextPage ? rows.slice(0, limit) : rows;
      const lastRow = pageRows[pageRows.length - 1];

      const result: CursorPagination<Item> = {
        limit,
        hasNextPage,
        items: pageRows.map(mapRowToItem),
      };

      if (hasNextPage && lastRow) {
        result.cursor = encodeCursor(lastRow.id);
      }

      return result;
    } catch (error: any) {
      loggerService.error('Error listing items', error);
      throw new DatabaseError('Error listing items');
    }
  }

  async create(entity: Item): Promise<Item> {
    try {
      const rows = await this.db
        .insert(items)
        .values({
          name: entity.name,
          description: entity.description,
        })
        .returning();
      return mapRowToItem(rows[0]);
    } catch (error: any) {
      loggerService.error('Error creating item', error);
      throw new DatabaseError('Error creating item');
    }
  }

  async update(id: string, entity: Partial<Item>): Promise<Item> {
    try {
      const rows = await this.db
        .update(items)
        .set({
          ...(entity.name !== undefined && { name: entity.name }),
          ...(entity.description !== undefined && { description: entity.description }),
          updatedAt: new Date(),
        })
        .where(eq(items.id, id))
        .returning();

      if (rows.length === 0) throw new NotFoundError('Item not found');
      return mapRowToItem(rows[0]);
    } catch (error: any) {
      if (error.name === 'NotFoundError') throw error;
      loggerService.error('Error updating item', error);
      throw new DatabaseError('Error updating item');
    }
  }

  async delete(id: string): Promise<void> {
    try {
      await this.db.delete(items).where(eq(items.id, id));
    } catch (error: any) {
      loggerService.error('Error deleting item', error);
      throw new DatabaseError('Error deleting item');
    }
  }
}

function mapRowToItem(row: typeof items.$inferSelect): Item {
  return {
    id: row.id,
    name: row.name,
    description: row.description,
    createdAt: row.createdAt.toISOString(),
    updatedAt: row.updatedAt.toISOString(),
  };
}
```

## Client Setup

The Drizzle client lives inside `src/data/` alongside the schema and repositories — not in `src/dependencies/`. The `src/dependencies/` folder is reserved for AWS clients (Cognito, DynamoDB, etc.).

```ts
// src/data/client.ts
import postgres from 'postgres';
import { drizzle } from 'drizzle-orm/postgres-js';
import * as schema from './schema';

const sql = postgres(process.env.DATABASE_URL!);

export const db = drizzle(sql, { schema });
```

Repositories import `db` directly from `client.ts`:

```ts
// src/data/item.repository.ts (top of file)
import { db } from './client';
```

Or receive it via constructor injection if the project uses a composition root pattern:

```ts
// src/dependencies/project.deps.ts
import { db } from '../data/client';
import { ItemRepository } from '../data/item.repository';

export const itemRepository = new ItemRepository(db);
```

## Drizzle Config

```ts
// drizzle.config.ts
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  schema: './src/data/schema.ts',
  out: './drizzle',
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
});
```

## Required Packages

```bash
npm install drizzle-orm postgres
npm install -D drizzle-kit
```

## Environment Variables

```dotenv
# .env.development
DATABASE_URL=postgres://postgres:password@localhost:5432/myapp_dev

# .env.production (or injected directly)
DATABASE_URL=postgres://user:pass@your-host:5432/myapp_prod
```

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

