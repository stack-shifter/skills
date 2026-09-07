# Connection Lifecycle

Keep connection operations behind a repository. The example stores one authorized scope per connection; adapt this to actual membership requirements without trusting message-supplied recipient scope.

## Repository Contract

The application `ConnectionRepository` implements these methods using the existing datastore client. Query construction and keys belong in that implementation, not in handlers or controllers.

```ts
// Export this type from src/data/repositories/connection.repository.ts
export type Connection = {
    connectionId: string;
    subjectId: string;
    scopeId: string;
    expiresAt: number; // epoch seconds
};

// Method signatures on the application repository:
// save(connection: Connection): Promise<void>
// findById(connectionId: string): Promise<Connection | null>
// listActiveByScope(scopeId: string, now: number): Promise<Connection[]>
// remove(connectionId: string): Promise<void>
```

`remove` succeeds when a record is already absent. `listActiveByScope` filters expiry and traverses pagination; for large fanout expose pages instead of loading every connection. If the datastore supports TTL, use it for eventual cleanup, not as an authorization or freshness check.

## Connect and Disconnect

These controller excerpts share the imports below. Connect context comes from the request authorizer and is validated before saving.

```ts
// src/controllers/socket.controller.ts
import { SocketResult } from '@stack-shifter/storm-stack';
import type { SocketEvent } from '../models/socket.model';
import { context, messagingService, updatePermissions } from '../app';

export const connect = async (event: SocketEvent) => {
    const claims = (event.requestContext as typeof event.requestContext & {
        authorizer?: Record<string, unknown>;
    }).authorizer;
    const expiresAt = Number(claims?.expiresAt);
    if (typeof claims?.subjectId !== 'string' || typeof claims?.scopeId !== 'string' ||
        !Number.isFinite(expiresAt) || expiresAt <= Math.floor(Date.now() / 1000)) {
        return SocketResult.unauthorized('Valid connection identity is required.');
    }
    await context.connections.save({
        connectionId: event.requestContext.connectionId,
        subjectId: claims.subjectId,
        scopeId: claims.scopeId,
        expiresAt,
    });
    return SocketResult.ok();
};

export const disconnect = async (event: SocketEvent) => {
    await context.connections.remove(event.requestContext.connectionId);
    return SocketResult.ok();
};

export const unknownAction = async () => {
    return SocketResult.badRequest('Unsupported message action.');
};
```

Do not send to the new connection from its connect handler; the handshake may not have completed. Disconnect is best-effort, so callback cleanup and expiry checks remain necessary.

## Publish an Update

```ts
// Also in src/controllers/socket.controller.ts
export const publishUpdate = async (event: SocketEvent) => {
    const sender = await context.connections.findById(event.requestContext.connectionId);
    const now = Math.floor(Date.now() / 1000);
    if (!sender || sender.expiresAt <= now) {
        return SocketResult.unauthorized('Connection expired or unavailable.');
    }
    if (!event.message) return SocketResult.badRequest('Validated message required.');
    if (!await updatePermissions.canPublish(sender.subjectId, sender.scopeId)) {
        return SocketResult.forbidden('Publishing is not allowed.');
    }
    const recipients = await context.connections.listActiveByScope(sender.scopeId, now);
    const result = await messagingService.sendUpdate(recipients, sender.connectionId, {
        type: 'update',
        text: event.message.text,
    });
    return SocketResult.ok({ type: 'ack', ...result });
};
```

The example excludes the sender from notifications and reports delivery counts in an acknowledgment. Recipient lookup must enforce current receive authorization, or use a documented membership-revocation mechanism; matching a stored scope alone is insufficient if permissions can change. Adapt sender inclusion and acknowledgment shape to the requested contract.
