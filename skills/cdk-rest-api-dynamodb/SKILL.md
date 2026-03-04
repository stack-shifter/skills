---
name: cdk-rest-api-dynamodb
description: Designs and implements REST APIs on AWS using the target repository's CDK patterns first, with AWS Lambda handlers and DynamoDB as the primary datastore. Use this whenever the user wants to add or modify API Gateway REST endpoints, Lambda handlers, Cognito auth, request models, reusable CDK constructs, or DynamoDB-backed CRUD routes, even if they only ask for "an endpoint", "a handler", "a table", or "some CDK wiring".
---

## Purpose

Use this skill to design and implement DynamoDB-backed REST APIs on AWS in a way that fits the target repository, with DynamoDB as the default datastore.

When local reusable CDK constructs exist, use them as the source of truth. When they do not, use the references in this skill as portable patterns to generate an equivalent structure rather than one-off code.

Portable references live in `references/`. Load only the patterns needed for the task:

- `references/rest-api-pattern.md` for API Gateway REST route composition
- `references/node-lambda-pattern.md` for Lambda defaults and bundling shape
- `references/dynamodb-pattern.md` for DynamoDB table and access-pattern guidance
- `references/importer-pattern.md` for importing existing AWS resources when no local importer helper exists
- `references/response-pattern.md` for standardized API responses when no local `RestResult`-style helper exists
- `references/jwt-pattern.md` for JWT verification when no local token service exists
- `references/runtime-composition-pattern.md` for `src/app.ts`-style singleton dependency wiring and repository context composition
- `references/middleware-pattern.md` for reusable Middy middleware such as auth, validation, and HTTP error translation
- `references/services-pattern.md` for portable logger, storage, notification, and mapper service design
- `references/utilities-pattern.md` for shared response helpers, error types, status codes, cursor helpers, and other runtime utilities

## Portability Rule

This skill should work across repositories. Do not assume a specific folder such as `lib/constructs/` exists, and do not assume the target repository already has the same abstractions.

The source of truth order is:

1. The target repository's current architecture, naming, and abstractions
2. The skill's documented patterns
3. The inline examples in this skill

If local code differs from the skill examples, follow local code and use the examples only as design guidance.

## Repository Discovery

Start every use of this skill with a short discovery pass before proposing code.

Search for:

- API-related CDK code
- Lambda wrapper constructs or helpers
- DynamoDB constructs or direct table creation patterns
- resource importer helpers for existing infrastructure
- runtime composition files such as `src/app.ts`, `src/data/context.ts`, or dependency containers
- reusable middleware for auth, validation, JSON parsing, and HTTP error handling
- shared services for logging, storage, notifications, token verification, and mapping
- standardized API response utilities
- JWT verification helpers or token services
- Stack files that already compose routes, auth, models, or permissions

Likely locations:

- `lib/constructs/`
- `lib/`
- `infra/constructs/`
- `infra/`
- `packages/*/lib/constructs/`
- `packages/*/infra/`
- stack files that instantiate `RestApi`, `Lambda`, `NodejsFunction`, or `TableV2`

After discovery, choose one mode and state it:

- `Existing pattern mode`: extend the repository's existing abstractions
- `Pattern generation mode`: generate the missing reusable pattern because no relevant abstraction exists

## Existing Pattern Mode

Use this mode when the repo already has abstractions similar to the current repository, such as:

- `lib/constructs/rest-api.ts` for API Gateway REST resources and route wiring
- `lib/constructs/node-lambda.ts` for Lambda defaults and bundling
- `lib/constructs/dynamodb.ts` for DynamoDB `TableV2` creation
- `lib/constructs/api-importer.ts` for importing existing AWS resources into stacks
- `lib/constructs/api-models.ts` for the route and API prop shapes
- `src/app.ts` or `src/dependencies/` for singleton runtime wiring
- `src/data/context.ts` for repository aggregation
- `src/middlewares/` for authorization, validation, and error middleware
- `src/services/logger.service.ts` for shared logging
- `src/services/storage/storage.service.ts` for presigned upload or download URLs
- `src/services/messaging/` for SES or notification integrations
- `src/services/mapping/` or `src/utilities/mappers/` for DTO and query translation
- `src/utilities/rest-result.ts` for standardized API Gateway responses
- `src/services/token.service.ts` for JWT validation built on `aws-jwt-verify`

