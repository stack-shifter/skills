# Node Lambda Pattern

Use this reference when you need a reusable Node.js Lambda wrapper or shared runtime defaults.

## Goal

Provide a consistent baseline for Node.js Lambda configuration.

## Defaults

- runtime: `NODEJS_24_X`
- architecture: `ARM_64`
- memory: `256`
- timeout: `15` seconds
- tracing: active
- bundle with `aws-lambda-nodejs`
- keep environment variables explicit

## Baseline Example

```ts
import * as path from 'node:path';
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as lambdaNodejs from 'aws-cdk-lib/aws-lambda-nodejs';
import * as logs from 'aws-cdk-lib/aws-logs';

const logGroup = new logs.LogGroup(this, 'CreateUserLogGroup', {
    logGroupName: `/aws/lambda/CreateUser${props.deploymentStage}`,
    removalPolicy: cdk.RemovalPolicy.DESTROY,
    retention: logs.RetentionDays.THREE_MONTHS,
});

const createUserFn = new lambdaNodejs.NodejsFunction(this, 'CreateUserFn', {
    functionName: `CreateUser${props.deploymentStage}`,
    description: 'Create a user record',
    entry: path.join(__dirname, '../../src/handlers/users/create-user.handler.ts'),
    handler: 'CREATE_USER_HANDLER',
    runtime: lambda.Runtime.NODEJS_24_X,
    architecture: lambda.Architecture.ARM_64,
    memorySize: 256,
    timeout: cdk.Duration.seconds(15),
    tracing: lambda.Tracing.ACTIVE,
    logGroup,
    bundling: {
        minify: true,
        externalModules: ['@aws-sdk/*'],
    },
    environment: {
        USERS_TABLE_NAME: usersTable.tableName,
    },
    vpc,
    vpcSubnets: vpc ? { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS } : undefined,
});
```

## Guidance

- If the repository already has a wrapper such as `NodeLambda`, extend it instead of bypassing it.
- If it does not, use `NodejsFunction` for TypeScript entrypoints and introduce one shared helper once the pattern repeats.
- Prefer one function per route unless the user explicitly wants a router Lambda
- Keep environment variable names stable and tied to actual infrastructure resources
- If the repo already has logging or tracing conventions, follow those instead
