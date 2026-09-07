# Controller Pattern

Use this reference when you need a reusable controller or route-handler pattern.

## Goal

Create `RequestHandler` async functions or equivalent handlers that read request data, call services or repositories, and return consistent HTTP responses.

When the repository already uses explicit `try/catch` blocks with `ErrorResponseBuilder`, preserve that pattern instead of forcing all named errors through the global error middleware.

## Baseline Example

```ts
// src/controllers/item.controller.ts
import { NextFunction, Request, RequestHandler, Response } from "express";
import { StatusCode } from "../utilities/status-code";
import { dbContext } from "../dependencies/app.dependencies";
import { ItemPostDto, ItemPutDto } from "../models/validation/item.validation";
import { NotFoundError } from "../data/repositories/errors/not-found.error";
import {
  ErrorResponseBuilder,
  ErrorType,
} from "../models/entities/error.model";

export const getItemByIdHandler: RequestHandler = async (
  request: Request,
  response: Response,
  next: NextFunction,
): Promise<void> => {
  // Read the item identifier from the route parameters.
  const id = request.params["itemId"] as string;

  try {
    // Load the item before building the response payload.
    const item = await dbContext.items.findById(id);

    // Return the not-found response when the item does not exist.
    if (!item) {
      response
        .status(StatusCode.NOT_FOUND)
        .json(
          new ErrorResponseBuilder()
            .setType(ErrorType.NOT_FOUND_ERROR)
            .setStatus(StatusCode.NOT_FOUND)
            .setMessage("Item not found")
            .build(),
        );
      return;
    }

    // Return the response payload when the item exists.
    response.status(StatusCode.OK).json(item);
  } catch (error: unknown) {
    // Translate known repository errors into the existing HTTP response shape.
    if (error instanceof NotFoundError) {
      response
        .status(StatusCode.NOT_FOUND)
        .json(
          new ErrorResponseBuilder()
            .setType(ErrorType.NOT_FOUND_ERROR)
            .setStatus(StatusCode.NOT_FOUND)
            .setMessage(error.message)
            .build(),
        );
      return;
    }

    // Delegate unexpected errors to the global error handler.
    next(error);
  }
};

export const createItemHandler: RequestHandler = async (
  request: Request,
  response: Response,
  next: NextFunction,
): Promise<void> => {
  // Read the request body used to create the item.
  const dto = request.body as ItemPostDto;

  try {
    // Create the item from the request payload.
    const item = await dbContext.items.create(dto);

    // Return the created resource and location header.
    response
      .location(`/v1/items/${item.id}`)
      .status(StatusCode.CREATED)
      .json(item);
  } catch (error: unknown) {
    // Delegate unexpected errors to the global error handler.
    next(error);
  }
};

export const updateItemHandler: RequestHandler = async (
  request: Request,
  response: Response,
  next: NextFunction,
): Promise<void> => {
  // Read the route id and request body used for the update.
  const id = request.params["itemId"] as string;
  const dto = request.body as ItemPutDto;

  // Reject updates when the path id and body id do not match.
  if (id !== dto.id) {
    response
      .status(StatusCode.BAD_REQUEST)
      .json(
        new ErrorResponseBuilder()
          .setType(ErrorType.VALIDATION_ERROR)
          .setStatus(StatusCode.BAD_REQUEST)
          .setMessage("Id does not match. Unable to update item.")
          .build(),
      );
    return;
  }

  try {
    // Persist the updated item values.
    await dbContext.items.update(dto);

    // Return the no-content response after a successful update.
    response.status(StatusCode.NO_CONTENT).end();
  } catch (error: unknown) {
    // Translate known repository errors into the existing HTTP response shape.
    if (error instanceof NotFoundError) {
      response
        .status(StatusCode.NOT_FOUND)
        .json(
          new ErrorResponseBuilder()
            .setType(ErrorType.NOT_FOUND_ERROR)
            .setStatus(StatusCode.NOT_FOUND)
            .setMessage(error.message)
            .build(),
        );
      return;
    }

    // Delegate unexpected errors to the global error handler.
    next(error);
  }
};

export const deleteItemHandler: RequestHandler = async (
  request: Request,
  response: Response,
  next: NextFunction,
): Promise<void> => {
  // Read the item identifier from the route parameters.
  const id = request.params["itemId"] as string;

  try {
    // Delete the item when it exists.
    await dbContext.items.remove(id);

    // Return the no-content response after a successful delete.
    response.status(StatusCode.NO_CONTENT).end();
  } catch (error: unknown) {
    // Delegate unexpected errors to the global error handler.
    next(error);
  }
};
```

## Guidance

- Always type as `RequestHandler` and return `Promise<void>`
- If the repository already has controller or handler conventions, extend them.
- If it does not, prefer throwing typed errors and letting `globalErrorHandler` map them to HTTP responses
- Only keep controller-local `try/catch` when the repository already uses that pattern or when a handler needs special response shaping
- `return` after every `response.status(...).json(...)` to prevent double-send
- Set `Location` header on 201 Created responses
- Use `response.status(StatusCode.NO_CONTENT).end()` for 204 responses — no body
- Read validated params/query/body from `response.locals.validated` only when the local validation middleware stores parsed output there
- Keep controllers thin: read input, call a service or repository, return a response
