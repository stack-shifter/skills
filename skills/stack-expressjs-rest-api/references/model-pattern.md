# Model Pattern

Use this reference when you need a reusable model and validation pattern.

## Goal

Create TypeScript types, DTOs, and Zod schemas for a domain entity in a model layout that matches the repository.

## Baseline Example

```ts
// src/models/entities/item.model.ts
export interface IItem {
  id?: string;
  name: string;
  createdAt?: string;
  modifiedAt?: string;
}

// src/models/query/item.query.ts
export interface IItemQuery {
  limit?: number;
  cursor?: string;
  sort?: 'asc' | 'desc';
}

// src/models/validation/item.validation.ts
import { z } from 'zod';

export const itemPathSchema = z.object({
  itemId: z.uuid(),
});

export const itemQuerySchema = z.object({
  sort: z.enum(['asc', 'desc']).optional(),
  limit: z.coerce.number().int().positive().max(100).optional(),
  cursor: z.string().optional(),
});

export const itemPostDtoSchema = z.object({
  name: z.string().trim().min(1).max(128),
});

export const itemPutDtoSchema = z.object({
  id: z.uuid(),
  name: z.string().trim().min(1).max(128),
});

export type ItemPostDto = z.infer<typeof itemPostDtoSchema>;
export type ItemPutDto = z.infer<typeof itemPutDtoSchema>;
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

- Use the repository's existing Zod import style first. Some repos use `zod`, others use `zod/v4`.
- If the repository already separates entities, DTOs, query types, and validation schemas, adapt to that pattern instead of forcing one-file-per-entity.
- If it does not, keeping related types and schemas together is a good default.
- Use `z.uuid()` for UUID fields, not `z.string().uuid()`
- Add schema transforms only when the repository already uses parsed output directly or when the normalized result is consumed immediately after validation
- Use `z.coerce.number()` for query params — they arrive as strings from Express
- Create separate schema objects per validation target: one for `id` params, one for query, one for body
- Expect validated and coerced values to be consumed from the repository's existing pattern, whether that is `response.locals.validated` or direct reads from the request object