Default to DynamoDB as the datastore unless the user explicitly wants something else.

## Pattern Generation Mode

Use this mode when the target repository does not already provide a reusable abstraction for one or more layers.

In this mode:

- use the references as guidance for the shape of the generated code, not as a requirement that exact files or class names exist
- introduce the smallest reusable abstraction that makes future endpoints easier to add
- prefer generating a reusable route, repository, middleware, or service pattern over embedding everything in a single handler
- keep generated names and folders aligned with the target repository's conventions

## Working Rules

1. Read the relevant local construct or stack code before proposing or writing endpoint infrastructure.
2. Prefer code snippets that mirror the actual construct APIs in the target repo instead of generic CDK examples.
3. If the repository already has a route composition abstraction such as `RestServerlessApi`, use it. If not, generate a small reusable pattern instead of scattering raw CDK logic.
4. Model data access around the repository's existing DynamoDB table pattern when it exists. Otherwise generate a consistent table and repository pattern.
5. Keep examples aligned with the repository defaults: Node.js 24.x, ARM64, Middy handlers, Zod validation, and API Gateway REST API.
6. When a stack needs to attach to existing infrastructure, prefer the local importer helper over direct `from*` imports scattered throughout the stack.
7. When controllers or handlers return API Gateway responses, prefer the local response utility over ad hoc response objects.
8. Prefer API Gateway Cognito authorizers for route protection when the repository already uses them for that endpoint shape. Use `aws-jwt-verify` only when JWT verification must happen inside Lambda code rather than at the API Gateway authorizer layer.
9. Reuse existing runtime composition in `src/app.ts`, `src/dependencies/`, or equivalent when it exists. If it does not, create a lightweight equivalent rather than instantiating AWS clients, repositories, or services inside handlers.
10. When local middleware exists, extend the existing Middy chain instead of embedding auth, validation, or error translation logic directly in controllers. If it does not, generate reusable middleware rather than inline checks.
11. Prefer small reusable services for cross-cutting concerns such as logging, storage, notifications, and DTO mapping.

## Default Assumptions

- API type: API Gateway REST API
- Compute: one Lambda per route unless the change is clearly tiny
- Datastore: DynamoDB
- Lambda runtime defaults: those from an existing Lambda wrapper when present, otherwise a consistent Node.js Lambda baseline
- Auth: inherit existing stack defaults; if auth is unclear, prefer route-level compatibility with Cognito authorizers
- Validation: use API Gateway request parameters/models plus existing handler-side validation
- JWT verification library: prefer `aws-jwt-verify` only for in-Lambda verification paths

If the user gives constraints that conflict with these defaults, adapt and state the change.

## Preferred `src/` Structure

Unless the target repository already has a different established layout, use an application structure like this:

```text
src/
├── controllers/
├── data/
│   └── context.ts
├── dependencies/
├── handlers/
├── middlewares/
├── models/
│   └── validation/
├── services/
├── utilities/
    └── mappers/
└── app.ts
```

Treat this as a conceptual layout, not a hard requirement:

- `src/handlers/` for Lambda entrypoints and Middy wiring
- `src/controllers/` for request orchestration and `RestResult` responses
- `src/services/` for business logic, auth helpers, and cross-cutting domain operations
- `src/data/` for repositories and datastore access
- `src/data/context.ts` for a shared repository aggregate such as `DatabaseContext` or `DataContext`
- `src/models/validation/` for Zod schemas and request validation shapes
- `src/middlewares/` for reusable Middy middleware
- `src/dependencies/` for dependency wiring and composition roots
- `src/utilities/` for shared helpers
- `src/utilities/mappers/` for mapping and transformation helpers
- `src/app.ts` for warm Lambda singleton wiring when the repository uses module-scope dependency exports

When generating endpoint work, prefer this file flow:

1. handler in `src/handlers/`
2. controller in `src/controllers/`
3. service in `src/services/` if business logic is more than trivial
4. repository in `src/data/` for DynamoDB access
5. validation schema in `src/models/validation/` when request validation is needed
6. dependency or context wiring in `src/app.ts` or `src/data/context.ts` when new repositories or services are introduced

