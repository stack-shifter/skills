# App Pattern

Use this reference when you need a reusable Express app bootstrap pattern.

## Goal

Create one app bootstrap module that loads environment variables, wires middleware, mounts routes, and starts the server.

## Baseline Example

```ts
// src/app.ts
import express, { Express } from 'express';
import cors from 'cors';
import helmet from 'helmet';
import itemRoutes from './routes/item.route';
import { globalErrorHandler } from './middlewares/global-error.middleware';
import { healthCheck } from './controllers/health.controller';

// Load env file after imports and before app setup.
// In production, env vars are injected directly — skip file loading.
if (process.env.NODE_ENV !== 'production') {
  process.loadEnvFile(`.env.${process.env.NODE_ENV || 'development'}`);
}

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

## Suggested Scripts

For new TypeScript projects, use a simple `tsx watch` development script:

```json
{
  "scripts": {
    "dev": "tsx watch src/app.ts",
    "build": "tsc -p tsconfig.json",
    "start": "node dist/app.js"
  }
}
```

If the repository already uses `nodemon`, `ts-node-dev`, or another runner, keep the local convention unless there is a clear reason to standardize.

## Guidance

- If the repository already has an app bootstrap, extend it instead of creating another one.
- If it does not, generate one bootstrap module instead of spreading setup across multiple files.
- `process.loadEnvFile()` is built into modern Node releases — no `dotenv` package needed
- Load the env file immediately after imports in the bootstrap module, and avoid module-level `process.env` reads in imported files before bootstrap runs
- Set `trust proxy` so `req.ip` and `req.ips` reflect the real client IP behind a load balancer
- Register `globalErrorHandler` last — Express 5 auto-forwards async rejections to it
- Use one consistent API prefix such as `/v1/` when the repository does not already use another prefixing scheme
- In Express 5, `express.json()` and `express.urlencoded()` are built-in; no `body-parser` needed
- For new apps, add graceful shutdown for HTTP server, database pool, and AWS clients if the repo does not already provide it
- Follow the repo's existing development runner first; for new TypeScript apps prefer `tsx watch` over `nodemon`
