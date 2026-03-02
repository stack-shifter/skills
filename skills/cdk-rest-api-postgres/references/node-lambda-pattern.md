# Node Lambda Pattern

Use this reference when you need a quick summary of this repository's Lambda wrapper and runtime defaults.

Read `lib/constructs/node-lambda.ts` first when making actual code changes.

## Goal

Keep Lambda configuration consistent with the project's existing `NodeLambda` construct.

## Defaults

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

## Required Shared Environment

`getDefaultLambdaEnvironment()` currently requires:

- `DATABASE_URL`
- `FRONTEND_URL`
- `S3_BUCKET`
- `SES_IDENTITY_EMAIL`
- `SES_IDENTITY_EMAIL_ARN`

It also sets:

- `CORS_ORIGIN`
- `ADMIN_GROUP`

## Guidance

- Prefer `NodeLambda` over direct `NodejsFunction` creation in this project.
- Keep new environment variables explicit and additive; do not bypass `getDefaultLambdaEnvironment()`.
- Reuse `getStackLambdaName(...)` for stack-consistent function names.
- If a route needs more memory or another timeout, override only that property instead of redefining the whole Lambda shape.
