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
import { getAllowedGroups } from '../utilities/allowed-groups';

const allowedGroups = getAllowedGroups();
const router: Router = Router();

router.get('/items', authorize(allowedGroups), validateQuery(itemQuerySchema), getItemsHandler);
router.get('/items/:itemId', authorize(allowedGroups), validateParams(itemIdSchema), getItemByIdHandler);
router.post('/items', authorize(allowedGroups), validateBody(itemPostDtoSchema), createItemHandler);
router.put(
  '/items/:itemId',
  authorize(allowedGroups),
  validateParams(itemIdSchema),
  validateBody(itemPutDtoSchema),
  updateItemHandler,
);
router.delete('/items/:itemId', authorize(allowedGroups), validateParams(itemIdSchema), deleteItemHandler);

export default router;
```

## Guidance

- `Router({ mergeParams: true })` preserves `req.params` from parent routers in nested route trees when nested routing is already part of the repository
- If the repository already has router modules, extend them. If it does not, generate router modules rather than registering every resource directly in the app bootstrap.
- If the repository already uses explicit `authorize(getAllowedGroups())` calls on each route, preserve that style
- Chain validation middleware directly on the route before the handler: `validateParams` then `validateBody`
- For public routes (e.g. health check or token-based share-link routes), register them on the app bootstrap or a public router without `authorize()`
- If the app mounts routers under `/v1`, keep route modules aligned with that prefixing strategy
- Mount the router from one centralized bootstrap or route-composition layer