If the repository already uses a different but coherent `src/` structure, follow that instead of forcing this one.

## Construct-Centered Guidance

### 1. Datastore design should start from a reusable table pattern

Use the local construct rather than raw `dynamodb.TableV2` when adding a new table. If the repository does not have one, create a reusable table pattern instead of repeating table setup inline.

```ts
import { DynamoDBTable } from '../constructs/dynamodb';

const usersTable = new DynamoDBTable(this, 'UsersTable', {
    tableName: `Users${props.deploymentStage}`,
    globalIndexOverload: 1,
    timeToLiveAttribute: 'ExpiresAt',
    deletionProtection: props.deploymentStage === 'Prod',
}).table;
```

Important pattern details:

- Primary keys are fixed as `PK` and `SK`
- `globalIndexOverload: 1` creates `GSI1` with `GSI1PK` and `GSI1SK`
- `globalIndexAttribute` creates named GSIs from explicit attribute names
- default `removalPolicy` is `RETAIN`

Design endpoint storage and access patterns around those keys instead of inventing a different schema shape in the skill output.

### 2. Route composition should be centralized

If the repository already has a route composition abstraction such as `RestServerlessApi`, extend it. Otherwise create one reusable route composition layer and keep route registration centralized.

Common route helpers are:

- `get`
- `getById`
- `post`
- `put`
- `delete`

Set shared route defaults once when multiple endpoints share auth, environment, table access, memory, or VPC settings.

```ts
import { RestServerlessApi } from '../constructs/rest-api';

const api = new RestServerlessApi(this, 'IdentityApi', {
    name: `IdentityApi${props.deploymentStage}`,
    corsOrigin: process.env.CORS_ORIGIN || '*',
    logging: {
        enabled: true,
        deploymentStage: props.deploymentStage,
        notificationEmail: process.env.NOTIFICATION_EMAIL,
    },
});

api.setDefaultRouteOptions({
    authorizer: api.createCognitoAuthorizer(userPool),
    grantTableAccess: [usersTable],
    environmentVariables: {
        USERS_TABLE_NAME: usersTable.tableName,
    },
});
```

Important pattern behavior:

- `routePath` must start with `/`
- default route options merge with per-route options
- setting `authorizer: undefined` on a route explicitly removes the default authorizer
- DynamoDB permissions are granted automatically from `grantTableAccess`
- `GET` routes get read permissions; `POST`, `PUT`, and `DELETE` get read/write permissions
- GSI query and scan permissions are added on `/index/*`

### 3. Use route snippets based on the local `RestRouteOptions`

When documenting or generating an endpoint, prefer snippets like these.

Create route:

```ts
import * as path from 'node:path';
import { CREATE_USER_HANDLER } from '../../src/handlers/users/create-user.handler';

api.post({
    routePath: '/users',
    lambdaName: `CreateUser${props.deploymentStage}`,
    filePath: path.join(__dirname, '../../src/handlers/users/create-user.handler.ts'),
    handlerName: CREATE_USER_HANDLER,
    description: 'Create a user record',
    model: createUserModel,
});
```

List route with query parameters:

```ts
import { LIST_USERS_HANDLER } from '../../src/handlers/users/list-users.handler';

api.get({
    routePath: '/users',
    lambdaName: `ListUsers${props.deploymentStage}`,
    filePath: path.join(__dirname, '../../src/handlers/users/list-users.handler.ts'),
    handlerName: LIST_USERS_HANDLER,
    description: 'List users',
    requestParameters: {
        'method.request.querystring.cursor': false,
        'method.request.querystring.limit': false,
    },
});
```

Get-by-id route with path parameter:

```ts
import { GET_USER_HANDLER } from '../../src/handlers/users/get-user.handler';

api.getById({
    routePath: '/users/{id}',
    lambdaName: `GetUser${props.deploymentStage}`,
    filePath: path.join(__dirname, '../../src/handlers/users/get-user.handler.ts'),
    handlerName: GET_USER_HANDLER,
    description: 'Get a user by id',
    requestParameters: {
        'method.request.path.id': true,
    },
});
```

