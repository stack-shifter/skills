# Response Pattern

Use this reference when the target repository does not already have a response helper such as `RestResult`.

## Goal

Return consistent API Gateway proxy responses from controllers and handlers.

## Default Shape

- `Content-Type: application/json`
- CORS headers set consistently
- JSON bodies for success and error responses
- stable status codes that match the operation

## Baseline Example

```ts
import { APIGatewayProxyResult } from 'aws-lambda';

const corsOrigin = process.env.CORS_ORIGIN || '*';

function ok(body: object): APIGatewayProxyResult {
    return {
        statusCode: 200,
        headers: {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Headers': 'Content-Type',
            'Access-Control-Allow-Origin': corsOrigin,
            'Access-Control-Allow-Methods': '*',
        },
        body: JSON.stringify(body),
    };
}

function created(body: object, location: string): APIGatewayProxyResult {
    return {
        statusCode: 201,
        headers: {
            'Content-Type': 'application/json',
            Location: location,
            'Access-Control-Allow-Headers': 'Content-Type, Location',
            'Access-Control-Allow-Origin': corsOrigin,
            'Access-Control-Allow-Methods': '*',
        },
        body: JSON.stringify(body),
    };
}

function notFound(message: string): APIGatewayProxyResult {
    return {
        statusCode: 404,
        headers: {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Headers': 'Content-Type',
            'Access-Control-Allow-Origin': corsOrigin,
            'Access-Control-Allow-Methods': '*',
        },
        body: JSON.stringify({ type: 'NOT_FOUND_ERROR', message, status: 404 }),
    };
}
```

## Guidance

- Prefer a shared helper over repeating inline response objects
- Keep success and error body shapes consistent within the repository
- If the repo later adds a helper like `RestResult`, switch to that and stop using this fallback
