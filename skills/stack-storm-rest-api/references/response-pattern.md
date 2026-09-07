# Response Pattern

Use this reference when you need a reusable API response pattern for controllers and middleware.

## Goal

Return consistent API Gateway proxy responses from controllers and middleware: stable status codes, CORS headers, and a uniform error body shape across all endpoints.

## When to Use

Load this reference when the repository does not already have a `RestResult`-style helper, or when you need to understand the full method surface of the existing one.

If the repository already has a `RestResult` class, use it — do not build a parallel response helper.

## Project-Local RestResult Class Shape

For this project-local pattern, use `RestResult` as a class of static factory methods. Each method returns an `APIGatewayProxyResult` with consistent headers. CORS origin is read from `process.env.CORS_ORIGIN` at call time, defaulting to `'*'`.

### Success Methods

```ts
RestResult.Ok(body: object)
RestResult.Accepted(body: object)
RestResult.NoContent()
RestResult.Created(body, resourceRoute)
RestResult.File(base64Body: string)
RestResult.MovedPermanently(newLocation)
```

### Error Methods

```ts
RestResult.BadRequest(message, details?)
RestResult.Unauthorized(message)
RestResult.Forbidden(message)
RestResult.NotFound(message)
RestResult.Conflict(message)
RestResult.InternalServerError(message)
RestResult.BadGateway(message)
RestResult.GatewayTimeout(message)
```

### Persistence Error Bridge

`fromPersistenceError(error: unknown): APIGatewayProxyResult | null` can map recognized repository-layer conflicts to HTTP responses so controllers do not need to inspect raw error shapes. Returns `null` for unrecognized errors so the caller can fall through to a generic 500.

```ts
try {
    await repository.save(item);
} catch (error) {
    const response = RestResult.fromPersistenceError(error);
    if (response) return response;
    return RestResult.InternalServerError("Unexpected error.");
}
```

Possible mappings:

| Error signal | Response |
|---|---|
| uniqueness violation from the repository layer | 409 Conflict |
| invalid reference or missing related record | 400 BadRequest |

## Error Body Shape

For a new API adopting this example contract, error responses follow this envelope. Preserve an existing API’s envelope instead:

```json
{
    "type": "VALIDATION_ERROR",
    "message": "Human-readable message",
    "status": 400,
    "details": { "field": "optional context" }
}
```

## Guidance

- Prefer one shared response helper over repeated inline `APIGatewayProxyResult` literals.
- Put repository-level conflict translation in `RestResult` or a dedicated helper so controllers stay small.
- Add only the methods the feature actually needs.
- Keep response helpers repository-agnostic.

## Storm Compatibility

Storm’s `RestResult` is a separate interface. Verify its installed methods and signatures; do not assume the project-local methods above exist in the library. The reviewed library uses problem-detail error bodies and `application/problem+json`. Switching helpers must not silently change status codes, error bodies, CORS behavior, or headers. Check parser, validation, and authorization failures as well as controller responses.
