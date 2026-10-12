# Schedule Pattern

The Storm examples apply when the library is installed. For existing project constructs without Storm, use `general-cdk.md`; keep the same
application layers.

Use this reference when the repository needs a reusable pattern for EventBridge-triggered scheduled Lambda jobs.

## Goal

Keep recurring background jobs consistent with the same Lambda and handler patterns used by the API.

Use Storm `LambdaNode` with CDK EventBridge scheduling. Keep the same application dependencies and handler registry conventions as API
routes.

## Common Shape

- one Storm Lambda and EventBridge rule per recurring job
- worker entrypoint uses the project's export-name convention; CDK reads dependency-free metadata or verified plain strings
- entrypoint type is `ScheduledEvent` from `aws-lambda`, not `APIGatewayProxyEvent`
- scheduled jobs should receive shared runtime environment through the same helper path as API routes when possible

## Storm Lambda with EventBridge

```ts
import * as path from 'node:path';
import { Duration } from 'aws-cdk-lib';
import { Rule, Schedule } from 'aws-cdk-lib/aws-events';
import { LambdaFunction } from 'aws-cdk-lib/aws-events-targets';
import { LambdaNode } from '@stack-shifter/storm-stack';
import { getDefaultLambdaEnvironment, getStackLambdaName } from './constructs/node-lambda';

const lambdaName = (name: string) => getStackLambdaName(name, props.deploymentStage);
const handlerPath = (rel: string) => path.join(__dirname, '..', rel);

const cleanup = new LambdaNode(this, 'ScheduledCleanup', {
	functionName: lambdaName('Cleanup'),
	entry: handlerPath('src/workers/cleanup.worker.ts'),
	handler: 'cleanupWorker',
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
import type { ScheduledEvent } from 'aws-lambda';
import { cleanupService } from '../app';

export const cleanupWorker = async (_event: ScheduledEvent): Promise<void> => {
	await cleanupService.run();
};
```

## Guidance

- Preserve existing registries and verify export names; keep infrastructure imports free of runtime initialization.
- Pass shared runtime settings through `getDefaultLambdaEnvironment()` or the local equivalent helper.
- If the job sends email, attach the send-email policy the same way API routes do.
- Keep scheduled jobs behind the same repository and service boundaries used by API routes.
- `cleanupService` is the project's composed service, not a Storm export. Workers do not compose HTTP authorization/validation middleware.
  Shared startup configuration must still be available to every worker importing that composition root.
- Choose frequency, batch bounds, concurrency, and retries from the actual workload. The twelve-hour schedule is illustrative, not a default
  requirement. Distinguish EventBridge delivery retries, Lambda asynchronous execution retries, and application cleanup retries; a caught
  failure will not trigger invocation retries.
- If explicit names are needed, scope rules and functions by deployment environment to prevent collisions. Do not add names when
  CDK-generated names already meet the requirement.
- For S3 cleanup ordering, partial failures, and durable retry state, read `storage-cleanup-pattern.md`.
