# Middleware Pattern

Use this reference when you need reusable Express middleware patterns for auth, validation, and global error handling.

## Goal

Create reusable middleware units for Cognito JWT auth, Zod request validation, and global error handling.

## Auth Middleware

```ts
// src/middlewares/authorize.middleware.ts
import { CognitoJwtVerifier } from 'aws-jwt-verify';
import { CognitoAccessTokenPayload } from 'aws-jwt-verify/jwt-model';
import { NextFunction, Request, RequestHandler, Response } from 'express';
import { z } from 'zod/v4';
import { ErrorResponseBuilder, ErrorType } from '../models/entities/error.model';
import { StatusCode } from '../utilities/status-code';

const jwtVerifier = CognitoJwtVerifier.create({
  userPoolId: process.env.COGNITO_USER_POOL_ID!,
  tokenUse: 'access',
  clientId: process.env.COGNITO_APP_CLIENT_IDS
    ? process.env.COGNITO_APP_CLIENT_IDS.split(',').map((id) => id.trim())
    : null,
});

const authHeaderSchema = z
  .string()
  .regex(/^Bearer\s[\w-]+\.[\w-]+\.[\w-]+$/, 'Invalid bearer token format');

export interface AuthenticatedRequest extends Request {
  user?: CognitoAccessTokenPayload;
}

export const authorize = (allowedGroups?: string[]): RequestHandler =>
  async (request: AuthenticatedRequest, response: Response, next: NextFunction): Promise<void> => {
    try {
      const authHeader = request.headers.authorization;
      const result = authHeaderSchema.safeParse(authHeader);

      if (!result.success) {
        response.status(StatusCode.UNAUTHORIZED).json(
          new ErrorResponseBuilder()
            .setType(ErrorType.UNAUTHORIZED_ERROR)
            .setStatus(StatusCode.UNAUTHORIZED)
            .setMessage('Unauthorized: Missing or invalid authorization header')
            .build()
        );
        return;
      }

      const token = authHeader!.split(' ')[1];
      const payload = await jwtVerifier.verify(token);
      const groups = payload['cognito:groups'] as string[] | undefined;

      if (allowedGroups && (!groups || !groups.some((g) => allowedGroups.includes(g)))) {
        response.status(StatusCode.FORBIDDEN).json(
          new ErrorResponseBuilder()
            .setType(ErrorType.FORBIDDEN_ERROR)
            .setStatus(StatusCode.FORBIDDEN)
            .setMessage('Forbidden: You do not have permission to access this resource')
            .build()
        );
        return;
      }

      request.user = payload;
      next();
    } catch {
      response.status(StatusCode.UNAUTHORIZED).json(
        new ErrorResponseBuilder()
          .setType(ErrorType.UNAUTHORIZED_ERROR)
          .setStatus(StatusCode.UNAUTHORIZED)
          .setMessage('Unauthorized: Invalid or expired token')
          .build()
      );
    }
  };
```

## Validation Middleware

```ts
// src/middlewares/validation.middleware.ts
import { NextFunction, Request, Response } from 'express';
import { ZodTypeAny } from 'zod/v4';
import { $ZodIssue } from 'zod/v4/core';
import { ErrorResponseBuilder, ErrorType } from '../models/entities/error.model';
import { StatusCode } from '../utilities/status-code';

export const validateBody = (schema: ZodTypeAny) =>
  (request: Request, response: Response, next: NextFunction): void => {
    const result = schema.safeParse(request.body);
    if (!result.success) {
      response.status(StatusCode.BAD_REQUEST).json(
        new ErrorResponseBuilder()
          .setType(ErrorType.VALIDATION_ERROR)
          .setStatus(StatusCode.BAD_REQUEST)
          .setMessage('Invalid request body')
          .setDetails(formatZodIssues(result.error.issues))
          .build()
      );
      return;
    }
    next();
  };

export const validateQuery = (schema: ZodTypeAny) =>
  (request: Request, response: Response, next: NextFunction): void => {
    const result = schema.safeParse(request.query);
    if (!result.success) {
      response.status(StatusCode.BAD_REQUEST).json(
        new ErrorResponseBuilder()
          .setType(ErrorType.VALIDATION_ERROR)
          .setStatus(StatusCode.BAD_REQUEST)
          .setMessage('Invalid query parameters')
          .setDetails(formatZodIssues(result.error.issues))
          .build()
      );
      return;
    }
    next();
  };

export const validateParams = (schema: ZodTypeAny) =>
  (request: Request, response: Response, next: NextFunction): void => {
    const result = schema.safeParse(request.params);
    if (!result.success) {
      response.status(StatusCode.BAD_REQUEST).json(
        new ErrorResponseBuilder()
          .setType(ErrorType.VALIDATION_ERROR)
          .setStatus(StatusCode.BAD_REQUEST)
          .setMessage('Invalid parameters')
          .setDetails(formatZodIssues(result.error.issues))
          .build()
      );
      return;
    }
    next();
  };

const formatZodIssues = (issues: $ZodIssue[]): Record<string, string> => {
  const details: Record<string, string> = {};
  for (const issue of issues) {
    details[issue.path.join('.')] = issue.message;
  }
  return details;
};
```

## Global Error Middleware

```ts
// src/middlewares/global-error.middleware.ts
import { ErrorRequestHandler, NextFunction, Request, Response } from 'express';
import { ErrorResponseBuilder, ErrorType } from '../models/entities/error.model';
import { StatusCode } from '../utilities/status-code';

export const globalErrorHandler: ErrorRequestHandler = (
  error: any,
  request: Request,
  response: Response,
  next: NextFunction
) => {
  if (response.headersSent) {
    next(error);
    return;
  }

  console.error('Global Error Handler', error);

  response.status(StatusCode.INTERNAL_SERVER_ERROR).json(
    new ErrorResponseBuilder()
      .setType(ErrorType.INTERNAL_SERVER_ERROR)
      .setStatus(StatusCode.INTERNAL_SERVER_ERROR)
      .setMessage('An unexpected error occurred. Please check the server logs for more details and try again.')
      .setDetails({ method: request.method, route: request.originalUrl })
      .build()
  );
};
```

## Guidance

- `authorize()` with no arguments authenticates any valid Cognito token; pass group names to also authorize
- If the repository already has middleware helpers, extend them instead of rewriting the request pipeline.
- If it does not, generate shared middleware units instead of embedding auth or validation directly in controllers.
- `authorize` attaches the decoded payload to `request.user` for downstream access
- Validation middleware returns early on failure — no `next()` on the error path
- If the repository already stores parsed Zod output on `response.locals.validated`, keep using that pattern
- If the repository only validates and continues reading from `request.body`, `request.params`, and `request.query`, preserve that pattern instead of mixing both approaches
- `globalErrorHandler` must declare all four parameters `(err, req, res, next)` or Express ignores it
- In Express 5, unhandled async rejections in route handlers reach `globalErrorHandler` automatically
