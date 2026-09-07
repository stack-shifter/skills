# Node Lambda Pattern

The Storm examples apply when the library is installed. For existing project constructs without Storm, use `general-cdk.md`; keep the same application layers.

Use this reference when you need guidance for a reusable Node.js Lambda wrapper or shared runtime defaults.

## Goal

Keep Lambda configuration consistent across routes, using Storm’s `LambdaNode` and shared application configuration.

## Storm Example

Use Storm’s `LambdaNode` for new infrastructure. This direct stack excerpt uses the project-defined path, naming, and environment helpers described below; `handlerPath` resolves the entry relative to the stack as shown in `rest-api-pattern.md`.

```ts
import { Duration } from 'aws-cdk-lib';
import { LambdaNode } from '@stack-shifter/storm-stack';
import { CLIENT_HANDLER } from '../src/handlers/client.handler';
import { getDefaultLambdaEnvironment, getStackLambdaName } from './constructs/node-lambda';

const handler = new LambdaNode(this, 'ClientsQueryHandler', {
    functionName: getStackLambdaName('ClientsQuery', props.deploymentStage),
    entry: handlerPath('src/handlers/client.handler.ts'),
    handler: CLIENT_HANDLER.QUERY,
    environment: getDefaultLambdaEnvironment(),
    memorySize: 512,
    timeout: Duration.seconds(20),
}).function;
```

The reviewed Storm wrapper defaults to 256 MB, Node.js 24, ARM64, a 15-second timeout, active tracing, and minification. Verify the installed version; inspect its actual log retention and SDK bundling settings.

Validate required environment values in the shared helper; non-null assertions do not validate configuration. Passing a resource name does not grant access to it. Verify configured handler names against actual exports and keep CDK imports free of runtime initialization.

## Shared Environment Pattern

A shared environment helper often needs values such as:

- `FRONTEND_URL`
- `S3_BUCKET`
- `SES_IDENTITY_EMAIL`
- `SES_IDENTITY_EMAIL_ARN`
- `CORS_ORIGIN`
- `ALLOWED_GROUP`
- datastore connection settings or repository configuration needed by the target repository

## Stack Helpers

If the repository exports helper functions from `node-lambda.ts`, use them in the stack instead of building strings inline:

- `getDefaultLambdaEnvironment()` — validates all required env vars and returns a shared environment map; use as `environment` in constructor `defaultRouteOptions` or per-route options
- `getStackLambdaName(name, deploymentStage)` — produces a consistent function name; use to build `functionName` for every route
- `createSendEmailPolicy(scope?)` — produces a minimal IAM policy for SES `SendEmail`; attach only to routes that send email

```ts
import { createSendEmailPolicy, getDefaultLambdaEnvironment, getStackLambdaName } from './constructs/node-lambda';

const sharedEnvironment = getDefaultLambdaEnvironment();
const lambdaName = (name: string) => getStackLambdaName(name, props.deploymentStage);

// For a route that sends email, attach the policy to its generated Lambda.
handler.addToRolePolicy(createSendEmailPolicy(this));
```

Read the local `node-lambda.ts` to confirm the actual exported names and signatures before using these.

## Guidance

- When Storm is selected, use `LambdaNode` directly; otherwise reuse the project wrapper or a local `LambdaNode` wrapping `NodejsFunction`.
- Keep new environment variables explicit and additive through one shared helper or config path.
- Reuse a shared naming helper when the stack already has one.
- If a route needs more memory or another timeout, override only that property instead of redefining the whole Lambda shape.
