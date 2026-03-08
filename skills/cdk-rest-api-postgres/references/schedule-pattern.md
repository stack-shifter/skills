# Schedule Pattern

Use this reference when you need a reusable pattern for EventBridge-triggered background Lambda jobs.

## Goal

Wire scheduled background work (cleanup jobs, digest emails, data expiry) through a shared scheduled Lambda construct rather than raw `events.Rule` + `NodejsFunction` pairs inline in the stack.

## Common Shape

- one `ScheduleLambda` construct per recurring job
- handler file follows the same `HANDLER` registry pattern as API handlers
- entrypoint type is `ScheduledEvent` from `aws-lambda`, not `APIGatewayProxyEvent`
- Postgres-backed jobs get database access via `DATABASE_URL` in `getDefaultLambdaEnvironment()` — no table-level IAM grants needed

## Baseline Example

```ts
import { ScheduleLambda } from './constructs/schedule';
import { CLEANUP_HANDLER } from '../src/handlers/cleanup.handler';
import { getDefaultLambdaEnvironment, getStackLambdaName } from './constructs/node-lambda';

const lambdaName = (name: string) => getStackLambdaName(name, props.deploymentStage);
const handlerPath = (rel: string) => path.join(__dirname, '..', rel);

new ScheduleLambda(this, 'ScheduledCleanup', {
    lambdaName: lambdaName('Cleanup'),
    filePath: handlerPath('src/handlers/cleanup.handler.ts'),
    handlerName: CLEANUP_HANDLER.RUN,
    description: 'Hard-delete due soft-deleted records.',
    hours: 12,
    environmentVariables: {
        ...getDefaultLambdaEnvironment(),
        CLEANUP_BATCH_SIZE: '100',
        SOFT_DELETE_RETENTION_HOURS: '168',
    },
});
```

## Handler Template

```ts
import { ScheduledEvent } from 'aws-lambda';

export const CLEANUP_HANDLER = {
    RUN: 'cleanupHandler',
} as const;

export const cleanupHandler = async (_event: ScheduledEvent): Promise<void> => {
    // build runtime dependencies and execute one cleanup cycle
};
```

## Key Points

- `hours` sets the EventBridge rate schedule
- `enabled` defaults to `true`; set `enabled: false` to deploy the rule disabled
- always use the `HANDLER` registry for `handlerName`, never a bare string
- pass `DATABASE_URL` through `getDefaultLambdaEnvironment()` for Postgres-backed jobs
- if the job sends email, attach the result of `createSendEmailPolicy(this)` via the construct's `additionalPolicies` option or directly to the Lambda function after construction

## Guidance

- If the repository already has a `ScheduleLambda` or equivalent construct, use it.
- If it does not, generate one small construct wrapping `NodejsFunction` and `events.Rule` rather than inlining both wherever needed.
- Keep scheduled job logic in a dedicated job class (e.g., `src/jobs/`) and keep the handler thin.
- Always export a typed `HANDLER` registry constant from the handler file.
