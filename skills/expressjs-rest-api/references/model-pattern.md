# Model Pattern

Use this reference when the target repository does not already have model files in `src/models/`.

## Goal

Create TypeScript types, DTOs, and Zod v4 schemas for a domain entity in one file.

## Baseline Example

```ts
// src/models/item.model.ts
import { z } from 'zod/v4';
import { BaseModel, BaseQuery } from './base.model';

// Domain type
export type Item = BaseModel & {
  name: string;
  description: string;
};

// DTOs
export type ItemPostDto = {
  name: string;
  description: string;
};

export type ItemPutDto = Omit<Item, 'createdAt' | 'updatedAt'>;

export type ItemQuery = BaseQuery;

// Zod schemas for validation middleware
export const itemIdSchema = z.object({
  id: z.uuid({ error: 'Invalid item ID format. Must be a valid UUID.' }),
});

export const itemQuerySchema = z.object({
  limit: z.coerce.number().max(60, { error: 'Limit maximum is 60.' }).optional(),
  cursor: z
    .string()
    .regex(/^[A-Za-z0-9+/=]+$/, { error: 'Invalid cursor format.' })
    .max(131072)
    .optional(),
});

export const itemPostDtoSchema = z.object({
  name: z.string().min(1, { error: 'Name is required.' }).max(128, { error: 'Name maximum length is 128.' }),
  description: z.string().min(1).max(512),
});

export const itemPutDtoSchema = z.object({
  id: z.uuidv4(),
  name: z.string().min(1).max(128),
  description: z.string().min(1).max(512),
});
```

## Base Model Reference

```ts
// src/models/base.model.ts
export type BaseModel = {
  id: string;
  createdAt?: string;
  updatedAt?: string;
};

export type BaseQuery = {
  limit?: number;
  cursor?: string;
};
```

## Guidance

- Use `from 'zod/v4'` — not `from 'zod'`
- Use `z.uuid()` for UUID fields, not `z.string().uuid()`
- Use `z.uuidv4()` in PUT schemas where the ID is part of the body
- Pass `{ error: '...' }` as the second argument to override default Zod messages
- Use `z.coerce.number()` for query params — they arrive as strings from Express
- Keep one file per entity: types, DTOs, and all Zod schemas together
- Create separate schema objects per validation target: one for `id` params, one for query, one for body
- Expect validated and coerced values to be consumed from `response.locals.validated`, especially for query parsing in Express 5
