# Modeling Workflow

Use this reference when the user needs a DynamoDB schema, table review, or early design guidance.

## Goal

Design DynamoDB from the application workload outward. Start with the questions the application must answer, then derive keys, indexes, and item collections from those questions.

## Workflow

### 1. Understand the application

Capture:

- entities
- relationships
- request paths or screens
- expected scale
- latency expectations
- analytics needs

An ERD can help, but it is not enough by itself.

### 2. Enumerate all access patterns

List every important operation explicitly.

Good examples:

- get user by username
- list orders for a user by creation time
- find account by email
- revoke expired session

Weak examples:

- manage users
- search data

For each access pattern, capture:

- operation
- caller
- parameters
- result shape
- item count
- sort order
- consistency needs

### 3. Choose the table strategy

Use `single-table` when:

- multiple entities belong in the same query paths
- item collections reduce round trips
- access patterns are known and performance matters

Use `multi-table` when:

- aggregates are mostly independent
- query flexibility is more important than the last bit of performance
- the product is still changing rapidly

Use another datastore, or narrow DynamoDB's role, when:

- ad hoc querying is primary
- analytics is primary
- relational joins are primary

### 4. Model the primary key first

Define the table around the primary access paths before reaching for GSIs.

Think in terms of item collections:

- parent with children
- entity with timeline
- entity with lookup record
- entity with uniqueness record

Aim to satisfy each important access pattern with as few requests as possible.

### 5. Add secondary indexes only for real needs

Use GSIs for alternate access paths that cannot be handled well with the base table.

Prefer:

- dedicated index attributes such as `GSI1PK` and `GSI1SK`
- sparse indexes when only a subset of items belongs in the index

Avoid adding indexes without a named access pattern.

### 6. Validate the model

Review for:

- hot partitions
- uneven partition key distribution
- unbounded collections
- heavy write concentration
- partition key values that could become celebrity or tenant hot spots
- migration complexity
- analytics pain
- large items or document-style blobs that approach DynamoDB limits

Be explicit about partition-key quality. A good partition key spreads reads and writes across many values instead of pushing activity onto a small number of hot keys.

If these risks are material, revise before implementation.
