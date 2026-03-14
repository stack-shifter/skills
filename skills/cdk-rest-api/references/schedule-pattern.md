# Schedule Pattern

Use this reference when the repository needs a reusable pattern for EventBridge-triggered scheduled Lambda jobs.

## Goal

Keep recurring background jobs consistent with the same Lambda and handler patterns used by the API.

## Common Shape

- one `ScheduleLambda` construct per recurring job
- handler file follows the same `HANDLER` registry pattern as API handlers
- entrypoint type is `ScheduledEvent` from `aws-lambda`, not `APIGatewayProxyEvent`
- scheduled jobs should receive shared runtime environment through the same helper path as API routes when possible

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
    description: 'Periodic cleanup job.',
    hours: 12,
    environmentVariables: {
        ...getDefaultLambdaEnvironment(),
        CLEANUP_BATCH_SIZE: '100',
    },
});
```

## Handler Example

```ts
import { ScheduledEvent } from 'aws-lambda';

export const CLEANUP_HANDLER = {
    RUN: 'cleanupHandler',
} as const;

export const cleanupHandler = async (_event: ScheduledEvent): Promise<void> => {
    // build runtime dependencies and execute one scheduled cycle
};
```

## Guidance

- Always use the `HANDLER` registry for `handlerName`, never a bare string.
- Pass shared runtime settings through `getDefaultLambdaEnvironment()` or the local equivalent helper.
- If the job sends email, attach the send-email policy the same way API routes do.
- Keep scheduled jobs behind the same repository and service boundaries used by API routes.
