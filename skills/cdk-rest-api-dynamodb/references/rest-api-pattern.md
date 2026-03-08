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
import { USERS_HANDLER } from '../src/handlers/users.handler';

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
    handler: USERS_HANDLER.SAVE,
    environment: {
        USERS_TABLE_NAME: usersTable.tableName,
    },
});

const getUserFn = new lambdaNodejs.NodejsFunction(this, 'GetUserFn', {
    runtime: lambda.Runtime.NODEJS_24_X,
    architecture: lambda.Architecture.ARM_64,
    entry: path.join(__dirname, '../../src/handlers/users/get-user.handler.ts'),
    handler: USERS_HANDLER.GET_BY_ID,
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

## HANDLER Registry Pattern

When using a route composition abstraction such as `RestServerlessApi`, handler files should export a typed registry constant so `handlerName` values in the stack are never bare strings. A rename in the handler file will not be caught at compile time without the registry.

```ts
// src/handlers/users.handler.ts
export const USERS_HANDLER = {
    GET_BY_ID: 'getByIdUserHandler',
    QUERY: 'queryUserHandler',
    SAVE: 'saveUserHandler',
    UPDATE_BY_ID: 'updateByIdUserHandler',
    DELETE: 'deleteUserHandler',
} as const;
```

Import and use the registry in the stack for all route wiring:

```ts
import { getDefaultLambdaEnvironment, getStackLambdaName } from './constructs/node-lambda';
import { USERS_HANDLER } from '../src/handlers/users.handler';
import { UPDATE_HANDLER } from '../src/handlers/update.handler';

const lambdaName = (name: string) => getStackLambdaName(name, props.deploymentStage);
const handlerPath = (p: string) => path.join(__dirname, '..', p);

// POST route
api.post({
    routePath: '/users',
    lambdaName: lambdaName('UsersPost'),
    filePath: handlerPath('src/handlers/users.handler.ts'),
    handlerName: USERS_HANDLER.SAVE,
    description: '/users',
    scopes: writeScopes,
});

// GET list route
api.get({
    routePath: '/users',
    lambdaName: lambdaName('UsersQuery'),
    filePath: handlerPath('src/handlers/users.handler.ts'),
    handlerName: USERS_HANDLER.QUERY,
    description: '/users',
    scopes: readScopes,
});

// GET by ID route
api.getById({
    routePath: '/users/{id}',
    lambdaName: lambdaName('UsersGetById'),
    filePath: handlerPath('src/handlers/users.handler.ts'),
    handlerName: USERS_HANDLER.GET_BY_ID,
    description: '/users/{id}',
    scopes: readScopes,
});

// Public route — override the default Cognito authorizer
api.get({
    routePath: '/share-links/{token}/updates',
    lambdaName: lambdaName('UpdatesQueryPublicByLink'),
    filePath: handlerPath('src/handlers/update.handler.ts'),
    handlerName: UPDATE_HANDLER.QUERY_PUBLIC_BY_LINK,
    description: '/share-links/{token}/updates',
    authorizer: undefined,
});
```

## Guidance

- If the repository already has a route composition abstraction, adapt this pattern to match it.
- If no abstraction exists, keep the route tree explicit with `addResource` and introduce one reusable place for API composition once the route count grows.
- Never write `handlerName` as a bare string literal — always import and use the `HANDLER` registry constant from the handler file.
- Grant read access to query handlers and read/write access to mutating handlers.
- Add request models for `POST` and `PUT` when gateway-level validation is part of the architecture.
- If the repository does not have a response helper like `RestResult`, generate one shared response utility instead of repeating inline response objects.
