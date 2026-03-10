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

## Soft-Delete Pattern

If the repository uses soft-delete with a scheduled hard-delete cleanup job, follow this pattern for any new entity that can be deleted via the API.

**Three-step `remove` method:**

1. **Soft-delete the primary row** — `UpdateCommand` sets `isDeleted: true`, `deletedAt`, and `updatedAt`.
2. **Immediately hard-delete all secondary rows** — lookup pointers, directory entries, and uniqueness locks are deleted right away (not by the cleanup job).
3. **Write a cleanup marker** — `PutCommand` stores a `buildCleanupItem()` record that the scheduled job will process after the retention window, hard-deleting the primary row and the marker itself.

Wire the cleanup Lambda using the `ScheduleLambda` construct — see `schedule-pattern.md`.

```ts
import {
    buildCleanupItem,
    buildRetentionDueAt,
    SoftDeleteEntityType,
} from '../db/utils/soft-delete';
import { DeleteCommand, PutCommand, UpdateCommand } from '@aws-sdk/lib-dynamodb';

async remove(id: string): Promise<void> {
    const existing = await this.findById(id);
    if (!existing) {
        return; // already deleted — idempotent
    }

    const deletedAt = new Date().toISOString();
    const retentionHours = Number(process.env.SOFT_DELETE_RETENTION_HOURS ?? '168');

    // Step 1: soft-delete the primary row.
    await this.db.send(
        new UpdateCommand({
            TableName: this.tableName,
            Key: {
                PK: entityPk(existing.organizationId),
                SK: entitySk(id),
            },
            UpdateExpression: 'SET isDeleted = :isDeleted, deletedAt = :deletedAt, updatedAt = :updatedAt',
            ExpressionAttributeValues: {
                ':isDeleted': true,
                ':deletedAt': deletedAt,
                ':updatedAt': deletedAt,
            },
        }),
    );

    // Step 2: immediately hard-delete secondary rows (lookup, directory, uniqueness lock, etc.).
    await this.db.send(
        new DeleteCommand({
            TableName: this.tableName,
            Key: { PK: lookupPk(id), SK: lookupPk(id) },
        }),
    );
    // ... delete other auxiliary rows as needed

    // Step 3: write cleanup marker — cleanup job hard-deletes the primary row + this marker.
    await this.db.send(
        new PutCommand({
            TableName: this.tableName,
            Item: buildCleanupItem({
                entityType: 'CLIENT' satisfies SoftDeleteEntityType,
                targetPk: entityPk(existing.organizationId),
                targetSk: entitySk(id),
                scheduledAt: buildRetentionDueAt(deletedAt, retentionHours),
                reason: 'SOFT_DELETE',
            }),
        }),
    );
}
```

**Cascade roots** — for entities that must propagate deletion to children, call each child repository's `remove()` for all active descendants before soft-deleting the root. The `SOFT_DELETE_CASCADE_MATRIX` in `src/data/db/utils/soft-delete.ts` declares the traversal:

```ts
// ORG    → ["PROJECT", "CLIENT"]
// PROJECT → ["UPDATE", "SHARE_LINK", "PROJECT_CLIENT_EDGE"]
import { SOFT_DELETE_CASCADE_MATRIX } from '../db/utils/soft-delete';

async remove(id: string): Promise<void> {
    const activeChildIds = await this.childRepository.findActiveIdsByParent(id);
    for (const childId of activeChildIds) {
        await this.childRepository.remove(childId);
    }
    // Then soft-delete this entity (UpdateCommand → cleanup marker PutCommand).
}
```

**Filtering deleted items** — all read methods must exclude soft-deleted rows:

```ts
// Option A: inline filter after query (preferred when rows are fetched into memory)
rows.filter((item) => !item.isDeleted)

// Option B: DynamoDB FilterExpression
FilterExpression: 'attribute_not_exists(isDeleted) OR isDeleted = :isDeleted',
ExpressionAttributeValues: { ':isDeleted': false }
```

**Share-link variant** — for entities with an expiry date, a second cleanup reason `"EXPIRES"` is also supported. Pass `reason: 'EXPIRES'` and set `scheduledAt` to the expiry timestamp rather than the retention due-at timestamp.

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
        isDeleted: false,
        deletedAt: null,
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
