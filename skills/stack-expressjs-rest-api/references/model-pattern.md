# Model Pattern

Keep the repository's model layout and Zod import style. These connected examples use Zod 4 and separate entities from validation.

```ts
// src/models/entities/item.model.ts
export interface Item {
  id: string;
  name: string;
  createdAt: string;
  modifiedAt: string;
}
```

```ts
// src/models/validation/item.validation.ts
import { z } from 'zod/v4';

export const itemPathSchema = z.object({
  itemId: z.uuid(),
});

export const itemQuerySchema = z.object({
  limit: z.coerce.number().int().positive().max(100).default(20),
  cursor: z.string().min(1).optional(),
});

export const itemPostDtoSchema = z.object({
  name: z.string().trim().min(1).max(128),
});

export const itemPutDtoSchema = z.object({
  id: z.uuid(),
  name: z.string().trim().min(1).max(128),
});

export type ItemPath = z.output<typeof itemPathSchema>;
export type ItemQuery = z.output<typeof itemQuerySchema>;
export type ItemPostDto = z.output<typeof itemPostDtoSchema>;
export type ItemPutDto = z.output<typeof itemPutDtoSchema>;
```

- Store parsed output in `response.locals.validated` and consume it in the controller. `safeParse()` does not update the original input.
- Keep one schema per validation target. Check repeated/nested query inputs against the actual parser configuration; do not assume a string.
- Defaults, coercions, and transforms are behavior: test that the resulting number, trimmed name, or default reaches application logic.
- Preserve established DTO names and schema layouts. Introduce fields and constraints only when the requested contract needs them.
- Validate cursor structure at the pagination boundary; a base64-looking string alone does not prove a valid cursor.
