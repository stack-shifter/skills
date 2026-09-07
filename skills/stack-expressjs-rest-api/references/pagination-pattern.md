# Pagination Pattern

Use this reference when you need a reusable pagination and cursor pattern.

## Goal

Create a shared pagination interface and cursor encoding helpers for paginated list endpoints.

## Baseline Example

```ts
// src/data/pagination.interface.ts

export interface Pagination<T> {
  limit: number;
  nextCursor: string | null;
  items: T[];
}

export const encodeCursor = <TKey>(key: TKey): string => {
  const jsonString = JSON.stringify(key);
  return Buffer.from(jsonString).toString('base64');
};

export const decodeCursor = <TKey>(encodedCursor: string): TKey => {
  const jsonString = Buffer.from(encodedCursor, 'base64').toString('utf-8');
  return JSON.parse(jsonString);
};
```

## Usage in a Repository

```ts
const cursorData = filter.cursor ? decodeCursor<{ name: string; id: string }>(filter.cursor) : undefined;
const rows = await this.db
  .select()
  .from(items)
  .where(cursorData ? sql<boolean>`(${items.name} > ${cursorData.name}) OR (${items.name} = ${cursorData.name} AND ${items.id} > ${cursorData.id})` : undefined)
  .orderBy(asc(items.name), asc(items.id))
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
  items: pageRows.map(mapRowToItem),
};
```

## Usage in a Controller

```ts
const limit = request.query['limit'] as string | undefined;
const cursor = request.query['cursor'] as string | undefined;

const paginated = await itemRepository.find({ limit: limit ? Number(limit) : undefined, cursor });
response.status(StatusCode.OK).json(paginated);
```

## Guidance

- If the repository already returns `nextCursor: null` when there is no next page, preserve that response shape
- If the repository already has pagination utilities, extend them instead of introducing a second cursor format.
- If it does not, generate one shared cursor pattern rather than encoding cursors ad hoc in each repository.
- Validate the cursor format in the model schema with `z.string().regex(/^[A-Za-z0-9+/=]+$/)` to catch malformed base64 early
- Cognito pagination tokens use `encodeURIComponent` / `decodeURIComponent` instead of base64 — use this pattern only for DynamoDB `LastEvaluatedKey` cursors
- Default limit and maximum cap should follow the repository's existing API contract
- For SQL-backed pagination, encode a stable ordered cursor key, not a random UUID by itself
