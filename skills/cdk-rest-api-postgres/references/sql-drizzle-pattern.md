# SQL Drizzle Pattern

Use this reference when a change touches Postgres persistence and the repository needs a clear Drizzle-based pattern.

## Goal

Implement persistence with Postgres and Drizzle in a way that matches the repository's current architecture when it exists, or establishes a clean architecture when it does not.

## Common Shape

- shared Drizzle client in `src/data/db/client.ts`
- schema modules under `src/data/db/schema/`
- repository classes under `src/data/repositories/`
- shared repository wiring in `src/data/context.ts`
- `dbContext` exported from `src/app.ts` for runtime reuse

## Baseline Schema Example

```ts
import { index, pgTable, text, timestamp, uuid } from "drizzle-orm/pg-core";

export const clients = pgTable(
    "clients",
    {
        id: uuid("id").defaultRandom().primaryKey(),
        name: text("name").notNull(),
        createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
        updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
    },
    (table) => ({
        nameIdx: index("clients_name_idx").on(table.name),
    }),
);
```

## Baseline Repository Example

```ts
import { asc, eq } from "drizzle-orm";
import { Db } from "../db/client";
import { clients } from "../db/schema";

export class ClientRepository {
    constructor(private readonly db: Db) {}

    async getById(id: string) {
        return this.db.query.clients.findFirst({
            where: eq(clients.id, id),
        });
    }

    async query() {
        return this.db.select().from(clients).orderBy(asc(clients.name));
    }
}
```

## Wiring Example

```ts
import { db } from "./data/db/client";
import { DatabaseContext } from "./data/context";

export const dbContext = new DatabaseContext(db);
```

## Guidance

- Persistence work belongs under `src/data/` or an equivalent data layer, not in handlers.
- Reuse the shared `db` instance and `dbContext` or an equivalent composition pattern; do not create ad hoc connections.
- Model constraints and indexes in Drizzle where possible.
- Use repository methods that reflect domain behavior, not DynamoDB key access patterns.
- Treat `uuid().defaultRandom()` primary keys as identifiers, not chronological cursors.
- For cursor pagination over time-ordered results, order by a stable tuple such as `created_at` plus `id`, encode both values in the cursor, and add a matching composite index.
- Do not paginate with `WHERE id > :cursor ORDER BY id` or equivalent patterns when `id` is a UUID v4, because random UUID ordering does not reflect insertion time.
- When sorting by `created_at`, always include a unique tiebreaker such as `id` so pagination stays stable when multiple rows share the same timestamp.
- Generate migrations through the repository's Drizzle workflow or introduce one coherent migration workflow instead of writing unrelated SQL by hand.
- Do not reintroduce DynamoDB-shaped data modeling or dual writes.
