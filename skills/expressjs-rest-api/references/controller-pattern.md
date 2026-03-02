# Controller Pattern

Use this reference when the target repository does not already have controller files in `src/controllers/`.

## Goal

Create `RequestHandler` async functions that read request data, call services or repositories, and return consistent HTTP responses.

## Baseline Example

```ts
// src/controllers/item.controller.ts
import { NextFunction, Request, RequestHandler, Response } from 'express';
import { StatusCode } from '../utilities/status-code';
import { itemRepository } from '../dependencies/aws.deps';
import { ItemPostDto } from '../models/item.model';
import { ErrorResponseBuilder, ErrorType } from '../models/error.model';

export const getItemByIdHandler: RequestHandler = async (
  request: Request,
  response: Response,
  next: NextFunction
): Promise<void> => {
  const id = request.params['id'];

  try {
    const item = await itemRepository.findById(id);

    if (!item) {
      response.status(StatusCode.NOT_FOUND).json(
        new ErrorResponseBuilder()
          .setType(ErrorType.NOT_FOUND_ERROR)
          .setStatus(StatusCode.NOT_FOUND)
          .setMessage('Item not found')
          .build()
      );
      return;
    }

    response.status(StatusCode.OK).json(item);
  } catch (error: any) {
    if (error.name === 'DatabaseError') {
      response.status(StatusCode.INTERNAL_SERVER_ERROR).json(
        new ErrorResponseBuilder()
          .setType(ErrorType.INTERNAL_SERVER_ERROR)
          .setStatus(StatusCode.INTERNAL_SERVER_ERROR)
          .setMessage('An internal server error occurred while retrieving the item')
          .build()
      );
      return;
    }
    next(error);
  }
};

export const createItemHandler: RequestHandler = async (
  request: Request,
  response: Response,
  next: NextFunction
): Promise<void> => {
  const dto = request.body as ItemPostDto;

  try {
    const item = await itemRepository.create(dto);
    response
      .setHeader('Location', `/v1/items/${item.id}`)
      .status(StatusCode.CREATED)
      .json(item);
  } catch (error: any) {
    if (error.name === 'ExistsError') {
      response.status(StatusCode.CONFLICT).json(
        new ErrorResponseBuilder()
          .setType(ErrorType.EXISTS_ERROR)
          .setStatus(StatusCode.CONFLICT)
          .setMessage('Item already exists')
          .build()
      );
      return;
    }
    next(error);
  }
};

export const updateItemHandler: RequestHandler = async (
  request: Request,
  response: Response,
  next: NextFunction
): Promise<void> => {
  const id = request.params['id'];
  const dto = request.body;

  try {
    await itemRepository.update(id, dto);
    response.status(StatusCode.NO_CONTENT).end();
  } catch (error: any) {
    if (error.name === 'NotFoundError') {
      response.status(StatusCode.NOT_FOUND).json(
        new ErrorResponseBuilder()
          .setType(ErrorType.NOT_FOUND_ERROR)
          .setStatus(StatusCode.NOT_FOUND)
          .setMessage('Item not found')
          .build()
      );
      return;
    }
    next(error);
  }
};

export const deleteItemHandler: RequestHandler = async (
  request: Request,
  response: Response,
  next: NextFunction
): Promise<void> => {
  const id = request.params['id'];

  try {
    await itemRepository.delete(id);
    response.status(StatusCode.NO_CONTENT).end();
  } catch (error: any) {
    next(error);
  }
};
```

## Guidance

- Always type as `RequestHandler` and return `Promise<void>`
- Branch on `error.name` (string) from the service or repository layer for per-case HTTP mapping
- Call `next(error)` as the final fallback — Express 5 forwards it to `globalErrorHandler`
- `return` after every `response.status(...).json(...)` to prevent double-send
- Set `Location` header on 201 Created responses
- Use `response.status(StatusCode.NO_CONTENT).end()` for 204 responses — no body
- Keep controllers thin: read params/body, call a service or repository, return a response
