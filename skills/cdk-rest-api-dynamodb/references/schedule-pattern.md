# Schedule Pattern

Use this reference when you need a reusable pattern for EventBridge-triggered background Lambda jobs.

## Goal

Wire scheduled background work (cleanup jobs, digest emails, data expiry) through a shared scheduled Lambda construct rather than raw `events.Rule` + `NodejsFunction` pairs inline in the stack.

## Common Shape

- one `ScheduleLambda` construct per recurring job
- handler file follows the same `HANDLER` registry pattern as API handlers
- entrypoint type is `ScheduledEvent` from `aws-lambda`, not `APIGatewayProxyEvent`
- DynamoDB table access granted via `tableGrants`, not `grantTableAccess` (that is only for API routes)

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
        DYNAMODB_TABLE: dynamoDbTableName,
        CLEANUP_BATCH_SIZE: '100',
        CLEANUP_MAX_ITEMS: '1000',
        SOFT_DELETE_RETENTION_HOURS: '168',
    },
    tableGrants: [{ table: dynamoDbTable, access: 'readWrite' }],
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
- `tableGrants` accepts `{ table, access: 'read' | 'readWrite' }` entries — do not use `grantTableAccess` (that is only for API routes)
- `enabled` defaults to `true`; set `enabled: false` to deploy the rule disabled
- always use the `HANDLER` registry for `handlerName`, never a bare string
- if the job sends email, attach the result of `createSendEmailPolicy()` to the Lambda after construction

## Guidance

- If the repository already has a `ScheduleLambda` or equivalent construct, use it.
- If it does not, generate one small construct wrapping `NodejsFunction` and `events.Rule` rather than inlining both wherever needed.
- Keep scheduled job logic in a dedicated job class (e.g., `src/jobs/`) and keep the handler thin.
- Always export a typed `HANDLER` registry constant from the handler file.
