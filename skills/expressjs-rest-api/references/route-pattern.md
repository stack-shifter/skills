# Route Pattern

Use this reference when the target repository does not already have route files in `src/routes/`.

## Goal

Create a `Router` instance that wires auth, validation middleware, and controller handlers per route.

## Baseline Example

```ts
// src/routes/item.route.ts
import { Router } from 'express';
import { authorize } from '../middlewares/authorize.middleware';
import { validateBody, validateParams, validateQuery } from '../middlewares/validation.middleware';
import {
  itemIdSchema,
  itemPostDtoSchema,
  itemPutDtoSchema,
  itemQuerySchema,
} from '../models/item.model';
import {
  createItemHandler,
  deleteItemHandler,
  getItemByIdHandler,
  getItemsHandler,
  updateItemHandler,
} from '../controllers/item.controller';

const router: Router = Router({ mergeParams: true });

// Apply auth to all routes in this router
router.use(authorize());

router.get('/', validateQuery(itemQuerySchema), getItemsHandler);
router.get('/:id', validateParams(itemIdSchema), getItemByIdHandler);
router.post('/', validateBody(itemPostDtoSchema), createItemHandler);
router.put('/:id', validateParams(itemIdSchema), validateBody(itemPutDtoSchema), updateItemHandler);
router.delete('/:id', validateParams(itemIdSchema), deleteItemHandler);

export default router;
```

## Guidance

- `Router({ mergeParams: true })` preserves `req.params` from parent routers in nested route trees
- `router.use(authorize())` applies auth to all routes; pass `authorize(['admin'])` to restrict by Cognito group
- Chain validation middleware directly on the route before the handler: `validateParams` then `validateBody`
- For public routes (e.g. health check), register them directly on `app` in `app.ts` without `authorize()`
- Mount this router in `app.ts` with `app.use('/v1/items', itemRoutes)`
