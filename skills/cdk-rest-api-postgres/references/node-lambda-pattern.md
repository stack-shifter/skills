# Node Lambda Pattern

Use this reference when you need guidance for a reusable Node.js Lambda wrapper or shared runtime defaults.

## Goal

Keep Lambda configuration consistent across routes, whether the repository already has a wrapper or needs one generated.

## Useful Defaults

- runtime: `NODEJS_24_X`
- architecture: `ARM_64`
- memory: `256`
- timeout: `15` seconds
- tracing: active
- log group retention: three months
- bundling: minified, `@aws-sdk/*` externalized

## Baseline Example

```ts
import { NodeLambda } from "./constructs/node-lambda";

const handler = new NodeLambda(this, "ClientsQueryHandler", {
    lambdaName: getStackLambdaName("ClientsQuery", props.deploymentStage),
    filePath: handlerPath("src/handlers/client.handler.ts"),
    handlerName: "queryClientHandler",
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
- `ADMIN_GROUP`

## Guidance

- If the repository already has a wrapper such as `NodeLambda`, extend it instead of bypassing it.
- If no wrapper exists, generate one small shared helper rather than redefining the same `NodejsFunction` options on every route.
- Keep new environment variables explicit and additive through one shared helper or config path.
- Reuse a shared naming helper when the stack already has one.
- If a route needs more memory or another timeout, override only that property instead of redefining the whole Lambda shape.
