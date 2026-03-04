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
