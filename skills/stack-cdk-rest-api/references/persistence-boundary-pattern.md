# Persistence Boundary Pattern

Use this reference when the API needs persistence access and the repository needs a coherent boundary between business logic and storage logic.

## Goal

Keep controllers and services unaware of datastore mechanics by putting persistence access behind repositories and aggregating repositories behind one shared context when needed.

## Rules

- Controllers and services should ask for domain operations, not construct datastore requests.
- Repositories own query construction, key construction, transactions, and persistence-specific error handling.
- When multiple repositories are needed together, expose them through one aggregate context such as `DatabaseContext`, `DataContext`, or `Context`.
- Handlers should depend on controllers or services, not on datastore clients directly.

## Portable Shape

```text
src/
├── app.ts
├── controllers/
├── data/
│   ├── context.ts
│   └── repositories/
└── services/
```

## Baseline Context

```ts
import { ProjectRepository } from "./repositories/project.repository";
import { UserRepository } from "./repositories/user.repository";

export class Context {
    readonly projects: ProjectRepository;
    readonly users: UserRepository;

    constructor(db: RepositoryDependencies) {
        this.projects = new ProjectRepository(db);
        this.users = new UserRepository(db);
    }
}
```

## Baseline Composition

```ts
import { Context } from "./data/context";

const db = createRepositoryDependencies();

export const context = new Context(db);
export const projectService = new ProjectService(context.projects);
```

## Guidance

- Extend an existing context if the repository already has one.
- If no context exists, introduce the smallest reusable version rather than wiring repositories ad hoc in each handler.
- Repository methods should be named after domain operations or access patterns, not after raw persistence APIs.
- Keep mapping between persistence objects and domain objects inside repositories or dedicated mappers, not in controllers.
- Do not let controllers or services construct datastore-specific commands, key expressions, or raw persistence requests.
