# Runtime Composition Pattern

Use this reference when the repository needs a reusable runtime composition layer with `src/app.ts`, but backed by DynamoDB instead of Drizzle.

Read these files first when they exist:

- `src/app.ts`
- `src/data/context.ts`
- `src/data/`
- `src/services/`

## Goal

Create one module-scope composition root that initializes AWS clients, repository context, and shared services once per Lambda runtime.

## Portable Shape

- `src/app.ts` exports singleton SDK clients and service instances
- `src/data/context.ts` aggregates DynamoDB repositories behind a shared `DataContext`
- controllers import from `src/app.ts`
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
