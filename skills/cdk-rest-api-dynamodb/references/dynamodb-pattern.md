# DynamoDB Pattern

Use this reference when you need a reusable DynamoDB table and access-pattern baseline.

## Goal

Create DynamoDB-backed endpoint infrastructure with predictable access patterns.

## Default Table Shape

- partition key: `PK`
- sort key: `SK`
- optional GSIs for alternate lookup patterns
- table name derived from service name and stage

## Strategy Choice

Default to single-table design for greenfield DynamoDB APIs unless the repository already has a clear multi-table boundary.

Choose single-table when:

- multiple entity types participate in shared query patterns
- you need hierarchical reads, timelines, or alternate access paths across entity types
- future flexibility matters more than table-by-table operational isolation

Choose multi-table when:

- aggregates are operationally independent
- access patterns are simple and table-local
- the repository already uses one-table-per-aggregate conventions

Do not mix single-table and multi-table casually inside the same feature area. Pick one strategy for the feature and keep repository APIs aligned with it.

## Baseline Example

```ts
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';

const usersTable = new dynamodb.TableV2(this, 'UsersTable', {
    tableName: `Users${props.deploymentStage}`,
    partitionKey: { name: 'PK', type: dynamodb.AttributeType.STRING },
    sortKey: { name: 'SK', type: dynamodb.AttributeType.STRING },
    globalSecondaryIndexes: [
        {
            indexName: 'GSI1',
            partitionKey: { name: 'GSI1PK', type: dynamodb.AttributeType.STRING },
            sortKey: { name: 'GSI1SK', type: dynamodb.AttributeType.STRING },
        },
    ],
    deletionProtection: props.deploymentStage === 'Prod',
});
```

## Endpoint Wiring Rules

- pass the table name to handlers via environment variables or a shared configuration helper
- grant `grantReadData` for read-only routes
- grant `grantReadWriteData` for create, update, and delete routes
- if handlers query GSIs, ensure IAM covers index resources when using custom policies

## Data Modeling Guidance

- If the repository already has a DynamoDB table construct or key convention, align with it.
- Otherwise use `PK` and `SK` for entity identity and hierarchy and `GSI1PK` and `GSI1SK` for the main alternate access path.
- Keep handlers thin; put key construction and query details into a repository or service layer.
- Prefer introducing a shared repository or key helper when multiple handlers use the same access pattern.
- For single-table designs, encode entity type and relationship intent into key prefixes.
- For multi-table designs, keep each table schema simple and avoid inventing single-table style prefixes without a real need.

## Entity Type Conventions

If the repository defines `EntityType` and `KeyPrefix` constants (e.g., in `src/data/db/utils/conventions.ts`), add new entity types and prefixes there rather than inventing ad-hoc string literals inside a repository:

```ts
export const EntityType = {
    USER: 'USER',
    ORDER: 'ORDER',
    // add new entity types here
} as const;

export const KeyPrefix = {
    USER: 'USER#',
    ORDER: 'ORDER#',
} as const;
```

Build PK/SK values inline in each repository using these constants for readability; do not create a shared key-builder factory.

## Hard-Delete Pattern

Use a single `TransactWriteCommand` to atomically hard-delete all item rows — the primary record, lookup pointer, directory entry, and uniqueness locks. There is no soft-delete marker and no cleanup job.

**`remove` method structure:**

1. Fetch the lookup pointer (or primary record) to retrieve current key values — the primary SK can embed a mutable name and must be read at delete time, not guessed.
2. Build all `Delete` operations for the transaction.
3. Send one `TransactWriteCommand`. Missing items do not cause failures when no condition expression is supplied on a `Delete`.

