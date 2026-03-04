# Runtime Composition Pattern

Use this reference when the repository needs a reusable runtime composition layer for shared DynamoDB clients, repositories, and services.

## Goal

Create one module-scope composition root that initializes AWS clients, repository context, and shared services once per Lambda runtime.

## Portable Shape

- a composition module such as `src/app.ts` exports singleton SDK clients and service instances
- a context module such as `src/data/context.ts` aggregates DynamoDB repositories behind a shared `DataContext`
- controllers import from the composition module
- handlers stay thin and do not construct repositories or SDK clients directly

## Baseline `src/app.ts`

```ts
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient } from "@aws-sdk/lib-dynamodb";
import { S3Client } from "@aws-sdk/client-s3";
import { DataContext } from "./data/context";
import { StorageService } from "./services/storage/storage.service";

const dynamoClient = new DynamoDBClient({});
export const documentClient = DynamoDBDocumentClient.from(dynamoClient);
export const s3Client = new S3Client({});

export const dataContext = new DataContext(documentClient);
export const storageService = new StorageService(s3Client);
```

## Baseline `DataContext`

```ts
import { DynamoDBDocumentClient } from "@aws-sdk/lib-dynamodb";
import { UserRepository } from "./repositories/user.repository";
import { ProjectRepository } from "./repositories/project.repository";

export class DataContext {
    readonly users: UserRepository;
    readonly projects: ProjectRepository;

    constructor(client: DynamoDBDocumentClient) {
        this.users = new UserRepository(client);
        this.projects = new ProjectRepository(client);
    }
}
```

## Guidance

- Reuse one `DynamoDBDocumentClient` per runtime.
- Centralize table names and shared environment lookups in one place when several repositories use them.
- Aggregate repositories behind a context object when controllers need more than one repository.
- Keep this layer focused on wiring only; business logic belongs in controllers and services.
- If the repository already has a container or dependency module, extend it instead of adding a parallel one.
