# Node Lambda Pattern

Use this reference when you need guidance for a reusable Node.js Lambda wrapper or shared runtime defaults.

## Goal

Keep Lambda configuration consistent across routes, whether the repository already has a wrapper or needs one generated.

## Useful Defaults

- runtime: `NODEJS_24_X`
- architecture: `ARM_64`
- memory: `128`
- timeout: `15` seconds
- tracing: active
- log group retention: three months
- bundling: minified, `@aws-sdk/*` externalized

## Baseline Example

```ts
import { CLIENT_HANDLER } from "../src/handlers/client.handler";
import { NodeLambda } from "./constructs/node-lambda";

const handler = new NodeLambda(this, "ClientsQueryHandler", {
    lambdaName: getStackLambdaName("ClientsQuery", props.deploymentStage),
    filePath: handlerPath("src/handlers/client.handler.ts"),
    handlerName: CLIENT_HANDLER.QUERY,
    description: "/clients",
    environmentVariables: getDefaultLambdaEnvironment(),
    memory: 512,
}).function;
```

## Shared Environment Pattern

A shared environment helper often needs values such as:

- `DATABASE_URL`
- `FRONTEND_URL`
- `S3_BUCKET`
- `SES_IDENTITY_EMAIL`
- `SES_IDENTITY_EMAIL_ARN`

It may also set:

- `CORS_ORIGIN`
- `ALLOWED_GROUP`

## Stack Helpers

If the repository exports helper functions from `node-lambda.ts`, use them in the stack instead of building strings inline:

- `getDefaultLambdaEnvironment()` — validates all required env vars and returns a shared environment map; use as `environmentVariables` in `setDefaultRouteOptions` or per-route options
- `getStackLambdaName(scope, name)` — derives a unique, stack-scoped Lambda function name; use to build `lambdaName` for every route
- `createSendEmailPolicy(scope)` — produces a minimal IAM policy for SES `SendEmail`; attach only to routes that send email

```ts
import { createSendEmailPolicy, getDefaultLambdaEnvironment, getStackLambdaName } from './constructs/node-lambda';

const sharedEnvironment = getDefaultLambdaEnvironment();
const lambdaName = (name: string) => getStackLambdaName(this, name);

// For routes that send email, attach the SES policy to the returned Lambda function.
const fn = api.post({ ... });
fn.addToRolePolicy(createSendEmailPolicy(this));
```

Read `lib/constructs/node-lambda.ts` to confirm the actual exported names and signatures before using these.

## Guidance

- If the repository already has a wrapper such as `NodeLambda`, extend it instead of bypassing it.
- If no wrapper exists, generate one small shared helper rather than redefining the same `NodejsFunction` options on every route.
- Keep new environment variables explicit and additive through one shared helper or config path.
- Reuse a shared naming helper when the stack already has one.
- If a route needs more memory or another timeout, override only that property instead of redefining the whole Lambda shape.