Public route that overrides a default authorizer:

```ts
api.get({
    routePath: '/health',
    lambdaName: `Health${props.deploymentStage}`,
    filePath: path.join(__dirname, '../../src/handlers/health.handler.ts'),
    handlerName: HEALTH_HANDLER,
    description: 'Health check',
    authorizer: undefined,
});
```

### 4. `NodeLambda` defines the Lambda baseline

If you need a standalone Lambda or need to explain route internals, use the local construct shape.

```ts
import { NodeLambda } from '../constructs/node-lambda';

const fn = new NodeLambda(this, 'CreateUserFn', {
    lambdaName: `CreateUser${props.deploymentStage}`,
    filePath: path.join(__dirname, '../../src/handlers/users/create-user.handler.ts'),
    handlerName: CREATE_USER_HANDLER,
    description: 'Create a user record',
    environmentVariables: {
        USERS_TABLE_NAME: usersTable.tableName,
    },
}).function;
```

Repository-specific defaults already baked in:

- runtime: `NODEJS_24_X`
- architecture: `ARM_64`
- memory: `256`
- timeout: `15`
- tracing: active
- log retention: three months
- `@aws-sdk/*` excluded from bundle

Do not restate conflicting defaults in skill-generated examples.

### 5. Prefer `Importer` for existing infrastructure

If the repository exposes an importer helper like `lib/constructs/api-importer.ts`, use it when wiring a stack to resources that already exist.

Prefer patterns like:

```ts
import { Importer } from '../constructs/api-importer';

const usersTable = Importer.getDynamoDbTable(this, process.env.DYNAMODB_TABLE!);
const userPool = Importer.getCognitoUserPoolById(this, process.env.COGNITO_USER_POOL_ID!);
const apiDomain = Importer.getApiGatewayDomainName(
    this,
    process.env.API_DOMAIN_NAME!,
    process.env.API_DOMAIN_TARGET!,
    process.env.API_DOMAIN_HOSTED_ZONE_ID!,
);
```

Use the importer when:

- the stack is extending an existing table, user pool, API, domain, or other shared resource
- the repository already centralizes `from*` imports behind an importer helper

Do not replace a repository-wide importer pattern with direct `fromTableName`, `fromUserPoolId`, or similar calls unless the user explicitly asks for that.

### 6. Prefer `RestResult` for API responses

If the repository has a response helper like `src/utilities/rest-result.ts`, use it as the default response format for controllers and handlers.

Prefer methods such as:

- `RestResult.Ok(...)`
- `RestResult.Created(..., location)`
- `RestResult.NoContent()`
- `RestResult.BadRequest(...)`
- `RestResult.NotFound(...)`
- `RestResult.Conflict(...)`
- `RestResult.InternalServerError(...)`

This keeps CORS headers, content types, status codes, and error body shapes consistent across endpoints.

If no local response helper exists, use the fallback references and produce standard `APIGatewayProxyResult` objects instead.

### 7. Prefer `aws-jwt-verify` only when Cognito authorizers are not handling auth

If a route is already protected by an API Gateway Cognito authorizer, do not add duplicate JWT verification in the Lambda by default.

Use `aws-jwt-verify` only when the repository validates Cognito JWTs or other OIDC JWTs inside Lambda code instead of relying on API Gateway Cognito authorizers for that route.

Prefer local abstractions like:

```ts
import { TokenService, TokenType } from '../services/token.service';

const tokenService = TokenService.forCognito(
    process.env.COGNITO_USER_POOL_ID!,
    process.env.COGNITO_CLIENT_ID!,
    process.env.AWS_REGION,
    TokenType.ACCESS_TOKEN,
);

const payload = await tokenService.validateToken(token);
```

Use this preference because it keeps JWT verification aligned with AWS and Cognito semantics, especially issuer, audience, token use, and JWKS handling.

Decision rule:

- if the route uses `createCognitoAuthorizer(...)` or an equivalent API Gateway Cognito authorizer, prefer that and do not add `aws-jwt-verify` in the handler unless the user explicitly needs claim re-verification inside Lambda
- if the route uses a custom token authorizer, Lambda-side verification flow, or non-Cognito OIDC JWT validation, prefer `aws-jwt-verify`
- if auth is already fully enforced at API Gateway and the handler only needs claims passed through context, do not introduce `aws-jwt-verify`

