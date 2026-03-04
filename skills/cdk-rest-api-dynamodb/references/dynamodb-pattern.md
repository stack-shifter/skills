# DynamoDB Pattern

Use this reference when you need a reusable DynamoDB table and access-pattern baseline.

## Goal

Create DynamoDB-backed endpoint infrastructure with predictable access patterns.

## Default Table Shape

- partition key: `PK`
- sort key: `SK`
- optional GSIs for alternate lookup patterns
- table name derived from service name and stage

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
