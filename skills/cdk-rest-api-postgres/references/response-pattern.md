# Response Pattern

Use this reference when you need a reusable API response pattern for controllers and middleware.

## Goal

Return consistent API Gateway proxy responses from controllers and middleware: stable status codes, CORS headers, and a uniform error body shape across all endpoints.

## When to Use

Load this reference when the repository does not already have a `RestResult`-style helper, or when you need to understand the full method surface of the existing one.

If the repository already has a `RestResult` class, use it — do not build a parallel response helper.

## RestResult Class Shape

Implement `RestResult` as a class of static factory methods. Each method returns an `APIGatewayProxyResult` with consistent headers. CORS origin is read from `process.env.CORS_ORIGIN` at call time, defaulting to `'*'`.

### Success Methods

```ts
RestResult.Ok(body: object)               // 200 — standard success
RestResult.Accepted(body: object)         // 202 — async job accepted
RestResult.NoContent()                    // 204 — mutation succeeded, no body needed
RestResult.Created(body, resourceRoute)   // 201 — resource created; sets Location header
RestResult.File(base64Body: string)       // 200 — binary response; isBase64Encoded: true, Content-Type: application/octet-stream
RestResult.MovedPermanently(newLocation)  // 301 — redirect; sets Location header
```

### Error Methods

```ts
RestResult.BadRequest(message, details?)  // 400 — validation failure; optional details object
RestResult.Unauthorized(message)          // 401 — missing or invalid credentials
RestResult.Forbidden(message)             // 403 — authenticated but not allowed
RestResult.NotFound(message)              // 404 — resource does not exist
RestResult.Conflict(message)              // 409 — uniqueness or state conflict
RestResult.InternalServerError(message)   // 500 — unexpected failure
RestResult.BadGateway(message)            // 502 — upstream service failure
RestResult.GatewayTimeout(message)        // 504 — upstream timeout
```

### Database Error Bridge

`fromDatabaseError(error: unknown): APIGatewayProxyResult | null` maps recognized Postgres constraint errors to HTTP responses so controllers do not need to inspect raw error shapes. Returns `null` for unrecognized errors so the caller can fall through to a generic 500.

```ts
try {
    await repository.save(item);
} catch (error) {
    const response = RestResult.fromDatabaseError(error);
    if (response) return response;
    return RestResult.InternalServerError('Unexpected error.');
}
```

Recognized mappings:

| Error signal | Response |
|---|---|
| `error.code === 'UNIQUE_VIOLATION'` (node-postgres / Drizzle) | 400 BadRequest — "conflicts with an existing record" |
| `error.code === 'FOREIGN_KEY_VIOLATION'` (node-postgres / Drizzle) | 400 BadRequest — "references a record that does not exist" |

## Error Body Shape

All error responses follow the same envelope:

```json
{
    "type": "VALIDATION_ERROR",
    "message": "Human-readable message",
    "status": 400,
    "details": { "field": "optional context" }
}
```

`type` values map to `ErrorType` constants (`VALIDATION_ERROR`, `NOT_FOUND_ERROR`, `UNAUTHORIZED_ERROR`, `FORBIDDEN_ERROR`, `CONFLICT_ERROR`, `INTERNAL_SERVER_ERROR`, `BAD_GATEWAY_ERROR`, `GATEWAY_TIMEOUT_ERROR`).

## Headers

All responses include:

- `Content-Type: application/json` (or `application/octet-stream` for `File`)
- `Access-Control-Allow-Origin: <CORS_ORIGIN or '*'>`
- `Access-Control-Allow-Methods: *`
- `Access-Control-Allow-Headers: Content-Type` (plus `Location` for `Created` and `MovedPermanently`)

## Baseline Implementation

If the repository does not already have a `RestResult` class, generate one using this shape:

