# Schema Reference Pattern

Use this reference whenever a new entity, GSI, access pattern, item family, or item shape is introduced or changed. The goal is to keep `docs/schema-reference.md` (or the equivalent path in the target repository) accurate and up to date as a developer-facing source of truth for the DynamoDB table schema.

## Purpose of the File

`docs/schema-reference.md` is a human-readable companion to the live table. It captures:

- which GSIs exist and what queries drive them
- every named access pattern tied to an API route or internal operation
- the PK/SK shape for every item family
- the full attribute set for each row type, with representative values
- lookup and uniqueness item relationships
- operational notes such as pagination, seeding, and soft-delete behavior

## When to Update

Update `docs/schema-reference.md` whenever you:

- add a new entity type or item family (new PK/SK pattern)
- add or change a GSI
- add a new access pattern tied to a route or internal operation
- add, remove, or rename attributes on an existing item shape
- add a lookup, uniqueness lock, or cleanup marker variant
- change cascade delete behavior or soft-delete retention logic

Do not defer schema reference updates to a follow-up. Write the diff in the same task that introduces the data model change.

## File Structure

The file uses these top-level sections in order:

```
# DynamoDB Single-Table Reference

## GSIs
## Core Access Patterns
## Item Families
## Item Shapes
## Lookup and Uniqueness Items
## Operational Notes
```

Only include sections that have content. Do not add empty sections.

### GSIs

List every GSI that is actively used by at least one runtime repository. For each GSI include:

- index name
- partition key attribute name
- sort key attribute name (or "not used" if the SK is absent from current queries)
- which repository method or feature uses it

Example:

```markdown
- `GSI1`
    - Partition key: `GSI1PK`
    - Sort key: not used by current runtime queries
    - Used by: category dependency check (`CategoryRepository.hasProjects`)
```

Omit GSIs that are defined in CDK but never queried in code.

### Core Access Patterns

One bullet per route or internal operation. Each entry must include:

- the HTTP method and path, or a short internal operation label
- which index (PK/SK or a named GSI)
- the key condition used in the query or get
- default sort direction
- any notable filter (e.g., `isInternal = true` exclusion, expiry check)

Example:

```markdown
- `GET /organizations/{organizationId}/projects/{projectId}/updates`
    - Access: `PK/SK`
    - Query: `PK = PROJECT#{projectId}` with `begins_with(SK, UPDATE#)`
    - Sort: DESC (default); by `SK = UPDATE#{createdAt}#{updateId}` (newest first)
```

### Item Families

One bullet per logical record type. State the PK and SK template using `#` separators and `{variable}` placeholders. Group related items together (e.g., primary and directory entries for the same entity).

Example:

```markdown
- Organization primary: `PK = ORG#{organizationId}`, `SK = ORG#{organizationId}`
- Organization directory: `PK = DIRECTORY#ORG`, `SK = NAME#{orgNameLower}#ORG#{organizationId}`
```

### Item Shapes

One fenced JSON block per item family, under a `###` heading named after the item type. Use representative placeholder values: `<uuid>` for UUIDs, `<iso>` for ISO 8601 timestamps, `<sha256-hex>` for hashes. Mark optional attributes with a `?` suffix on the key name.

Example:

```markdown
### Organization primary

\`\`\`json
{
    "PK": "ORG#<uuid>",
    "SK": "ORG#<uuid>",
    "entityType": "ORG",
    "organizationId": "<uuid>",
    "organizationName": "Acme Corp",
    "organizationNameLower": "acme corp",
    "createdAt": "<iso>",
    "updatedAt": "<iso>",
    "isDeleted?": false,
    "deletedAt?": null
}
\`\`\`
```

### Lookup and Uniqueness Items

Describe auxiliary row categories in plain text:

- which entities have lookup rows and what indirection they provide
- which uniqueness locks exist and what logical value they enforce
- cascade and soft-delete relationships in brief (entity → dependents)

### Operational Notes

Short bullets covering runtime behavior that does not fit the above sections:

- pagination mechanics (e.g., opaque base64url cursor tokens)
- migration policy (e.g., no SQL migrations needed)
- seeding approach (e.g., `scripts/seed.ts`)

## Generating the File from Scratch

When no `docs/schema-reference.md` exists yet, create it by:

1. Reading every repository file under `src/data/repositories/` to collect PK/SK key expressions, GSI queries, and item write shapes.
2. Reading entity type and key prefix constants from `src/data/db/utils/conventions.ts` or equivalent.
3. Reading soft-delete helpers to understand cleanup marker shape.
4. Assembling the sections above in order and writing the file to `docs/schema-reference.md`.

The generated file should match exactly what the live code does — do not add speculative attributes or hypothetical access patterns.

## Updating an Existing File

When extending an existing file:

1. Add the new GSI under `## GSIs` if it does not already appear.
2. Add the new access pattern(s) under `## Core Access Patterns` in the same bullet style as existing entries.
3. Add the new item family line(s) under `## Item Families`.
4. Add the new `### <Entity> <type>` block(s) under `## Item Shapes`.
5. Update `## Lookup and Uniqueness Items` if the new entity introduces lookup or lock rows.
6. Update `## Operational Notes` only if the change affects pagination, seeding, or migration behavior.

Do not reorder or reformat sections that are not touched by the current change.

## Placement and Naming

Default path: `docs/schema-reference.md`.

If the target repository uses a different docs folder (e.g., `documentation/`, `wiki/`), place the file there and keep the name `schema-reference.md`. Note the chosen path in your response when it differs from the default.
