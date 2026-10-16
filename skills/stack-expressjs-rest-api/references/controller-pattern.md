# Controller Pattern

Keep controllers readable: consume validated inputs, call repositories or services, and shape HTTP responses. These connected examples use
arrow functions and the shared `dbContext`. Preserve local `try/catch` mappings when the repository already requires them; Express 5 can
forward unexpected async failures automatically.

## Connected CRUD Examples

Route middleware validates the corresponding target before each handler. Use plain `RequestHandler` declarations and name the DTO type
where the validated value is read. These annotations describe middleware output; they do not perform runtime validation. Successful writes
return the existing response contract, including Location and bodyless 204 responses.

```ts
// src/controllers/item.controller.ts
import type { RequestHandler } from 'express';
import { dbContext } from '../dependencies/app.dependencies';
import type {
  ItemPath, ItemQuery, ItemPostDto, ItemPutDto,
} from '../models/validation/item.validation';
import { ErrorResponseBuilder, ErrorType } from '../models/entities/error.model';
import { NotFoundError, ValidationError } from '../utilities/errors';
import { StatusCode } from '../utilities/status-code';

export const getItemsHandler: RequestHandler = async (
  _request, response,
): Promise<void> => {
  const query: ItemQuery = response.locals.validated.query;
  const page = await dbContext.items.findAll(query);
  response.status(StatusCode.OK).json(page);
};

export const getItemByIdHandler: RequestHandler = async (
  _request, response, next,
): Promise<void> => {
  const { itemId }: ItemPath = response.locals.validated.params;
  try {
    const item = await dbContext.items.findById(itemId);
    if (!item) throw new NotFoundError('Item not found');
    response.status(StatusCode.OK).json(item);
  } catch (error: unknown) {
    // Keep local error shaping when established; let the global handler cover other failures.
    if (error instanceof NotFoundError) {
      response.status(StatusCode.NOT_FOUND).json(
        new ErrorResponseBuilder()
          .setType(ErrorType.NOT_FOUND_ERROR)
          .setStatus(StatusCode.NOT_FOUND)
          .setMessage(error.message)
          .build(),
      );
      return;
    }
    next(error);
  }
};

export const createItemHandler: RequestHandler = async (
  _request, response,
): Promise<void> => {
  const dto: ItemPostDto = response.locals.validated.body;
  const item = await dbContext.items.create(dto);
  response.location(`/v1/items/${item.id}`).status(StatusCode.CREATED).json(item);
};

export const updateItemHandler: RequestHandler = async (
  _request, response,
): Promise<void> => {
  const { itemId }: ItemPath = response.locals.validated.params;
  const dto: ItemPutDto = response.locals.validated.body;
  if (itemId !== dto.id) {
    throw new ValidationError('Id does not match. Unable to update item.');
  }
  await dbContext.items.update(itemId, dto);
  response.status(StatusCode.NO_CONTENT).end();
};

export const deleteItemHandler: RequestHandler = async (
  _request, response,
): Promise<void> => {
  const { itemId }: ItemPath = response.locals.validated.params;
  await dbContext.items.delete(itemId);
  response.status(StatusCode.NO_CONTENT).end();
};
```

Prefer plain `RequestHandler` for separately declared handlers and explicit DTO types on local variables. Avoid long positional generic
arguments or extra locals types unless required. Inline route callbacks can use inferred Express types. See the official
[TypeScript installation guidance](https://expressjs.com/en/5x/starter/installing/). Follow existing response contracts rather than replacing
them with this example. Return immediately after a response when more code could execute; avoid double sends and `next()` after success.

Put record-ownership authorization before reads/writes when required, using trusted identity and repository queries that enforce the actual
access boundary. A router's group authorization does not replace resource-level authorization.

## Reading Shared Request Context

When a controller needs middleware-added identity, annotate its request with the shared type from
[middleware-pattern.md](middleware-pattern.md). Keep the handler declaration simple:

```ts
// src/controllers/current-user.controller.ts
import type { RequestHandler } from 'express';
import type { AuthenticatedRequest } from '../models/authenticated-request';
import { AuthenticationError } from '../utilities/errors';
import { StatusCode } from '../utilities/status-code';

export const getCurrentUserHandler: RequestHandler = async (
  request: AuthenticatedRequest, response,
): Promise<void> => {
  const user = request.user;
  if (!user) {
    throw new AuthenticationError('Authentication required');
  }

  response.status(StatusCode.OK).json({ id: user.sub });
};
```

Mount this handler after `authorize()` in the existing router. The shared type does not authenticate callers, so check absent identity.
Keep parsed inputs in `response.locals.validated`; extend the request type only for shared context the flow needs.
