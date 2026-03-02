# App Pattern

Use this reference when the target repository does not already have an Express app entry point.

## Goal

Create a single `src/app.ts` that loads environment variables, wires middleware, mounts routes, and starts the server.

## Baseline Example

```ts
// src/app.ts

// Load env file before anything else reads process.env.
// In production, env vars are injected directly — skip file loading.
if (process.env.NODE_ENV !== 'production') {
  process.loadEnvFile(`.env.${process.env.NODE_ENV || 'development'}`);
}

import express, { Express } from 'express';
import cors from 'cors';
import helmet from 'helmet';
import itemRoutes from './routes/item.route';
import { globalErrorHandler } from './middlewares/global-error.middleware';
import { healthCheck } from './controllers/health.controller';

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

## Environment Files

Create three files at the project root:

```
.env.example        ← committed to git; documents all required variables
.env.development    ← local values; gitignored
.env.production     ← production values; gitignored
```

Add to `.gitignore`:

```
.env.development
.env.production
```

`.env.example` should contain every variable with empty or placeholder values so new contributors know what to fill in:

```dotenv
NODE_ENV=
PORT=3000
CORS_ORIGIN=
AWS_REGION=
COGNITO_USER_POOL_ID=
COGNITO_APP_CLIENT_IDS=
DYNAMODB_TABLE=
# Local development only
AWS_PROFILE=
DYNAMODB_ENDPOINT=
```

## Guidance

- `process.loadEnvFile()` is built into Node.js 22.9+ / Node.js 24 — no `dotenv` package needed
- The `loadEnvFile` call must appear before any `import` that reads `process.env` at module load time; place it at the very top of `app.ts` before other imports, or in a dedicated `src/env.ts` loaded first
- Set `trust proxy` so `req.ip` and `req.ips` reflect the real client IP behind a load balancer
- Register `globalErrorHandler` last — Express 5 auto-forwards async rejections to it
- Use `/v1/` prefix on all resource routes; the health check is public with no auth middleware
- In Express 5, `express.json()` and `express.urlencoded()` are built-in; no `body-parser` needed
