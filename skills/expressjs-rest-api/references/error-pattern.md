# Error Pattern

Use this reference when the target repository does not already have typed error classes or an error response builder.

## Goal

Create a named error class hierarchy for flow control and an `ErrorResponseBuilder` for consistent HTTP error responses.

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
```

## Error Response Model

```ts
// src/models/error.model.ts
import { StatusCode } from '../utilities/status-code';

export type ErrorResponse = {
  type: ErrorType;
  message: string;
  status?: StatusCode;
  details?: Record<string, any>;
};

export enum ErrorType {
  VALIDATION_ERROR = 'ValidationError',
  NOT_FOUND_ERROR = 'NotFoundError',
  EXISTS_ERROR = 'ExistsError',
  UNAUTHORIZED_ERROR = 'UnauthorizedError',
  FORBIDDEN_ERROR = 'ForbiddenError',
  INTERNAL_SERVER_ERROR = 'InternalServerError',
  PASSWORD_SET_ERROR = 'PasswordError',
}

export class ErrorResponseBuilder {
  private readonly error: ErrorResponse = {
    type: ErrorType.INTERNAL_SERVER_ERROR,
    message: '',
  };

  setType(type: ErrorType): this {
    this.error.type = type;
    return this;
  }

  setStatus(status: StatusCode): this {
    this.error.status = status;
    return this;
  }

  setMessage(message: string): this {
    this.error.message = message;
    return this;
  }

  setDetails(details: Record<string, any>): this {
    this.error.details = details;
    return this;
  }

  build(): ErrorResponse {
    return this.error;
  }
}
```

## Guidance

- Always set `this.name` explicitly in each class — controllers branch on `error.name` (string), not `instanceof`, so the name must survive module boundary crossings
- Add domain-specific error subclasses (e.g. `GroupNotFoundError extends IdentityError`) as needed rather than reusing generic errors
- `ErrorResponseBuilder` uses a fluent builder pattern; chain `setType`, `setStatus`, `setMessage`, and optionally `setDetails` before calling `build()`
- Keep `ErrorType` enum values as human-readable strings — they appear in API responses
