# Response Pattern

Use this reference when you need a summary of the repository's API response conventions.

Read `src/utilities/rest-result.ts` first when changing runtime code.

## Goal

Return consistent API Gateway proxy responses from controllers and middleware.

## Repository Shape

This project standardizes responses through `RestResult`.

Common helpers:

- `RestResult.Ok(...)`
- `RestResult.Created(...)`
- `RestResult.NoContent()`
- `RestResult.BadRequest(...)`
- `RestResult.NotFound(...)`
- `RestResult.Unauthorized(...)`
- `RestResult.Forbidden(...)`
- `RestResult.InternalServerError(...)`
- `RestResult.fromDatabaseError(...)`

## Baseline Example

```ts
try {
    const client = await dbContext.clients.getById(clientId);
    if (!client) {
        return RestResult.NotFound("Id was not found in database.");
    }

    return RestResult.Ok(clientMapper.entityToResponseDto(client));
} catch (error: unknown) {
    const dbResult = RestResult.fromDatabaseError(error);
    if (dbResult) {
        return dbResult;
    }

    return RestResult.InternalServerError("Unable to perform operation on the item.");
}
```

## Guidance

- Prefer `RestResult` over inline `APIGatewayProxyResult` objects.
- Let `RestResult` own CORS and shared headers.
- Check `RestResult.fromDatabaseError(...)` before falling back to a generic 500 when repository code throws SQL constraint failures.
