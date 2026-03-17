# Implementation Guidance

Use this reference when translating a DynamoDB model into application code.

## Boundary rule

Implement DynamoDB mapping at the data-access boundary.

Prefer a repository pattern so DynamoDB concerns stay isolated from the domain and business layer.

Inside the application core:

- use application objects
- use domain-oriented names
- avoid leaking raw DynamoDB attribute structures
- avoid constructing partition keys, sort keys, or DynamoDB expressions

At the boundary:

- expose repository methods that represent domain reads and writes
- construct the final key strings at the repository call site
- map items to domain objects
- perform type conversion
- attach indexing attributes

Typical shape:

- service/business layer calls repository interfaces
- repository implementation owns DynamoDB key design and request construction
- mappers translate between DynamoDB items and domain entities

Prefer repository code where the stored key values are obvious in place:

```ts
Item: {
  PK: `WORKSPACE#${workspaceId}`,
  SK: `PROJECT#NAME#${projectNameLower}#ID#${projectId}`,
  GSI1PK: `CATEGORY#WORKSPACE#${workspaceId}#${categoryId}`,
  GSI1SK: `PROJECT#${projectId}`,
}
```

Avoid hiding these values behind a shared `keys.ts`, `Keys` object, or helper whose main job is string interpolation. In this style, a little duplication is acceptable because it keeps reads and writes easier to follow.

## Attribute separation

Keep indexing attributes separate from application attributes.

Example indexing attributes:

- `PK`
- `SK`
- `GSI1PK`
- `GSI1SK`

Example application attributes:

- `Username`
- `Email`
- `CreatedAt`

Do not rely on the fact that an application value is encoded inside a key template. Keep the real attribute too.

## Index attribute naming

Prefer dedicated attributes per index such as:

- `GSI1PK`
- `GSI1SK`
- `GSI2PK`
- `GSI2SK`

Do not reuse one attribute across several indexes just to save storage. It makes the model harder to evolve.

## Type attribute

Include a `Type` attribute on each item unless the repository already has an equivalent convention.

This helps with:

- debugging
- migrations and ETL scripts
- table exploration
- analytics export and re-normalization

## Tooling guidance

- Prefer explicit repository or data-access methods over a generic ORM.
- Small debug scripts can make DynamoDB access patterns easier to inspect than the console.
- Keep code close to the modeled access patterns so a reader can see which query each method supports.
- Prefer explicit string literals in `Item`, `Key`, and `ExpressionAttributeValues` blocks over indirection through centralized key-builder modules.

## Repository pattern preference

Repository methods should hide DynamoDB mechanics from the rest of the application.

Prefer:

- `getUserByEmail(email)`
- `listTenantInvoices(tenantId, cursor)`
- `saveOrder(order)`

Avoid pushing these concerns into services or controllers:

- `PK` and `SK` construction
- `Query` versus `GetItem` decisions
- condition expression assembly
- GSI selection

The business layer should know what it wants to load or save, not how DynamoDB stores it.

Within the repository layer, do not over-abstract key construction. Small local variables are fine when they improve readability, but the final PK/SK/GSI shapes should remain visible near the DynamoDB request that uses them.

## Concurrency and versioning

When correctness depends on protecting updates from races:

- use condition expressions for optimistic concurrency on single-item updates
- use transactions for multi-item atomic changes
- use explicit version attributes when the repository needs compare-and-swap style updates

Do not assume last write wins is acceptable unless the user says so.

If the design will run across DynamoDB global tables, call out that cross-Region conflict resolution behaves differently from simple optimistic locking expectations.
