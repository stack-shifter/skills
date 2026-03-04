# REST API Pattern

Use this reference when you need a reusable API Gateway REST route composition pattern.

## Goal

Create a reusable REST API pattern for API Gateway + Lambda with DynamoDB-backed handlers.

## Default Shape

- API Gateway REST API
- one Lambda per route
- route-level request validation where needed
- Cognito authorizer only when required by the user or existing stack conventions
- environment variables passed into handlers for table names and feature flags

## Baseline Example

```ts
import * as path from 'node:path';
import * as cdk from 'aws-cdk-lib';
import * as apiGateway from 'aws-cdk-lib/aws-apigateway';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as lambdaNodejs from 'aws-cdk-lib/aws-lambda-nodejs';

const api = new apiGateway.RestApi(this, 'UsersApi', {
    restApiName: `UsersApi${props.deploymentStage}`,
    defaultCorsPreflightOptions: {
        allowOrigins: apiGateway.Cors.ALL_ORIGINS,
        allowMethods: apiGateway.Cors.ALL_METHODS,
    },
});

const users = api.root.addResource('users');
const userById = users.addResource('{id}');

const createUserFn = new lambdaNodejs.NodejsFunction(this, 'CreateUserFn', {
    runtime: lambda.Runtime.NODEJS_24_X,
    architecture: lambda.Architecture.ARM_64,
    entry: path.join(__dirname, '../../src/handlers/users/create-user.handler.ts'),
    handler: 'CREATE_USER_HANDLER',
    environment: {
        USERS_TABLE_NAME: usersTable.tableName,
    },
});

const getUserFn = new lambdaNodejs.NodejsFunction(this, 'GetUserFn', {
    runtime: lambda.Runtime.NODEJS_24_X,
    architecture: lambda.Architecture.ARM_64,
    entry: path.join(__dirname, '../../src/handlers/users/get-user.handler.ts'),
    handler: 'GET_USER_HANDLER',
    environment: {
        USERS_TABLE_NAME: usersTable.tableName,
    },
});

usersTable.grantReadWriteData(createUserFn);
usersTable.grantReadData(getUserFn);

users.addMethod('POST', new apiGateway.LambdaIntegration(createUserFn));
userById.addMethod('GET', new apiGateway.LambdaIntegration(getUserFn), {
    requestParameters: {
        'method.request.path.id': true,
    },
});
```

## Guidance

- If the repository already has a route composition abstraction, adapt this pattern to match it.
- If no abstraction exists, keep the route tree explicit with `addResource` and introduce one reusable place for API composition once the route count grows.
- Grant read access to query handlers and read/write access to mutating handlers.
- Add request models for `POST` and `PUT` when gateway-level validation is part of the architecture.
- If the repository does not have a response helper like `RestResult`, generate one shared response utility instead of repeating inline response objects.
