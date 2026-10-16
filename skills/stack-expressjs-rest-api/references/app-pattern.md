# App Pattern

Separate application composition from process startup. Preserve an established bootstrap; for new apps export an app that tests can import
without opening a listener or loading developer environment files.

The filenames are examples, not requirements. Keep an existing `app.ts` entrypoint when appropriate. For Lambda Web Functions, adapt its
listen address using [lambda-web-functions.md](lambda-web-functions.md); preserve the same routes and application layers.

## App Composition

The router declares `/items`, so mount it under `/v1` exactly once.

```ts
// src/app.ts
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import itemRoutes from './routes/item.route';
import { globalErrorHandler } from './middlewares/global-error.middleware';
import { NotFoundError } from './utilities/errors';

export const app = express();

// Enable trust proxy only for the actual trusted proxy topology; leave it off by default.
app.use(helmet());
app.use(cors({ origin: process.env.CORS_ORIGIN || '*' }));
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

app.use('/v1', itemRoutes);
app.get('/heartbeat', (_request, response) => {
  response.json({ status: 'ok' });
});

// A missing route is not automatically an Express error.
app.use((_request, _response, next) => {
  next(new NotFoundError('Route not found'));
});
app.use(globalErrorHandler);
```

CORS is a browser policy, not authorization. Preserve the project's allowed origins and credential rules. Configure `trust proxy` for known
hops or addresses only when deployed behind a trusted proxy; unconditional `true` can trust caller-controlled forwarded headers. See
[Express behind proxies](https://expressjs.com/en/guide/behind-proxies/).

## Process Startup

Load local environment files through the development command below, before this module runs. Production receives injected configuration.
Keep the project's module system; the extensionless examples work with `tsx` or compatible CommonJS compilation.

```ts
// src/server.ts
import { app } from './app';

const port = Number(process.env.PORT || 3000);

const server = app.listen(port, (error?: Error) => {
  if (error) {
    console.error('Unable to start HTTP server', error);
    process.exitCode = 1;
    return;
  }
  console.log(`Server running on port ${port}`);
});

const shutdown = (): void => {
  server.close((error) => {
    if (error) {
      console.error('Unable to close HTTP server', error);
      process.exitCode = 1;
    }
    // Close repository pools and other owned resources here using the existing lifecycle hooks.
  });
};

process.once('SIGTERM', shutdown);
process.once('SIGINT', shutdown);
```

Follow existing shutdown timeouts and resource cleanup. Do not add an HTTP listener or HTTP auth middleware to background workers.

## Environment and Scripts

Commit `.env.example` with required keys and fictional values; gitignore local `.env.development` and any `.env.production`. Production
normally receives injected configuration. Do not create unused production files or load development files in tests.

```dotenv
NODE_ENV=development
PORT=3000
CORS_ORIGIN=http://localhost:5173
COGNITO_USER_POOL_ID=us-east-1_EXAMPLE
COGNITO_APP_CLIENT_IDS=example-client-id
```

Match the existing runner. For a new TypeScript app, install `tsx` as a development dependency:

```bash
npm install --save-dev tsx
```

Use watch mode for development, a separate type check, and compiled JavaScript for production. `tsx` does not type-check:

```json
{
  "scripts": {
    "dev": "tsx watch --env-file=.env.development src/server.ts",
    "typecheck": "tsc --noEmit",
    "build": "tsc -p tsconfig.json",
    "start": "node dist/server.js"
  }
}
```

Use the project's existing `tsconfig.json` first. Only when none exists, use the current TypeScript recommendation for Node.js projects:
`NodeNext`, which follows current Node module behavior rather than a fixed older version. The fallback below keeps CommonJS output and
extensionless imports through `"type": "commonjs"` (or no `type`) in `package.json`; it does not require an ESM migration.

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "rootDir": "src",
    "outDir": "dist",
    "strict": true,
    "esModuleInterop": true
  },
  "include": ["src/**/*.ts"]
}
```

Use a current supported TypeScript release and check compatibility with the deployed Node runtime. For an existing ESM project, preserve
its import/build conventions. See TypeScript's [module guidance](https://www.typescriptlang.org/tsconfig/module.html).

Keep test-runner types/configuration in the existing test setup. The development command loads the env file before app imports execute;
production `start` uses injected variables. Retain another established bootstrap and avoid duplicate env loading. Native Node type stripping does not support the enum/parameter-property syntax used elsewhere in these
references and does not replace type checking. See [Express installation](https://expressjs.com/en/5x/starter/installing/) for prerequisites
and TypeScript setup, and the [migration guide](https://expressjs.com/en/guide/migrating-5/) for listener error behavior.