Avoid:

- duplicating Cognito authorizer validation inside Lambda without a clear reason
- hand-rolled signature verification
- using `decode` as if it were verification
- introducing another JWT library when the repo already uses `aws-jwt-verify`

If the repository does not already have a token service or helper, load `references/jwt-pattern.md`.

## Fallback Mode

Use this mode only when the repository does not already expose relevant constructs or composition helpers.

When using fallback mode, load the relevant file from `references/` instead of relying only on the inline examples in this skill.

Also use the fallback references when the repository lacks:

- an importer helper for existing resources
- a standard API response helper such as `RestResult`
- a JWT verification helper or token service
- a runtime dependency container such as `src/app.ts`
- reusable middleware for auth, validation, or HTTP error handling
- shared services for logging, storage, notifications, or mapping
- shared utility modules for errors, status codes, or cursor helpers

In fallback mode:

- generate a reusable API construct or stack pattern instead of one-off infrastructure code
- use API Gateway REST API unless the user explicitly wants HTTP API
- use one Lambda per route by default
- use a DynamoDB table with `PK` and `SK`
- keep examples concise and portable

Portable baseline:

```ts
const table = new dynamodb.TableV2(this, 'UsersTable', {
    partitionKey: { name: 'PK', type: dynamodb.AttributeType.STRING },
    sortKey: { name: 'SK', type: dynamodb.AttributeType.STRING },
});

const createUserFn = new lambdaNodejs.NodejsFunction(this, 'CreateUserFn', {
    runtime: lambda.Runtime.NODEJS_24_X,
    architecture: lambda.Architecture.ARM_64,
    entry: path.join(__dirname, '../../src/handlers/create-user.handler.ts'),
    handler: 'CREATE_USER_HANDLER',
    environment: {
        USERS_TABLE_NAME: table.tableName,
    },
});

table.grantReadWriteData(createUserFn);

api.root.addResource('users').addMethod('POST', new apiGateway.LambdaIntegration(createUserFn));
```

If the repository later adds its own constructs, stop using the fallback and move back to local construct mode.

## Request Modeling

When the user asks for request validation, use the local API Gateway route support:

- `model` for JSON request-body validation
- `requestParameters` for path/query/header requirements
- `post` uses body validation
- `get` and `getById` use parameter validation
- `put` supports mixed validation

If the repo already has API models or Zod schemas for the feature area, reuse them instead of inventing parallel validation definitions.

If the repo already has a standard API response helper, reuse it in the controller or handler examples.

## DynamoDB-First Endpoint Design

When designing endpoints, bias toward patterns that fit the table construct and common single-table usage:

- Store entities under `PK` and `SK`
- Use GSIs for alternate lookup patterns
- Pass tables through `grantTableAccess`
- Pass table names through `environmentVariables`
- Keep Lambda handlers thin and push key construction/query logic into repositories or services

Prefer examples such as:

```ts
api.put({
    routePath: '/users/{id}',
    lambdaName: `UpdateUser${props.deploymentStage}`,
    filePath: path.join(__dirname, '../../src/handlers/users/update-user.handler.ts'),
    handlerName: UPDATE_USER_HANDLER,
    description: 'Update a user',
    requestParameters: {
        'method.request.path.id': true,
    },
    model: updateUserModel,
});
```

That keeps the skill focused on endpoint composition and DynamoDB-backed handler wiring, which is the main use case for this skill.

## Response Style

When using this skill, produce:

1. A short statement of the endpoint(s) being added or changed
2. A note saying whether you are using `Local construct mode` or `Fallback mode`
3. CDK snippets based on the local constructs when present, or the portable baseline when not
4. Any handler, model, or repository follow-on work needed to make the endpoint functional
5. Explicit assumptions where auth, table shape, or validation is ambiguous

Avoid generic AWS guidance when a repository-specific construct snippet would answer the request better. Treat the repository's code as authoritative and the skill as workflow guidance, not as a competing source of truth.
