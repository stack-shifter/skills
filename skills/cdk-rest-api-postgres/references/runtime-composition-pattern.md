# Runtime Composition Pattern

Use this reference when the repository needs a reusable runtime composition layer with `src/app.ts` and `src/data/context.ts`.

Read these files first when they exist:

- `src/app.ts`
- `src/data/context.ts`
- `src/data/db/client.ts`
- `src/data/repositories/`

## Goal

Create one module-scope composition root that initializes long-lived clients, repository context, and shared services once per Lambda runtime.

## Portable Shape

- `src/app.ts` exports singleton SDK clients, service instances, mapper instances, and `dbContext`
- `src/data/context.ts` aggregates repositories behind a single `DatabaseContext`
- controllers import from `src/app.ts`
- handlers stay declarative and do not construct dependencies directly

## Baseline `src/app.ts`

```ts
import { SESv2Client } from "@aws-sdk/client-sesv2";
import { S3Client } from "@aws-sdk/client-s3";
import { db } from "./data/db/client";
import { DatabaseContext } from "./data/context";
import { EmailService } from "./services/messaging/email.service";
import { StorageService } from "./services/storage/storage.service";

export const emailClient = new SESv2Client({ maxAttempts: 2 });
export const s3Client = new S3Client({});

export const dbContext = new DatabaseContext(db);
export const storageService = new StorageService(s3Client);
export const emailService = new EmailService(emailClient);
```

## Baseline `DatabaseContext`

```ts
import { Db } from "./db/client";
import { ClientRepository } from "./repositories/client.repository";
import { ProjectRepository } from "./repositories/project.repository";

export class DatabaseContext {
    readonly clients: ClientRepository;
    readonly projects: ProjectRepository;

    constructor(db: Db) {
        this.clients = new ClientRepository(db);
        this.projects = new ProjectRepository(db);
    }
}
```

## Guidance

- Reuse one shared `db` client; do not open ad hoc connections in handlers.
- Add a repository to `DatabaseContext` when more than one controller or service may need it.
- Prefer direct imports from `src/app.ts` over introducing a heavy DI container unless the repo already uses one.
- Keep this module limited to wiring. Do not put request logic in it.
- When the repository already has a composition root, extend it instead of adding a second one.
