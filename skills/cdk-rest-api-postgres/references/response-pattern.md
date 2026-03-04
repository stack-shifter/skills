# Response Pattern

Use this reference when you need a reusable API response pattern for controllers and middleware.

## Goal

Return consistent API Gateway proxy responses from controllers and middleware.

## Shared Response Shape

Many repositories standardize responses through a helper such as `RestResult`.

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

- If the repository already has a helper such as `RestResult`, use it.
- If it does not, generate one shared response helper instead of repeating inline `APIGatewayProxyResult` objects.
- Let the shared helper own CORS and common headers.
- Check `fromDatabaseError(...)` or an equivalent helper before falling back to a generic 500 when repository code throws recognized SQL constraint failures.
