# Middleware and Messages

Use the displayed Storm helper imports when installed. Without Storm, use the application helper imports described in `general-cdk.md`; the application flow stays the same.

Use common middleware and message middleware in the same style as REST, with WebSocket types and parsing. Do not apply HTTP content-type validators or REST event normalizers to socket messages.

## Typed Input

```ts
// src/models/socket.model.ts
import type { APIGatewayProxyWebsocketEventV2 } from 'aws-lambda';

export type UpdateMessage = { action: 'updates'; text: string };
export type SocketEvent = APIGatewayProxyWebsocketEventV2 & {
    message?: UpdateMessage;
};
```

For this new example contract, clients send `{"action":"updates","text":"An update"}`. Preserve existing message schemas rather than replacing them with this example.

## Validation Middleware

```ts
// src/middlewares/socket.middleware.ts
import type { MiddlewareObj } from '@middy/core';
import type { APIGatewayProxyStructuredResultV2 } from 'aws-lambda';
import { SocketResult } from '@stack-shifter/storm-stack';
import type { SocketEvent } from '../models/socket.model';

export const validateUpdateMessage = (): MiddlewareObj<SocketEvent, APIGatewayProxyStructuredResultV2> => {
    return {
        before: (request) => {
            let value: unknown;
            try {
                value = JSON.parse(request.event.body ?? '');
            } catch {
                return SocketResult.badRequest('Invalid JSON.');
            }
            if (typeof value !== 'object' || value === null ||
                !('action' in value) || value.action !== 'updates' ||
                !('text' in value) || typeof value.text !== 'string' ||
                value.text.trim().length === 0) {
                return SocketResult.badRequest('An updates action and text are required.');
            }
            request.event.message = { action: 'updates', text: value.text.trim() };
        },
    };
};
```

Use existing schema validation when available, assigning its parsed output to `event.message`. Choose message-size limits from requirements. The common error middleware below uses the same result contract; unexpected details stay in logs.

```ts
// Also in src/middlewares/socket.middleware.ts
export const handleSocketError = (): MiddlewareObj<SocketEvent, APIGatewayProxyStructuredResultV2> => {
    return {
        onError: (request) => {
            console.error('WebSocket request failed', { requestId: request.context.awsRequestId });
            request.response = SocketResult.internalServerError('Unable to process message.');
        },
    };
};
```

Use the project's shared logger in generated code. Do not log full events or tokens.

## Handler Composition

```ts
// src/handlers/socket.handler.ts
import middy from '@middy/core';
import type { Handler, APIGatewayProxyStructuredResultV2 } from 'aws-lambda';
import type { SocketEvent } from '../models/socket.model';
import { connect, disconnect, unknownAction, publishUpdate } from '../controllers/socket.controller';
import { handleSocketError, validateUpdateMessage } from '../middlewares/socket.middleware';

type SocketHandler = Handler<SocketEvent, APIGatewayProxyStructuredResultV2>;

const withCommonMiddleware = (handler: SocketHandler) =>
    middy(handler).use(handleSocketError());

const withMessageMiddleware = (handler: SocketHandler) =>
    withCommonMiddleware(handler).use(validateUpdateMessage());

export const connectHandler = withCommonMiddleware(connect);
export const disconnectHandler = withCommonMiddleware(disconnect);
export const defaultHandler = withCommonMiddleware(unknownAction);
export const updatesHandler = withMessageMiddleware(publishUpdate);
```

Check types and short-circuit behavior against the installed Middy version. Rejected validation must not invoke controllers. Keep the request-authorizer handler separate from proxy response middleware.

## Responses and Delivery

Storm `SocketResult` methods are lowercase: `ok`, `badRequest`, `unauthorized`, `forbidden`, and `internalServerError`, among others. These create Lambda integration results; they do not broadcast messages.

A successful connect result accepts the handshake. For message routes, `returnResponse: true` enables the route response used by the example for acknowledgments and errors. Explicit `postToConnection` calls deliver notifications to selected clients independently. Test both paths: returning `SocketResult.ok()` alone is not a callback send or proof of delivery.

`$default` handles unmatched actions, including payloads that cannot select a custom route. Return a clear unsupported-message result. Keep callback payloads and acknowledgments distinct; preserve the application's error envelope when extending an existing API.
