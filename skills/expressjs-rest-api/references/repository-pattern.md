# Repository Pattern (DynamoDB)

Use this reference when you need a reusable DynamoDB repository pattern for an Express API.

For Postgres projects, use `references/postgres-pattern.md` instead.

## Goal

Create a typed DynamoDB repository class that implements a shared repository contract and wraps infrastructure failures as domain or database errors.

## Repository Interface

```ts
// src/data/repository.interface.ts
import { CursorPagination } from './pagination';

export interface Repository<T> {
  findById(id: string): Promise<T | null>;
  findAll(query?: { cursor?: string; limit?: number }): Promise<CursorPagination<T>>;
  create(entity: T): Promise<T>;
  update(id: string, entity: Partial<T>): Promise<T>;
  delete(id: string): Promise<void>;
}
```

## Baseline Example

```ts
// src/data/item.repository.ts
import { DynamoDBDocument } from '@aws-sdk/lib-dynamodb';
import { Repository } from './repository.interface';
import { CursorPagination, decodeCursor, encodeCursor } from './pagination';
import { Item } from '../models/item.model';
import { loggerService } from '../dependencies/project.deps';
import { DatabaseError } from '../utilities/errors';

export class ItemRepository implements Repository<Item> {
  private dbContext: DynamoDBDocument;

  constructor(dbContext: DynamoDBDocument) {
    this.dbContext = dbContext;
  }

  async findById(id: string): Promise<Item | null> {
    try {
      const response = await this.dbContext.get({
        TableName: process.env.DYNAMODB_TABLE,
        Key: { PK: `ITEM#${id}`, SK: `METADATA#${id}` },
      });

      if (!response.Item) return null;

      return {
        id: response.Item['PK'].split('#')[1],
        name: response.Item['Name'],
        description: response.Item['Description'],
      };
    } catch (error: any) {
      loggerService.error('Error retrieving item', error);
      throw new DatabaseError('Error retrieving item');
    }
  }

  async findAll(query?: { cursor?: string; limit?: number }): Promise<CursorPagination<Item>> {
    try {
      const { Items, LastEvaluatedKey } = await this.dbContext.scan({
        TableName: process.env.DYNAMODB_TABLE,
        Limit: query?.limit || 10,
        ExclusiveStartKey: query?.cursor
          ? (decodeCursor(query.cursor).key as any)
          : undefined,
      });

      const result: CursorPagination<Item> = {
        limit: query?.limit || 10,
        hasNextPage: !!LastEvaluatedKey,
        items: (Items || []).map((item) => ({
          id: item['PK'].split('#')[1],
          name: item['Name'],
          description: item['Description'],
        })),
      };

      if (LastEvaluatedKey) {
        result.cursor = encodeCursor(LastEvaluatedKey);
      }

      return result;
    } catch (error: any) {
      loggerService.error('Error listing items', error);
      throw new DatabaseError('Error listing items');
    }
  }

  async create(entity: Item): Promise<Item> {
    try {
      await this.dbContext.put({
        TableName: process.env.DYNAMODB_TABLE,
        Item: {
          PK: `ITEM#${entity.id}`,
          SK: `METADATA#${entity.id}`,
          Name: entity.name,
          Description: entity.description,
        },
      });
      return entity;
    } catch (error: any) {
      loggerService.error('Error creating item', error);
      throw new DatabaseError('Error creating item');
    }
  }

  async update(id: string, entity: Partial<Item>): Promise<Item> {
    try {
      const updates: string[] = [];
      const names: Record<string, string> = {};
      const values: Record<string, unknown> = {};

      if (entity.name !== undefined) {
        updates.push('#name = :name');
        names['#name'] = 'Name';
        values[':name'] = entity.name;
      }

      if (entity.description !== undefined) {
        updates.push('#desc = :desc');
        names['#desc'] = 'Description';
        values[':desc'] = entity.description;
      }

      if (updates.length === 0) {
        const current = await this.findById(id);
        if (!current) throw new DatabaseError('Item not found');
        return current;
      }

      const response = await this.dbContext.update({
        TableName: process.env.DYNAMODB_TABLE,
        Key: { PK: `ITEM#${id}`, SK: `METADATA#${id}` },
        UpdateExpression: `SET ${updates.join(', ')}`,
        ExpressionAttributeNames: names,
        ExpressionAttributeValues: values,
        ConditionExpression: 'attribute_exists(PK) AND attribute_exists(SK)',
        ReturnValues: 'ALL_NEW',
      });

      if (!response.Attributes) {
        throw new DatabaseError('Item not found');
      }

      return {
        id: response.Attributes['PK'].split('#')[1],
        name: response.Attributes['Name'],
        description: response.Attributes['Description'],
      };
    } catch (error: any) {
      if (error.name === 'ConditionalCheckFailedException') {
        throw new DatabaseError('Item not found');
      }
      loggerService.error('Error updating item', error);
      throw new DatabaseError('Error updating item');
    }
  }

  async delete(id: string): Promise<void> {
    try {
      await this.dbContext.delete({
        TableName: process.env.DYNAMODB_TABLE,
        Key: { PK: `ITEM#${id}`, SK: `METADATA#${id}` },
        ConditionExpression: 'attribute_exists(PK) AND attribute_exists(SK)',
      });
    } catch (error: any) {
      if (error.name === 'ConditionalCheckFailedException') {
        throw new DatabaseError('Item not found');
      }
      loggerService.error('Error deleting item', error);
      throw new DatabaseError('Error deleting item');
    }
  }
}
```

## Dependency Wiring Pattern

```ts
// src/dependencies/db.deps.ts
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocument } from '@aws-sdk/lib-dynamodb';
import { ItemRepository } from '../data/item.repository';

const isLocalDynamoDB = !!process.env.DYNAMODB_ENDPOINT;

export const dynamoDBClient = DynamoDBDocument.from(
  new DynamoDBClient({
    region: process.env.AWS_REGION,
    ...(isLocalDynamoDB
      ? {
          credentials: { accessKeyId: 'dummy', secretAccessKey: 'dummy' },
          endpoint: process.env.DYNAMODB_ENDPOINT,
        }
      : {}),
  })
);

export const itemRepository = new ItemRepository(dynamoDBClient);
```

## Guidance

- Keys use `ENTITY#id` prefix pattern: `PK: 'ITEM#<id>'`, `SK: 'METADATA#<id>'`
- If the repository already has a repository contract or dependency composition pattern, adapt this shape to it instead of introducing a second one.
- If it does not, generate one reusable repository and composition pattern rather than calling the AWS SDK directly from controllers.
- Always catch DynamoDB SDK errors and rethrow as `DatabaseError` — controllers do the HTTP mapping
- Use `ExpressionAttributeNames` to avoid DynamoDB reserved word conflicts in update expressions
- In production, supply no credentials — the SDK picks up the IAM role from the compute environment
- `DYNAMODB_ENDPOINT` in `.env.development` enables a local DynamoDB container via Docker Compose
- Prefer `Query` over `Scan` when the repository has a known access pattern; use `Scan` only as a generic fallback
