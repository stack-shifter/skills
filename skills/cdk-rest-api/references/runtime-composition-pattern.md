# Runtime Composition Pattern

Use this reference when the repository needs a reusable runtime composition layer for shared clients, repository context, and services.

## Goal

Create one module-scope composition root that initializes long-lived clients, repository context, and shared services once per Lambda runtime.

## Portable Shape

- a composition module such as `src/app.ts` exports singleton SDK clients, service instances, mapper instances, and `repositoryContext`
- a context module such as `src/data/context.ts` aggregates repositories behind a single context object
- controllers import from `src/app.ts`
- handlers stay declarative and do not construct dependencies directly

## Baseline `src/app.ts`

```ts
import { SESv2Client } from "@aws-sdk/client-sesv2";
import { S3Client } from "@aws-sdk/client-s3";
import { RepositoryContext } from "./data/context";
import { EmailService } from "./services/messaging/email.service";
import { StorageService } from "./services/storage/storage.service";

export const emailClient = new SESv2Client({ maxAttempts: 2 });
export const s3Client = new S3Client({});

export const projectMapper = new ProjectMapper();
export const repositoryContext = new RepositoryContext(createRepositoryDependencies());
export const storageService = new StorageService(s3Client);
export const emailService = new EmailService(emailClient);
```

## Baseline Repository Context

```ts
import { ClientRepository } from "./repositories/client.repository";
import { ProjectRepository } from "./repositories/project.repository";

export class RepositoryContext {
    readonly clients: ClientRepository;
    readonly projects: ProjectRepository;

    constructor(deps: RepositoryDependencies) {
        this.clients = new ClientRepository(deps);
        this.projects = new ProjectRepository(deps);
    }
}
```

## Guidance

- Reuse long-lived runtime dependencies; do not construct repository dependencies ad hoc in handlers.
- Add a repository to the shared repository context when more than one controller or service may need it.
- Prefer one lightweight composition module over introducing a heavy DI container unless the repo already uses one.
- Keep this module limited to wiring. Do not put request logic in it.
- When the repository already has a composition root, extend it instead of adding a second one.
