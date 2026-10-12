# Middleware Pattern

Use this reference when adding or changing reusable Middy middleware in a repository-backed API.

## Goal

Keep handlers declarative by centralizing request normalization, authorization, validation, and middleware error translation.

## Middleware Responsibilities

- `injectLambdaContext()` — adapter that bridges `@aws-lambda-powertools/logger/middleware` with Middy v7's handler type
- `authorization.middleware.ts` normalizes Cognito claims and enforces allowed groups
- `validation.middleware.ts` validates headers, path params, query params, and body with Zod
- `http-error.middleware.ts` converts parser and middleware exceptions into `RestResult` responses

## Handler Composition Pattern

The logger, context adapter, error middleware, authorization middleware, validators, and controllers below are application modules. Reuse
their existing imports. Include the logging adapter only when the project uses it; verify the handler type against the installed Middy
version.

Use `withCommonMiddleware` and `withWriteMiddleware` factory helpers inside each handler file rather than repeating the base chain inline.

```ts
import middy from '@middy/core';
import type { APIGatewayProxyHandler } from 'aws-lambda';
import httpEventNormalizer from '@middy/http-event-normalizer';
import httpHeaderNormalizer from '@middy/http-header-normalizer';

const withCommonMiddleware = (handler: APIGatewayProxyHandler) =>
	middy(handler)
		.use(httpHeaderNormalizer())
		.use(httpEventNormalizer())
		.use(injectLambdaContext(logger, { logEvent: false }))
		.use(handleHttpError());

// In this example validateBody owns JSON parsing as well as schema validation.
const withWriteMiddleware = (handler: APIGatewayProxyHandler) => withCommonMiddleware(handler);
```

Append route-specific middleware to the returned instance:

```ts
export const getByIdEntityHandler = withCommonMiddleware(getByIdEntityController)
	.use(authorizedGroup({ allowed: [allowedGroup] }))
	.use(validatePathParameters(entityPathSchema));

export const saveEntityHandler = withWriteMiddleware(saveEntityController)
	.use(authorizedGroup({ allowed: [allowedGroup] }))
	.use(validateHeaders(jsonContentTypeSchema))
	.use(validateBody(entityPostDtoSchema));
```

## Optional Middy JSON Body Parser

`@middy/http-json-body-parser` is a valid alternative when it fits the repository's middleware chain. Preserve an existing parser or choose
it when separating JSON decoding from schema validation improves clarity. It is optional, not prohibited or required.

For example, compose the parser after authorization and before body validation:

```ts
import httpJsonBodyParser from '@middy/http-json-body-parser';

export const saveEntityHandler = withCommonMiddleware(saveEntityController)
	.use(authorizedGroup({ allowed: [allowedGroup] }))
	.use(httpJsonBodyParser())
	.use(validateBody(entityPostDtoSchema));
```

In this variant, the parser decodes JSON and `validateBody` validates the resulting object and preserves Zod output. Keep parser
content-type checks enabled unless an explicit header validator owns them; only then use `disableContentTypeError: true` if supported by the
installed version. Translate parser failures through the existing error middleware. If the project uses a write helper, place the parser
there only when the resulting authorization/parsing order still matches the route's contract. Do not add or remove a parser dependency
without the appropriate task authorization.

## Guidance

### Parsed Query Output Example

This standalone Middy example validates a query once, preserves its numeric conversion/default, and consumes that output in the controller.
Compose the project's authorization and error middleware around it as required.

```ts
import middy from '@middy/core';
import type { APIGatewayProxyEvent, APIGatewayProxyResult } from 'aws-lambda';
import { RestResult } from '@stack-shifter/storm-stack/framework';
import { z } from 'zod';

const recordsQuerySchema = z.object({
	page: z.coerce.number().int().min(1).max(Number.MAX_SAFE_INTEGER).default(1),
});

type RecordsQuery = z.output<typeof recordsQuerySchema>;

export const createRecordsHandler = (corsOrigin: string) => {
	const validateQuery: middy.MiddlewareObj<APIGatewayProxyEvent, APIGatewayProxyResult> = {
		before: async ({ event }) => {
			const result = recordsQuerySchema.safeParse(event.queryStringParameters ?? {});

			if (!result.success) {
				return RestResult.badRequest('Invalid query parameters.', { origin: corsOrigin });
			}

			// Replace raw strings with Zod's converted values and defaults for the controller.
			Object.assign(event, { queryStringParameters: result.data });
		},
	};

	const controller = async (event: APIGatewayProxyEvent): Promise<APIGatewayProxyResult> => {
		// Gateway's declared string-map type does not describe the middleware's validated output.
		const query = event.queryStringParameters as unknown as RecordsQuery;

		return RestResult.ok({ page: query.page }, { origin: corsOrigin });
	};

	return middy(controller).use(validateQuery);
};
```

The assertion is permitted only because this handler always runs the matching validator first. It does not perform conversion or validate
untrusted input. Preserve an existing typed validated-event approach instead when the project already provides one. Extend this example's
minimal error response with the project's stable reason and field-error contract.

- If the repository already has middleware helpers, extend them.
- If it does not, generate middleware as shared reusable units instead of embedding auth or validation in handlers.
- Keep middleware reusable; deliberate replacement of validated request fields is an explicit boundary operation.
- Put request-shape validation in middleware, not controllers.
- Read verified Gateway claims through a shared accessor when identity is needed. Do not invent claims from client headers or require claim
  mutation when the existing claims suffice.
- Return `RestResult` values for expected request failures such as bad input or missing auth context.
- Log unexpected middleware failures through the shared logger before returning a generic 500 response.

## Validation and Error Contract

- Preserve parsed schema outputs, including coercions and defaults, in the typed input consumed by controllers.
- A simple project-local approach assigns successful Zod output back to the corresponding event field. Comment that this replaces raw values
  with converted values/defaults. Controllers may assert the validated route-specific type; `as Type` alone performs no conversion. Do not
  parse the same input again just to recover transformed values.
- Choose one JSON parsing owner. If validation already parses string bodies and handles malformed JSON, do not add a redundant parser
  dependency. If Middy's parser is used, preserve its errors and ordering, and do not parse again in validation.
- Test malformed JSON, missing bodies, invalid path/query values, and authorization failures. Failed requests must not reach controllers.
- The write example delegates content-type checks to `validateHeaders(jsonContentTypeSchema)`. Where a separate parser is used, keep its
  content-type checks enabled unless another boundary validator owns that check. Preserve the project's accepted content-type contract.
- Verify error middleware ordering against the installed version so parser and middleware exceptions use the same `RestResult` contract as
  controllers.
