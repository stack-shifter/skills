# Route Pattern

Compose authorization, validation, and controller handlers in the existing router. Keep public routes explicit.

## Baseline Example

```ts
// src/routes/item.route.ts
import { Router } from 'express';
import { authorize } from '../middlewares/authorize.middleware';
import { validateBody, validateParams, validateQuery } from '../middlewares/validation.middleware';
import {
  itemPathSchema,
  itemPostDtoSchema,
  itemPutDtoSchema,
  itemQuerySchema,
} from '../models/validation/item.validation';
import {
  createItemHandler,
  deleteItemHandler,
  getItemByIdHandler,
  getItemsHandler,
  updateItemHandler,
} from '../controllers/item.controller';

// Fictional group: use the project's approved groups/scopes.
const allowedGroups = ['editors'];
const router: Router = Router();

router.get('/items', authorize(allowedGroups), validateQuery(itemQuerySchema), getItemsHandler);
router.get('/items/:itemId', authorize(allowedGroups), validateParams(itemPathSchema), getItemByIdHandler);
router.post('/items', authorize(allowedGroups), validateBody(itemPostDtoSchema), createItemHandler);
router.put(
  '/items/:itemId',
  authorize(allowedGroups),
  validateParams(itemPathSchema),
  validateBody(itemPutDtoSchema),
  updateItemHandler,
);
router.delete('/items/:itemId', authorize(allowedGroups), validateParams(itemPathSchema), deleteItemHandler);

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

The app example mounts this router at `/v1`, so the resulting paths are `/v1/items` and `/v1/items/:itemId`. Do not mount it at
`/v1/items` again. Preserve another established prefix convention consistently across app, router, Location headers, and tests.
