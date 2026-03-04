# Route Pattern

Use this reference when you need a reusable router pattern for Express routes.

## Goal

Create a `Router` instance or equivalent route module that wires auth, validation middleware, and controller handlers per route.

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
- If the repository already has router modules, extend them. If it does not, generate router modules rather than registering every resource directly in the app bootstrap.
- `router.use(authorize())` applies auth to all routes; pass `authorize(['admin'])` to restrict by Cognito group
- Chain validation middleware directly on the route before the handler: `validateParams` then `validateBody`
- For public routes (e.g. health check), register them on the app bootstrap or a public router without `authorize()`
- Mount the router from one centralized bootstrap or route-composition layer
