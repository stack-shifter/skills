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

export class ConflictError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'ConflictError';
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

## Global Error Mapping

```ts
// src/middlewares/global-error.middleware.ts
import { ErrorRequestHandler } from 'express';
import { ErrorResponseBuilder, ErrorType } from '../models/error.model';
import { StatusCode } from '../utilities/status-code';

export const globalErrorHandler: ErrorRequestHandler = (error, request, response, next) => {
  if (response.headersSent) {
    next(error);
    return;
  }

  switch (error.name) {
    case 'ValidationError':
      response.status(StatusCode.BAD_REQUEST).json(
        new ErrorResponseBuilder()
          .setType(ErrorType.VALIDATION_ERROR)
          .setStatus(StatusCode.BAD_REQUEST)
          .setMessage(error.message)
          .build()
      );
      return;
    case 'AuthenticationError':
      response.status(StatusCode.UNAUTHORIZED).json(
        new ErrorResponseBuilder()
          .setType(ErrorType.UNAUTHORIZED_ERROR)
          .setStatus(StatusCode.UNAUTHORIZED)
          .setMessage(error.message)
          .build()
      );
      return;
    case 'AuthorizationError':
      response.status(StatusCode.FORBIDDEN).json(
        new ErrorResponseBuilder()
          .setType(ErrorType.FORBIDDEN_ERROR)
          .setStatus(StatusCode.FORBIDDEN)
          .setMessage(error.message)
          .build()
      );
      return;
    case 'NotFoundError':
      response.status(StatusCode.NOT_FOUND).json(
        new ErrorResponseBuilder()
          .setType(ErrorType.NOT_FOUND_ERROR)
          .setStatus(StatusCode.NOT_FOUND)
          .setMessage(error.message)
          .build()
      );
      return;
    case 'ConflictError':
      response.status(StatusCode.CONFLICT).json(
        new ErrorResponseBuilder()
          .setType(ErrorType.EXISTS_ERROR)
          .setStatus(StatusCode.CONFLICT)
          .setMessage(error.message)
          .build()
      );
      return;
    default:
      response.status(StatusCode.INTERNAL_SERVER_ERROR).json(
        new ErrorResponseBuilder()
          .setType(ErrorType.INTERNAL_SERVER_ERROR)
          .setStatus(StatusCode.INTERNAL_SERVER_ERROR)
          .setMessage('An unexpected error occurred. Please check the server logs for more details and try again.')
          .setDetails({ method: request.method, route: request.originalUrl })
          .build()
      );
  }
};
```

## Guidance

- Always set `this.name` explicitly in each class — controllers branch on `error.name` (string), not `instanceof`, so the name must survive module boundary crossings
- Add domain-specific error subclasses (e.g. `GroupNotFoundError extends IdentityError`) as needed rather than reusing generic errors
- `ErrorResponseBuilder` uses a fluent builder pattern; chain `setType`, `setStatus`, `setMessage`, and optionally `setDetails` before calling `build()`
- Keep `ErrorType` enum values as human-readable strings — they appear in API responses
- In new fallback code, centralize HTTP mapping in `globalErrorHandler` instead of repeating the same response builders in every controller
