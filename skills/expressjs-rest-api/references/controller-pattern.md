# Controller Pattern

Use this reference when the target repository does not already have controller files in `src/controllers/`.

## Goal

Create `RequestHandler` async functions that read request data, call services or repositories, and return consistent HTTP responses.

## Baseline Example

```ts
// src/controllers/item.controller.ts
import { Request, RequestHandler, Response } from 'express';
import { StatusCode } from '../utilities/status-code';
import { itemRepository } from '../dependencies/aws.deps';
import { ItemPostDto } from '../models/item.model';
import { NotFoundError } from '../utilities/errors';

export const getItemByIdHandler: RequestHandler = async (
  request: Request,
  response: Response
): Promise<void> => {
  const id = response.locals.validated?.params.id ?? request.params['id'];
  const item = await itemRepository.findById(id);

  if (!item) {
    throw new NotFoundError('Item not found');
  }

  response.status(StatusCode.OK).json(item);
};

export const createItemHandler: RequestHandler = async (
  request: Request,
  response: Response
): Promise<void> => {
  const dto = (response.locals.validated?.body ?? request.body) as ItemPostDto;
  const item = await itemRepository.create(dto);

  response
    .setHeader('Location', `/v1/items/${item.id}`)
    .status(StatusCode.CREATED)
    .json(item);
};

export const updateItemHandler: RequestHandler = async (
  request: Request,
  response: Response
): Promise<void> => {
  const id = response.locals.validated?.params.id ?? request.params['id'];
  const dto = response.locals.validated?.body ?? request.body;

  await itemRepository.update(id, dto);
  response.status(StatusCode.NO_CONTENT).end();
};

export const deleteItemHandler: RequestHandler = async (
  request: Request,
  response: Response
): Promise<void> => {
  const id = response.locals.validated?.params.id ?? request.params['id'];

  await itemRepository.delete(id);
  response.status(StatusCode.NO_CONTENT).end();
};
```

## Guidance

- Always type as `RequestHandler` and return `Promise<void>`
- In new fallback code, prefer throwing typed errors and letting `globalErrorHandler` map them to HTTP responses
- Only keep controller-local `try/catch` when the repository already uses that pattern or when a handler needs special response shaping
- `return` after every `response.status(...).json(...)` to prevent double-send
- Set `Location` header on 201 Created responses
- Use `response.status(StatusCode.NO_CONTENT).end()` for 204 responses — no body
- Read validated params/query/body from `response.locals.validated` when validation middleware provides them
- Keep controllers thin: read input, call a service or repository, return a response
