# Status Code Pattern

Use this reference when the target repository does not already have a `StatusCode` enum in `src/utilities/`.

## Goal

Create a typed `StatusCode` enum so HTTP status codes are never written as magic numbers in controllers or middleware.

## Baseline Example

```ts
// src/utilities/status-code.ts

export enum StatusCode {
  // 2xx Success
  OK = 200,
  CREATED = 201,
  ACCEPTED = 202,
  NO_CONTENT = 204,

  // 3xx Redirection
  MOVED_PERMANENTLY = 301,
  NOT_MODIFIED = 304,

  // 4xx Client Errors
  BAD_REQUEST = 400,
  UNAUTHORIZED = 401,
  FORBIDDEN = 403,
  NOT_FOUND = 404,
  METHOD_NOT_ALLOWED = 405,
  CONFLICT = 409,
  GONE = 410,
  UNPROCESSABLE_ENTITY = 422,
  TOO_MANY_REQUESTS = 429,

  // 5xx Server Errors
  INTERNAL_SERVER_ERROR = 500,
  BAD_GATEWAY = 502,
  SERVICE_UNAVAILABLE = 503,
}
```

## Usage

```ts
import { StatusCode } from '../utilities/status-code';

// In a controller
response.status(StatusCode.OK).json(item);
response.status(StatusCode.CREATED).json(item);
response.status(StatusCode.NO_CONTENT).end();
response.status(StatusCode.NOT_FOUND).json(errorResponse);
response.status(StatusCode.CONFLICT).json(errorResponse);
response.status(StatusCode.INTERNAL_SERVER_ERROR).json(errorResponse);

// In middleware
response.status(StatusCode.UNAUTHORIZED).json(errorResponse);
response.status(StatusCode.FORBIDDEN).json(errorResponse);
response.status(StatusCode.BAD_REQUEST).json(errorResponse);
```

## Guidance

- Import `StatusCode` in every controller, middleware, and service that sets HTTP status
- Pair with `ErrorResponseBuilder` from `references/error-pattern.md` for consistent error bodies
- Add additional codes as the project needs them; keep the enum the single source of truth
