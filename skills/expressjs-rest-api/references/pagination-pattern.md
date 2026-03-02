# Pagination Pattern

Use this reference when the target repository does not already have pagination utilities in `src/data/`.

## Goal

Create a `CursorPagination` interface and base64 cursor encoding helpers for paginated list endpoints.

## Baseline Example

```ts
// src/data/pagination.ts

export interface CursorPagination<T> {
  limit: number;
  hasNextPage: boolean;
  items: T[];
  cursor?: string;
}

export interface Cursor {
  key: string | Record<string, any>;
}

export const encodeCursor = (key: string | Record<string, any>): string => {
  const jsonString = JSON.stringify({ key });
  return Buffer.from(jsonString).toString('base64');
};

export const decodeCursor = (encodedCursor: string): Cursor => {
  const jsonString = Buffer.from(encodedCursor, 'base64').toString('utf-8');
  return JSON.parse(jsonString);
};
```

## Usage in a Repository

```ts
const { Items, LastEvaluatedKey } = await this.dbContext.scan({
  TableName: process.env.DYNAMODB_TABLE,
  Limit: query?.limit || 10,
  ExclusiveStartKey: query?.cursor
    ? (decodeCursor(query.cursor).key as any)
    : undefined,
});

const result: CursorPagination<Item> = {
  limit: query?.limit || 10,
  hasNextPage: !!LastEvaluatedKey,
  items: (Items || []).map(mapDynamoItemToItem),
};

if (LastEvaluatedKey) {
  result.cursor = encodeCursor(LastEvaluatedKey);
}

return result;
```

## Usage in a Controller

```ts
const limit = request.query['limit'] ? parseInt(request.query['limit'] as string) : undefined;
const cursor = request.query['cursor'] as string | undefined;

const paginated = await itemRepository.findAll({ limit, cursor });
response.status(StatusCode.OK).json(paginated);
```

## Guidance

- `cursor` is omitted from the response when there is no next page — do not include it as `null` or `undefined` explicitly
- Validate the cursor format in the model schema with `z.string().regex(/^[A-Za-z0-9+/=]+$/)` to catch malformed base64 early
- Cognito pagination tokens use `encodeURIComponent` / `decodeURIComponent` instead of base64 — use this pattern only for DynamoDB `LastEvaluatedKey` cursors
- Default limit to `10` when not provided; cap at `60` in the Zod schema