```ts
import { GetCommand, TransactWriteCommand } from '@aws-sdk/lib-dynamodb';

async remove(id: string): Promise<void> {
    // Fetch the lookup pointer to get the current primary SK (which may embed a name).
    const lookupResult = await this.db.send(
        new GetCommand({
            TableName: this.tableName,
            Key: { PK: `ENTITY#${id}`, SK: `ENTITY#${id}` },
        }),
    );
    if (!lookupResult.Item) {
        return; // already deleted — idempotent
    }

    const { entitySk, organizationId, nameLower } = lookupResult.Item;

    // Delete all item rows atomically.
    await this.db.send(
        new TransactWriteCommand({
            TransactItems: [
                { Delete: { TableName: this.tableName, Key: { PK: `ORG#${organizationId}`, SK: entitySk } } },
                { Delete: { TableName: this.tableName, Key: { PK: `ENTITY#${id}`, SK: `ENTITY#${id}` } } },
                { Delete: { TableName: this.tableName, Key: { PK: `UNIQ#ENTITY#NAME#${nameLower}`, SK: `UNIQ#ENTITY#NAME#${nameLower}` } } },
            ],
        }),
    );
}
```

**Cascade deletes** — for entities that own child records (e.g., a Project owns Updates, Share-links, and edge records), query each child collection and call the child repository's `remove()` before deleting the parent. Keep cascade orchestration in a service layer, not the repository.

**DynamoDB TTL for auto-expiring items** — for records with a known expiry date (such as share links), write a `ttl` field as a Unix epoch in seconds alongside the ISO string. DynamoDB auto-deletes expired rows without a cleanup job. Apply `ttl` to every item row (primary, lookup, token) in the same create transaction so all rows expire together:

```ts
ttl: Math.floor(expiresAt.getTime() / 1000), // DynamoDB TTL — auto-expires the row
expiresAt: expiresAt.toISOString(),           // ISO string for application logic
```

Enable TTL on the table by setting `timeToLiveAttribute: 'ttl'` in the CDK table definition.

## Transactional Writes

Use `TransactWriteCommand` when a create or update operation must write several items atomically — for example, the primary entity record, a directory entry for sorted list queries, and a uniqueness lock, all in one shot. If any item's condition fails, DynamoDB rolls back the entire batch.

Prepare all items before sending the transaction so the code reads top-to-bottom as a data declaration rather than nested AWS calls.

```ts
import { TransactWriteCommand } from '@aws-sdk/lib-dynamodb';

async create(input: IEntity): Promise<IEntity> {
    const id = randomUUID();
    const now = new Date().toISOString();
    const nameLower = normalize(input.name);

    // Primary record
    const primary = {
        PK: `ENTITY#${id}`,
        SK: `ENTITY#${id}`,
        entityType: EntityType.ENTITY,
        entityId: id,
        name: input.name,
        nameLower,
        createdAt: now,
        updatedAt: now,
    };

    // Directory entry — enables sorted list queries via GSI or SK range scan
    const directory = {
        PK: 'DIRECTORY#ENTITY',
        SK: `NAME#${nameLower}#ENTITY#${id}`,
        entityType: EntityType.ENTITY_DIRECTORY,
        entityId: id,
        name: input.name,
        nameLower,
        updatedAt: now,
    };

    // Uniqueness lock — prevents two records with the same normalized name
    const lock = {
        PK: `UNIQ#ENTITY#NAME#${nameLower}`,
        SK: `UNIQ#ENTITY#NAME#${nameLower}`,
        entityType: EntityType.UNIQUE_LOCK,
        entityId: id,
    };

    await this.db.send(
        new TransactWriteCommand({
            TransactItems: [
                {
                    Put: {
                        TableName: this.tableName,
                        Item: primary,
                        // Reject duplicates — fails the whole transaction if the key already exists
                        ConditionExpression: 'attribute_not_exists(PK) AND attribute_not_exists(SK)',
                    },
                },
                {
                    Put: {
                        TableName: this.tableName,
                        Item: directory,
                        ConditionExpression: 'attribute_not_exists(PK) AND attribute_not_exists(SK)',
                    },
                },
                {
                    Put: {
                        TableName: this.tableName,
                        Item: lock,
                        ConditionExpression: 'attribute_not_exists(PK) AND attribute_not_exists(SK)',
                    },
                },
            ],
        }),
    );

    return (await this.findById(id))!;
}
```

**Update with lock rotation** — when a mutable field (such as a name) is part of the uniqueness lock or directory key, an update must delete the old auxiliary rows and write new ones in the same transaction:

```ts
await this.db.send(
    new TransactWriteCommand({
        TransactItems: [
            { Put: { TableName: this.tableName, Item: updatedPrimary } },
            { Delete: { TableName: this.tableName, Key: { PK: oldLockPk, SK: oldLockPk } } },
            { Put: { TableName: this.tableName, Item: newLock, ConditionExpression: 'attribute_not_exists(PK) AND attribute_not_exists(SK)' } },
            { Delete: { TableName: this.tableName, Key: { PK: oldDirectoryPk, SK: oldDirectorySk } } },
            { Put: { TableName: this.tableName, Item: newDirectory } },
        ],
    }),
);
```

**Error handling** — `ConditionalCheckFailedException` and `TransactionCanceledException` signal a constraint violation. Map them in the controller using `RestResult.fromDatabaseError(error)` rather than inspecting the error name directly in the repository.
