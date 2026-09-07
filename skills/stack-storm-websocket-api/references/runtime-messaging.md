# Runtime Composition and Messaging

Reuse module-scope clients and your familiar repository context. The repository and permission service are application modules; they are not Storm exports.

## Context and app.ts

```ts
// src/data/context.ts
import type { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb';
import { ConnectionRepository } from './repositories/connection.repository';

export class Context {
    readonly connections: ConnectionRepository;

    constructor(client: DynamoDBDocumentClient, tableName: string) {
        this.connections = new ConnectionRepository(client, tableName);
    }
}
```

```ts
// src/app.ts
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb';
import { ApiGatewayManagementApi } from '@aws-sdk/client-apigatewaymanagementapi';
import { Context } from './data/context';
import { MessagingService } from './services/messaging.service';
export { updatePermissions } from './services/update-permissions.service';

const tableName = process.env.CONNECTION_TABLE;
if (!tableName) throw new Error('CONNECTION_TABLE is required');

const documentClient = DynamoDBDocumentClient.from(new DynamoDBClient({}));
export const context = new Context(documentClient, tableName);
const managementClient = new ApiGatewayManagementApi({
    endpoint: process.env.MANAGEMENT_API_ENDPOINT,
});
export const messagingService = new MessagingService(managementClient, context.connections);
```

Only the sending Lambda needs a configured management endpoint and permission to use it. The service checks the endpoint before sending. Do not run endpoint validation at module import for connect/disconnect routes that share the module but do not send. Keep request state out of singletons. Reuse existing dependency modules instead of adding a second composition root.

## Messaging Service

```ts
// src/services/messaging.service.ts
import { ApiGatewayManagementApi, GoneException } from '@aws-sdk/client-apigatewaymanagementapi';
import type { Connection, ConnectionRepository } from '../data/repositories/connection.repository';

export class MessagingService {
    constructor(
        private readonly client: ApiGatewayManagementApi,
        private readonly connections: ConnectionRepository,
    ) {}

    async sendUpdate(recipients: Connection[], senderId: string, message: unknown) {
        if (!process.env.MANAGEMENT_API_ENDPOINT) {
            throw new Error('MANAGEMENT_API_ENDPOINT is required for sending');
        }
        let sent = 0;
        let failed = 0;
        let stale = 0;
        for (const recipient of recipients) {
            if (recipient.connectionId === senderId) continue;
            if (recipient.expiresAt <= Math.floor(Date.now() / 1000)) continue;
            try {
                await this.client.postToConnection({
                    ConnectionId: recipient.connectionId,
                    Data: Buffer.from(JSON.stringify(message)),
                });
                sent++;
            } catch (error) {
                if (error instanceof GoneException) {
                    stale++;
                    try {
                        await this.connections.remove(recipient.connectionId);
                    } catch {
                        failed++;
                    }
                } else {
                    failed++;
                }
            }
        }
        return { sent, failed, stale };
    }
}
```

The example continues after recipient failures. `sent` counts accepted callback requests, not client processing confirmations; `failed` counts callback or cleanup failures. Use the project's logger for unexpected failures and cleanup failures without logging credentials or message bodies. Do not delete connections on generic transport failures.

Use simple sequential sends for small fanout. For larger workloads, implement bounded concurrency or queued delivery only when required. Retry transient failures with a bound and account for possible duplicates; a broadcast is not an atomic operation. Stale cleanup after `GoneException` applies to established stored connections, not as a workaround for sending during the handshake.
