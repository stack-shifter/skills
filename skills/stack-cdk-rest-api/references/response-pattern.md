# Response Pattern

Reuse the project's response helper. For new error contracts, prefer RFC 9457 Problem Details; preserve established APIs until an explicit
migration is requested. Read [problem-details.md](problem-details.md) for standard members, optional extensions, and safety boundaries.

## Storm Responses

When installed, inspect `@stack-shifter/storm-stack/framework` exports, types, and tests first. The reviewed interface has lowercase
`ok`, `created`, `noContent`, `badRequest`, `unauthorized`, `forbidden`, `notFound`, `conflict`, and `internalServerError` methods. Project-local
PascalCase helpers and `fromPersistenceError` are separate interfaces; do not assume Storm provides them. Do not import CDK through the
root package or bypass exports with deep imports. Do not install Storm merely for response formatting.

The examples assume the reviewed `RestResult` signature and an application-configured `corsOrigin`. Storm owns the Problem Details body,
serialization, HTTP status, and headers; application code supplies safe detail text and optional extensions.

```ts
import { RestResult } from '@stack-shifter/storm-stack/framework';

const corsOrigin = 'https://app.example.com'; // Supplied by the project's configuration in application code.

const response = RestResult.badRequest('Invalid request body.', {
  origin: corsOrigin,
  extensions: {
    reason: 'InvalidFields',
    fields: { mileage: ['Enter a non-negative whole-mile value.'] },
  },
});
```

Resulting response:

```http
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json
Access-Control-Allow-Origin: https://app.example.com
```

```json
{
  "type": "about:blank",
  "title": "Bad Request",
  "status": 400,
  "detail": "Invalid request body.",
  "reason": "InvalidFields",
  "fields": { "mileage": ["Enter a non-negative whole-mile value."] }
}
```

`extensions` is a helper argument. Its members appear at the top level of the response, not inside an `extensions` property. Preserve
custom type URIs and instance behavior through the installed helper's actual options. Do not override the problem content type through
header options. Pass configured CORS on controller and middleware failures; preserve authentication challenges and other required headers.

Success responses keep the existing contract. For the reviewed Storm interface:

```ts
RestResult.ok({ id: 'example' }, { origin: corsOrigin });
RestResult.created({ id: 'example' }, { origin: corsOrigin, location: '/items/example' });
RestResult.noContent({ origin: corsOrigin });
```

## Expected Domain Errors

These excerpts assume existing `UserNotFoundError`/`UnitConflictError` classes that guarantee safe public messages, a `userService`, and a
project logger that redacts diagnostics. Services/repositories throw domain errors; controllers or the existing boundary mapper own HTTP
translation. Preserve an existing mapper rather than repeating mappings throughout controllers.

```ts
try {
  const updated = await userService.update(userId, input);
  return RestResult.ok(updated, { origin: corsOrigin });
} catch (error: unknown) {
  if (error instanceof UserNotFoundError) {
    return RestResult.notFound(error.message, {
      origin: corsOrigin,
      extensions: { reason: 'UserNotFound' },
    });
  }

  // The project's logger must redact error objects before writing them.
  loggerService.error(
    'Unable to update user.',
    error instanceof Error ? error : new Error('Unknown update failure.'),
  );
  return RestResult.internalServerError('Unable to update user.', { origin: corsOrigin });
}
```

A controller whose operation can produce the existing `UnitConflictError` can map it before its generic fallback:

```ts
if (error instanceof UnitConflictError) {
  return RestResult.conflict(error.message, {
    origin: corsOrigin,
    extensions: { reason: 'UnitConflict', fields: { unit: [error.message] } },
  });
}
```

Do not return arbitrary database/SDK error messages. Keep intentional 404 responses for unauthorized resources. Upload ownership and parent
record permissions are separate checks; validate access before including upload identifiers:

```ts
return RestResult.badRequest('One or more uploads are unavailable.', {
  origin: corsOrigin,
  extensions: { reason: 'UploadInvalid', files: { [uploadId]: ['Upload is unavailable.'] } },
});
```

## Project-Local Fallback Without Storm

If no suitable helper exists, add only the function needed by the new contract. This example assumes Node.js and `@types/aws-lambda`.
It uses an explicit configured origin rather than introducing a wildcard fallback. Preserve the project's complete CORS/credentials policy.
The helper supports the demonstrated `about:blank` contract; reuse existing handling for custom types/instances when present.

```ts
// src/utilities/problem-response.ts
import { STATUS_CODES } from 'node:http';
import type { APIGatewayProxyResult } from 'aws-lambda';

type ProblemExtensions = {
  reason?: string;
  fields?: Record<string, string[]>;
  files?: Record<string, string[]>;
};

export const problemResult = (
  statusCode: number,
  detail: string,
  origin: string,
  extensions: ProblemExtensions = {},
  headers: Record<string, string> = {},
): APIGatewayProxyResult => ({
  statusCode,
  headers: {
    ...headers,
    'Access-Control-Allow-Origin': origin,
    'Content-Type': 'application/problem+json',
  },
  body: JSON.stringify({
    ...extensions,
    type: 'about:blank',
    title: STATUS_CODES[statusCode] ?? 'HTTP Error',
    status: statusCode,
    detail,
  }),
});
```

Use application-selected 4xx/5xx statuses, never arbitrary status values from unknown errors. Header inputs are trusted application
configuration; do not supply conflicting case variants of Content-Type or CORS headers. Extensions cannot override standard body members.
Existing success helpers continue returning their own JSON/bodyless responses.

## Existing Project-Local Helpers

An established `RestResult.Ok`, `Created`, `NoContent`, `BadRequest`, or similar interface may use a custom `type`/`message`/`details`
envelope. Preserve it when extending the API. Changing to Storm's lowercase methods and Problem Details is a contract migration, including
content type, reasons, field keys, CORS, and headers. Do not mix old and new contracts accidentally.

If `fromPersistenceError(error)` already exists, preserve its domain mapping; otherwise map typed domain errors at the existing HTTP
boundary. Repositories should translate driver-specific failures into domain errors without building HTTP responses. Add only the methods
required by the feature; keep response helpers free of persistence imports and error registries.

API Gateway authorizers, throttling, and upstream proxies may emit separate responses. Inspect those layers before claiming uniform
Problem Details coverage, and keep infrastructure changes/deployment outside scope unless authorized.
