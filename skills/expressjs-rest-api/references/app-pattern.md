# App Pattern

Use this reference when the target repository does not already have an Express app entry point.

## Goal

Create a single `src/app.ts` that wires middleware, mounts routes, and starts the server.

## Baseline Example

```ts
// src/app.ts
import express, { Express } from 'express';
import dotenv from 'dotenv';
import cors from 'cors';
import helmet from 'helmet';
import itemRoutes from './routes/item.route';
import { globalErrorHandler } from './middlewares/global-error.middleware';
import { healthCheck } from './controllers/health.controller';

dotenv.config();

const app: Express = express();

app.set('trust proxy', true);
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(cors({ origin: process.env.CORS_ORIGIN || '*' }));
app.use(helmet());

const PORT = process.env.PORT || 3000;

app.use('/v1/items', itemRoutes);
app.get('/v1/health', healthCheck);

// Global error handler must be the last middleware registered
app.use(globalErrorHandler);

app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

## Guidance

- Set `trust proxy` so `req.ip` and `req.ips` reflect the real client IP behind a load balancer
- Register `globalErrorHandler` last — Express 5 auto-forwards async rejections to it
- Use `/v1/` prefix on all resource routes; the health check is public (no auth middleware)
- Keep `dotenv.config()` at the top before any code that reads `process.env`
- In Express 5, `express.json()` and `express.urlencoded()` are built-in; no `body-parser` needed
