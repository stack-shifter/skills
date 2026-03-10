# Utilities Pattern

Use this reference when the repository needs shared helpers that do not belong in handlers, repositories, or service integrations.

## Goal

Centralize reusable response, error, and helper logic so DynamoDB controllers and handlers stay consistent.

## Recommended Utilities

- `rest-result.ts` for API Gateway responses
- `errors.ts` for application-specific error classes
- `status-code.ts` for named HTTP status constants
- cursor and pagination helpers for encoding pagination state and sorting rows in memory
- key helpers when multiple repositories share the same PK/SK prefix conventions

## Baseline `RestResult` Rules

- set consistent JSON and CORS headers
- expose success and error constructors instead of repeating inline response objects
- prefer repository-agnostic error helpers for not found, validation, unauthorized, forbidden, and internal server errors

## Pagination and Cursor Helpers

When multiple repositories implement cursor-based pagination, centralize the shared mechanics instead of repeating them per repository. Two concerns belong in separate files:

- **`pagination.ts`** — generic row operations: encoding/decoding opaque cursor tokens, sorting rows with a direction, filtering rows after a cursor boundary, and slicing a page.
- **`cursor.ts`** — DynamoDB-specific helpers: encoding `LastEvaluatedKey` to a cursor string and decoding it back to `ExclusiveStartKey`.

Keeping these separate means pure row-manipulation logic stays independent of the DynamoDB SDK, which simplifies testing.

### `pagination.ts` shape

```ts
import { QueryCommandOutput } from '@aws-sdk/lib-dynamodb';
import { Sort } from '../models/enums/sort.enum';

/** DynamoDB cursor key shape used for ExclusiveStartKey / LastEvaluatedKey. */
export type DynamoCursorKey = NonNullable<QueryCommandOutput['LastEvaluatedKey']>;

/** Encode an arbitrary cursor payload into an opaque base64url string. */
export const encodeCursor = (data: Record<string, unknown>): string =>
    Buffer.from(JSON.stringify(data)).toString('base64url');

/** Decode an opaque base64url cursor payload back into a typed object. */
export const decodeCursor = <T>(cursor: string): T =>
    JSON.parse(Buffer.from(cursor, 'base64url').toString('utf8')) as T;

/** Apply sort direction to a comparator result. */
export const compareWithSort = (value: number, sort: Sort): number =>
    sort === Sort.DESC ? -value : value;

/** Sort rows using a shared comparator and sort direction. */
export const sortRows = <TRow>(
    rows: TRow[],
    comparator: (left: TRow, right: TRow) => number,
    sort: Sort,
): TRow[] => rows.sort((left, right) => compareWithSort(comparator(left, right), sort));

/** Continue rows strictly after cursor boundary for the current sort direction. */
export const filterRowsAfterCursor = <TRow, TCursor>(
    rows: TRow[],
    cursor: TCursor,
    comparator: (row: TRow, cursor: TCursor) => number,
    sort: Sort,
): TRow[] =>
    rows.filter((row) => {
        const cmp = comparator(row, cursor);
        return sort === Sort.DESC ? cmp < 0 : cmp > 0;
    });

/** Slice rows into a page and tell callers whether another page exists. */
export const paginateRows = <TRow>(
    rows: TRow[],
    limit: number,
): { hasNextPage: boolean; pageRows: TRow[] } => {
    const hasNextPage = rows.length > limit;
    return { hasNextPage, pageRows: hasNextPage ? rows.slice(0, limit) : rows };
};
```

### `cursor.ts` shape

```ts
import { DynamoCursorKey } from './pagination';

/** Encode a DynamoDB LastEvaluatedKey to an opaque cursor string. */
export const encodeDynamoCursor = (lastEvaluatedKey: DynamoCursorKey | undefined): string | undefined => {
    if (!lastEvaluatedKey) return undefined;
    return Buffer.from(JSON.stringify(lastEvaluatedKey), 'utf8').toString('base64url');
};

/** Decode an opaque cursor string back to a DynamoDB ExclusiveStartKey. */
export const decodeDynamoCursor = (cursor: string | undefined): DynamoCursorKey | undefined => {
    if (!cursor || cursor.trim().length === 0) return undefined;
    return JSON.parse(Buffer.from(cursor, 'base64url').toString('utf8')) as DynamoCursorKey;
};
```

### Repository usage pattern

Use `encodeDynamoCursor` / `decodeDynamoCursor` when the repository delegates pagination entirely to DynamoDB (simple key-range queries where DynamoDB's `LastEvaluatedKey` is sufficient).

Use `encodeCursor` / `decodeCursor` + `sortRows` + `filterRowsAfterCursor` + `paginateRows` when the repository fetches a broader result set and applies in-memory sorting or filtering — for example, when results span multiple GSI rows or require a custom comparator that DynamoDB cannot express as a key condition.

```ts
// Example: in-memory sorted pagination in a repository list method
import { encodeCursor, decodeCursor, sortRows, filterRowsAfterCursor, paginateRows } from '../db/utils/pagination';

type ClientCursor = { lastName: string; id: string };

async list(params: ListClientsParams): Promise<PagedResult<IClient>> {
    const { sort, limit, cursor } = params;
    const cursorData = cursor ? decodeCursor<ClientCursor>(cursor) : undefined;

    // Fetch all candidates (or a broad page) from DynamoDB
    const result = await this.db.send(new QueryCommand({ ... }));
    let rows = (result.Items ?? []) as ClientItem[];

    // Sort by the shared comparator + direction
    rows = sortRows(rows, (a, b) => {
        const last = a.lastName.localeCompare(b.lastName);
        return last !== 0 ? last : a.clientId.localeCompare(b.clientId);
    }, sort);

    // Advance past the cursor when continuing a previous page
    if (cursorData) {
        rows = filterRowsAfterCursor(rows, cursorData, (item, c) => {
            const last = item.lastName.localeCompare(c.lastName);
            return last !== 0 ? last : item.clientId.localeCompare(c.id);
        }, sort);
    }

    // Slice the page and build the next cursor from the last row
    const { hasNextPage, pageRows } = paginateRows(rows, limit);
    const lastRow = pageRows[pageRows.length - 1];

    return {
        limit,
        nextCursor: hasNextPage
            ? encodeCursor<ClientCursor>({ lastName: lastRow.lastName, id: lastRow.clientId })
            : null,
        items: pageRows.map((row) => this.mapClient(row)),
    };
}
```

## Guidance

- Prefer one shared response helper over duplicated `APIGatewayProxyResult` literals.
- Put pagination cursor encoding in a utility when several repositories share it.
- Separate generic row operations (`pagination.ts`) from DynamoDB key encoding (`cursor.ts`) so the row logic stays testable without AWS SDK dependencies.
- Add key helpers only when a PK or SK convention is used in multiple repositories; otherwise keep key construction local to the repository.
- Keep utility functions pure where possible.