```ts
import { APIGatewayProxyResult } from 'aws-lambda';

export class RestResult {
    private static getCorsOrigin(): string {
        return process.env.CORS_ORIGIN || '*';
    }

    static Ok(body: object): APIGatewayProxyResult {
        return {
            statusCode: 200,
            headers: {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Headers': 'Content-Type',
                'Access-Control-Allow-Origin': RestResult.getCorsOrigin(),
                'Access-Control-Allow-Methods': '*',
            },
            body: JSON.stringify(body),
        };
    }

    static Created(body: object, resourceRoute: string): APIGatewayProxyResult {
        return {
            statusCode: 201,
            headers: {
                'Content-Type': 'application/json',
                Location: resourceRoute,
                'Access-Control-Allow-Headers': 'Content-Type, Location',
                'Access-Control-Allow-Origin': RestResult.getCorsOrigin(),
                'Access-Control-Allow-Methods': '*',
            },
            body: JSON.stringify(body),
        };
    }

    static NoContent(): APIGatewayProxyResult {
        return {
            statusCode: 204,
            headers: {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Headers': 'Content-Type',
                'Access-Control-Allow-Origin': RestResult.getCorsOrigin(),
                'Access-Control-Allow-Methods': '*',
            },
            body: JSON.stringify({ message: 'No content' }),
        };
    }

    static BadRequest(message: string, details?: Record<string, unknown>): APIGatewayProxyResult {
        return {
            statusCode: 400,
            headers: {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Headers': 'Content-Type',
                'Access-Control-Allow-Origin': RestResult.getCorsOrigin(),
                'Access-Control-Allow-Methods': '*',
            },
            body: JSON.stringify({ type: 'VALIDATION_ERROR', message, status: 400, ...(details ? { details } : {}) }),
        };
    }

    static Unauthorized(message: string): APIGatewayProxyResult {
        return {
            statusCode: 401,
            headers: {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Headers': 'Content-Type',
                'Access-Control-Allow-Origin': RestResult.getCorsOrigin(),
                'Access-Control-Allow-Methods': '*',
            },
            body: JSON.stringify({ type: 'UNAUTHORIZED_ERROR', message, status: 401 }),
        };
    }

    static Forbidden(message: string): APIGatewayProxyResult {
        return {
            statusCode: 403,
            headers: {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Headers': 'Content-Type',
                'Access-Control-Allow-Origin': RestResult.getCorsOrigin(),
                'Access-Control-Allow-Methods': '*',
            },
            body: JSON.stringify({ type: 'FORBIDDEN_ERROR', message, status: 403 }),
        };
    }

    static NotFound(message: string): APIGatewayProxyResult {
        return {
            statusCode: 404,
            headers: {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Headers': 'Content-Type',
                'Access-Control-Allow-Origin': RestResult.getCorsOrigin(),
                'Access-Control-Allow-Methods': '*',
            },
            body: JSON.stringify({ type: 'NOT_FOUND_ERROR', message, status: 404 }),
        };
    }

    static Conflict(message: string): APIGatewayProxyResult {
        return {
            statusCode: 409,
            headers: {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Headers': 'Content-Type',
                'Access-Control-Allow-Origin': RestResult.getCorsOrigin(),
                'Access-Control-Allow-Methods': '*',
            },
            body: JSON.stringify({ type: 'CONFLICT_ERROR', message, status: 409 }),
        };
    }

    static InternalServerError(message: string): APIGatewayProxyResult {
        return {
            statusCode: 500,
            headers: {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Headers': 'Content-Type',
                'Access-Control-Allow-Origin': RestResult.getCorsOrigin(),
                'Access-Control-Allow-Methods': '*',
            },
            body: JSON.stringify({ type: 'INTERNAL_SERVER_ERROR', message, status: 500 }),
        };
    }

    static fromDatabaseError(error: unknown): APIGatewayProxyResult | null {
        const code = typeof error === 'object' && error !== null && 'code' in error
            ? String((error as any).code)
            : undefined;

        if (code === 'UNIQUE_VIOLATION') return RestResult.BadRequest('Request conflicts with an existing record.');
        if (code === 'FOREIGN_KEY_VIOLATION') return RestResult.BadRequest('Request references a record that does not exist.');

        return null;
    }
}
```

Add the remaining methods (`Accepted`, `File`, `MovedPermanently`, `BadGateway`, `GatewayTimeout`) following the same pattern when the feature needs them.
