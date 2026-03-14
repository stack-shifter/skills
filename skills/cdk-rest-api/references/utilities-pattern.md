# Utilities Pattern

Use this reference when the repository needs shared runtime helpers that do not belong in controllers, repositories, or integration services.

## Goal

Centralize pure reusable helpers so handlers and controllers stay consistent.

## Recommended Utilities

- `rest-result.ts` for HTTP success and error responses
- `errors.ts` for application-specific error classes
- `status-code.ts` for named HTTP status constants
- cursor, encoding, date, or parsing helpers that are reused across repositories and controllers

## Baseline `RestResult` Rules

- set `Content-Type: application/json`
- set consistent CORS headers
- expose `Ok`, `Created`, `NoContent`, `BadRequest`, `NotFound`, `Unauthorized`, `Forbidden`, and `InternalServerError`
- include `fromPersistenceError(error)` when repository-level conflict mapping is needed

## Cursor and Pagination Helpers

When multiple repositories implement cursor-based pagination, centralize the shared mechanics instead of repeating them per repository.

For repository-backed pagination, prefer opaque cursor-based pagination when the API supports continuation tokens. A cursor often carries a stable ordered tuple such as `{ createdAt: string; id: string }`.

### `pagination.ts` shape

```ts
export const encodeCursor = (data: Record<string, unknown>): string =>
    Buffer.from(JSON.stringify(data)).toString('base64url');

export const decodeCursor = <T>(cursor: string): T =>
    JSON.parse(Buffer.from(cursor, 'base64url').toString('utf8')) as T;
```

### Repository usage pattern

Use `encodeCursor` / `decodeCursor` in the repository layer rather than exposing cursor internals to controllers:

```ts
type ProjectCursor = { createdAt: string; id: string };

async list(params: ListProjectsParams): Promise<PagedResult<IProject>> {
    const { limit, cursor } = params;
    const cursorData = cursor ? decodeCursor<ProjectCursor>(cursor) : undefined;

    const rows = await this.fetchProjectRows(cursorData, limit + 1);
    const hasNextPage = rows.length > limit;
    const pageRows = hasNextPage ? rows.slice(0, limit) : rows;
    const lastRow = pageRows[pageRows.length - 1];

    return {
        limit,
        nextCursor: hasNextPage
            ? encodeCursor<ProjectCursor>({ createdAt: lastRow.createdAt, id: lastRow.id })
            : null,
        items: pageRows.map((row) => this.mapProject(row)),
    };
}
```

## Guidance

- Prefer one shared response helper over repeated inline `APIGatewayProxyResult` objects.
- Put persistence-error translation in `RestResult` or a dedicated utility so controllers stay small.
- Keep utility functions pure where possible.
- Use custom error classes only when callers handle them differently from generic errors.
- Put `encodeCursor` / `decodeCursor` in a shared utility when more than one repository uses cursor pagination; keep the encoded cursor shape stable across deploys.
