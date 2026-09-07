# Strategy Patterns

Use this reference when choosing a DynamoDB pattern for relationships, filtering, sorting, uniqueness, pagination, or expiry.

## One-to-many

Common options:

- duplicate data into the parent view when the child data is shallow and frequently needed
- use a composite primary key and `Query` when children naturally belong in the same item collection
- use a secondary index when the alternate parent-to-child access path is the important one
- use hierarchical sort keys when you need structured ordering inside one partition

Pick the strategy that makes the main query cheap and predictable.

Watch for direct lookup gaps. A sort key such as `PROJECT#<createdAt>#<projectId>` is great for ordered listing, but awkward if the application later needs lookup by `projectId` alone. In those cases, consider:

- carrying the full composite key in the caller
- adding a lookup record
- using an identifier strategy that preserves needed ordering
- using a dedicated alternate access path

## Many-to-many

Common options:

- shallow duplication
- adjacency list
- materialized graph
- multiple requests when the complexity of denormalization is not worth it

Do not force a graph-like workload into a simplistic model without checking traversal requirements.

## Filtering

Prefer these options in roughly this order:

1. partition key restriction
2. sort key restriction
3. composite sort key encoding
4. sparse secondary index
5. filter expression
6. client-side filtering

Treat filter expressions as a last-mile cleanup step, not the main selectivity mechanism.

## Sorting

Sorting is a key-design problem.

Use the sort key to encode:

- chronological order
- lexical order
- hierarchical order
- compound order such as `STATUS#CREATED_AT#ID`

Watch for:

- changing sort attributes
- descending order needs
- numeric ordering pitfalls
- zero-padding requirements

## Uniqueness

For uniqueness on secondary attributes, use dedicated uniqueness records plus conditional writes.

Typical pattern:

- write the main item
- write one or more uniqueness items
- use a transaction or conditional writes so duplicates fail cleanly

## Pagination

Design pagination into the access pattern from the start.

Remember:

- DynamoDB paginates natively
- the 1 MB page limit matters
- cursors should reflect the last evaluated key or an application-safe encoding of it

Do not promise offset-based pagination if the underlying pattern is keyset-based.

## Multi-tenant SaaS

For B2B SaaS systems, start by asking whether most reads and writes are scoped to one tenant.

If they are, a strong default is:

- partition key rooted on the tenant
- multiple item types colocated in the tenant item collection
- tenant-local queries served from the base table
- cross-tenant or global lookups pushed to GSIs

This often works well for:

- tenant dashboard reads
- tenant-owned users, projects, tasks, or subscriptions
- tenant-scoped listings and summaries

Be careful when a supposedly tenant-scoped model still needs frequent global queries such as:

- login by email across all tenants
- admin listings across all tenants
- global reporting views

Those patterns should be named explicitly and modeled as exceptional access paths rather than assumed to fall out of the base table design.

## Partition key distribution

Treat partition key design as both a modeling and scaling concern.

Prefer partition keys that:

- spread traffic across many values
- avoid concentrating heavy reads or writes on one tenant, one status bucket, or one timestamp bucket
- still preserve the item collections needed for the main access patterns

If a natural partition key will be hot, consider:

- write sharding
- read sharding
- bucketing by time or logical segment
- changing the retrieval flow so the hottest path is no longer bound to one key

Tenant partitioning is powerful, but check whether a single large tenant could become a hot key. If one tenant can dominate traffic, you may need tenant bucketing or a different partition strategy for the hottest entities.

## Overloaded GSIs

Overloading a GSI can be a good optimization once the design is stable.

Use it when:

- multiple access patterns can safely share one index
- the prefixes remain clear and non-overlapping
- the combined index still reads cleanly to future maintainers

Prefer separate GSIs first when clarity is more important than minimizing index count. Overload only when the shared design remains understandable.

## Large items

If an item can grow large, address that in the design rather than treating it as an implementation detail.

Common options:

- vertical partitioning across multiple related items in one collection
- compression for large text attributes when that tradeoff is acceptable
- storing large blobs in S3 and keeping metadata plus pointers in DynamoDB

Do not treat DynamoDB like a general document store for arbitrarily large payloads.

## TTL

Use TTL for expiry cleanup when the row naturally expires, such as sessions, temporary links, or ephemeral tokens.

But:

- TTL deletion is not immediate
- the application should still validate expiry for correctness-sensitive flows
- TTL is a cleanup mechanism, not a precise scheduler

## Time-series workloads

Time-series data is a notable special case.

For very large time-ordered datasets, consider:

- one table per period when operationally justified
- scaling down or archiving older periods
- keeping active writes isolated from colder historical data

Do not force all time-series data into one forever-growing table if period-based isolation materially simplifies scale or cost management.
