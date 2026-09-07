# Schedule Pattern

Use this reference when the repository needs a reusable pattern for EventBridge-triggered scheduled Lambda jobs.

## Goal

Keep recurring background jobs consistent with the same Lambda and handler patterns used by the API.

Use Storm `LambdaNode` with CDK EventBridge scheduling. Keep the same application dependencies and handler registry conventions as API routes.

## Common Shape

- one Storm Lambda and EventBridge rule per recurring job
- handler file follows the same `HANDLER` registry pattern as API handlers
- entrypoint type is `ScheduledEvent` from `aws-lambda`, not `APIGatewayProxyEvent`
- scheduled jobs should receive shared runtime environment through the same helper path as API routes when possible

## Storm Lambda with EventBridge

```ts
import * as path from 'node:path';
import { Duration } from 'aws-cdk-lib';
import { Rule, Schedule } from 'aws-cdk-lib/aws-events';
import { LambdaFunction } from 'aws-cdk-lib/aws-events-targets';
import { LambdaNode } from '@stack-shifter/storm-stack';
import { CLEANUP_HANDLER } from '../src/handlers/cleanup.handler';
import { getDefaultLambdaEnvironment, getStackLambdaName } from './constructs/node-lambda';

const lambdaName = (name: string) => getStackLambdaName(name, props.deploymentStage);
const handlerPath = (rel: string) => path.join(__dirname, '..', rel);

const cleanup = new LambdaNode(this, 'ScheduledCleanup', {
    functionName: lambdaName('Cleanup'),
    entry: handlerPath('src/handlers/cleanup.handler.ts'),
    handler: CLEANUP_HANDLER.RUN,
    description: 'Periodic cleanup job.',
    environment: {
        ...getDefaultLambdaEnvironment(),
        CLEANUP_BATCH_SIZE: '100',
    },
}).function;

new Rule(this, 'CleanupSchedule', {
    schedule: Schedule.rate(Duration.hours(12)),
    targets: [new LambdaFunction(cleanup)],
});
```

## Handler Example

```ts
import { ScheduledEvent } from 'aws-lambda';

export const CLEANUP_HANDLER = {
    RUN: 'cleanupHandler',
} as const;

export const cleanupHandler = async (_event: ScheduledEvent): Promise<void> => {
    // execute one scheduled cycle using reusable module-scope dependencies
};
```

## Guidance

- Preserve existing registries and verify export names; keep infrastructure imports free of runtime initialization.
- Pass shared runtime settings through `getDefaultLambdaEnvironment()` or the local equivalent helper.
- If the job sends email, attach the send-email policy the same way API routes do.
- Keep scheduled jobs behind the same repository and service boundaries used by API routes.
