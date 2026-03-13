# Error Pattern

Use this reference when you need a reusable typed-error and HTTP error-response pattern.

## Goal

Create a named error class hierarchy for flow control and an `ErrorResponseBuilder` or equivalent shared pattern for consistent HTTP error responses.

## Error Classes

```ts
// src/utilities/errors.ts

export class IdentityError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'IdentityError';
  }
}

export class UserNotFoundError extends IdentityError {
  constructor(message: string) {
    super(message);
    this.name = 'UserNotFoundError';
  }
}

export class UserExistError extends IdentityError {
  constructor(message: string) {
    super(message);
    this.name = 'UserExistError';
  }
}

export class UserPasswordError extends IdentityError {
  constructor(message: string) {
    super(message);
    this.name = 'UserPasswordError';
  }
}

export class DatabaseError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'DatabaseError';
  }
}

export class NotFoundError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'NotFoundError';
  }
}

export class ValidationError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'ValidationError';
  }
}

export class AuthenticationError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'AuthenticationError';
  }
}

export class AuthorizationError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'AuthorizationError';
  }
}

export class ConflictError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'ConflictError';
  }
}
```

## Error Response Model

```ts
// src/models/entities/error.model.ts
import { StatusCode } from '../utilities/status-code';

export type ErrorResponse = {
  type: ErrorType;
  message: string;
  status?: StatusCode;
  details?: Record<string, unknown>;
};

export enum ErrorType {
  VALIDATION_ERROR = 'ValidationError',
  NOT_FOUND_ERROR = 'NotFoundError',
  EXISTS_ERROR = 'ExistsError',
  UNAUTHORIZED_ERROR = 'UnauthorizedError',
  FORBIDDEN_ERROR = 'ForbiddenError',
  INTERNAL_SERVER_ERROR = 'InternalServerError',
  PASSWORD_SET_ERROR = 'PasswordError',
  CONFLICT_ERROR = 'ConflictError',
  BAD_GATEWAY_ERROR = 'BadGatewayError',
  GATEWAY_TIMEOUT_ERROR = 'GatewayTimeoutError',
}

export class ErrorResponseBuilder {
  private readonly errorResponse: ErrorResponse;

  constructor() {
    this.errorResponse = {
      type: ErrorType.INTERNAL_SERVER_ERROR,
      status: StatusCode.INTERNAL_SERVER_ERROR,
      message: '',
    };
  }

  setType(type: ErrorType): this {
    this.errorResponse.type = type;
    return this;
  }

  setStatus(status: StatusCode): this {
    this.errorResponse.status = status;
    return this;
  }

  setMessage(message: string): this {
    this.errorResponse.message = message;
    return this;
  }

  setDetails(details: Record<string, unknown>): this {
    this.errorResponse.details = details;
    return this;
  }

  build(): ErrorResponse {
    return this.errorResponse;
  }
}
```

## Global Error Mapping

```ts
// Global error middleware may stay narrow when controllers already map
// common repository and validation errors themselves.
```

## Guidance

- Always set `this.name` explicitly in each class — controllers branch on `error.name` (string), not `instanceof`, so the name must survive module boundary crossings
- If the repository already has typed errors or an error mapper, extend it instead of introducing a second hierarchy.
- If it does not, centralize HTTP mapping in one place instead of repeating the same response shapes across controllers.
- If controllers already use `ErrorResponseBuilder` in local `try/catch` blocks for known errors, preserve that approach and let the global handler remain the fallback for unexpected failures
- Add domain-specific error subclasses (e.g. `GroupNotFoundError extends IdentityError`) as needed rather than reusing generic errors
- `ErrorResponseBuilder` uses a fluent builder pattern; chain `setType`, `setStatus`, `setMessage`, and optionally `setDetails` before calling `build()`
- Keep `ErrorType` enum values as human-readable strings — they appear in API responses
