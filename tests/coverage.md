# Coverage

Source baseline: `8640b12f0ac1d9d2915f3749a97dbb158de74a7f`. Root README and
docs remain unchanged. This is a section-level reconciliation, not a claim
that all guarantees have been executed or proven.

## Source Sections

Every source section has a destination. Surrounding introductions, shared rules,
examples, and qualifications are included in the corresponding page or its
linked guarantee page.

| Source Section | Specification |
| --- | --- |
| [commits: Commit](../docs/commits.md#commit) | [commits.md](commits.md#queue-a-write) |
| [commits: Database Selection](../docs/commits.md#database-selection) | [database.md](database.md#route-a-whole-commit) |
| [commits: Result](../docs/commits.md#result) | [commits.md](commits.md#read-the-result) |
| [commits: Capture and Object State](../docs/commits.md#capture-and-object-state) | [commit-conditions.md](commit-conditions.md#keep-the-last-written-etag) |
| [commits: Conditions Apply to the Whole Batch](../docs/commits.md#conditions-apply-to-the-whole-batch) | [commit-conditions.md](commit-conditions.md#check-initial-absence-twice) |
| [commits: Retrying a Callable](../docs/commits.md#retrying-a-callable) | [retries.md](retries.md) |
| [commits: Limits](../docs/commits.md#limits) | [guarantees.md](guarantees.md#gate-a-whole-batch) |
| [database: Default Database](../docs/database.md#default-database) | [database.md](database.md#select-the-process-default) |
| [database: Database](../docs/database.md#database) | [connections.md](connections.md#construct-lazily) |
| [database: Connection Settings](../docs/database.md#connection-settings) | [connections.md](connections.md#configure-timeouts) |
| [database: Serialization Settings](../docs/database.md#serialization-settings) | [serialization.md](serialization.md#observe-primitive-conversion) |
| [database: Supplied Clients](../docs/database.md#supplied-clients) | [connections.md](connections.md#supply-a-client) |
| [database: Ownership](../docs/database.md#ownership) | [connections.md](connections.md#supply-a-client) |
| [database: Close](../docs/database.md#close) | [connections.md](connections.md#reopen-lazily) |
| [database: Context Managers](../docs/database.md#context-managers) | [connections.md](connections.md#use-a-lifetime-context) |
| [database: Bind](../docs/database.md#bind) | [database.md](database.md#bind-a-model) |
| [database: Using](../docs/database.md#using) | [database.md](database.md#routing-rules) |
| [database: Instance State](../docs/database.md#instance-state) | [database.md](database.md#remember-the-successful-destination) |
| [database: Edges](../docs/database.md#edges) | [database.md](database.md#route-edges-independently) |
| [database: Commit Routing](../docs/database.md#commit-routing) | [database.md](database.md#route-a-whole-commit) |
| [database: Concurrency](../docs/database.md#concurrency) | [connections.md](connections.md#share-connections-carefully) |
| [errors: Exceptions](../docs/errors.md#exceptions) | [errors.md](errors.md#exception-boundaries) |
| [errors: Arguments](../docs/errors.md#arguments) | [errors.md](errors.md#use-the-integer-protocol) |
| [errors: Storage Failures](../docs/errors.md#storage-failures) | [guarantees.md](guarantees.md#lose-confirmation) |
| [example: Models](../docs/example.md#models) | [example.md](example.md#declare-a-user) |
| [example: Writes](../docs/example.md#writes) | [example.md](example.md#save-together) |
| [example: Queries](../docs/example.md#queries) | [example.md](example.md#list-open-assignments-newest-first) |
| [indexes: Ordering](../docs/indexes.md#ordering) | [indexes.md](indexes.md#select-an-equality-prefix) |
| [indexes: Automatic Primary Traversal](../docs/indexes.md#automatic-primary-traversal) | [indexes.md](indexes.md#traverse-primary-keys-automatically) |
| [indexes: Predicates and Ranges](../docs/indexes.md#predicates-and-ranges) | [indexes.md](indexes.md#bound-the-next-field) |
| [indexes: Contradictory Filters](../docs/indexes.md#contradictory-filters) | [indexes.md](indexes.md#return-an-empty-contradictory-range) |
| [indexes: First and Last](../docs/indexes.md#first-and-last) | [indexes.md](indexes.md#read-either-end) |
| [indexes: Count](../docs/indexes.md#count) | [indexes.md](indexes.md#count-index-membership) |
| [indexes: Indexed Values](../docs/indexes.md#indexed-values) | [index-values.md](index-values.md#built-in-comparisons) |
| [indexes: Index Adapters](../docs/indexes.md#index-adapters) | [index-values.md](index-values.md#define-a-custom-comparison) |
| [models: Field](../docs/models.md#field) | [models.md](models.md#declare-fields) |
| [models: Construction and Loading](../docs/models.md#construction-and-loading) | [models.md](models.md#load-through-the-constructor) |
| [models: Node](../docs/models.md#node) | [identity.md](identity.md#inherit-the-key-default) |
| [models: Edge Endpoints](../docs/models.md#edge-endpoints) | [models.md](models.md#declare-typed-endpoints) |
| [models: Generated Keys](../docs/models.md#generated-keys) | [identity.md](identity.md#generate-a-key) |
| [models: Primary Keys](../docs/models.md#primary-keys) | [identity.md](identity.md#supply-a-string-key) |
| [models: Namespaces and Storage Identity](../docs/models.md#namespaces-and-storage-identity) | [identity.md](identity.md#separate-equal-model-names) |
| [mutations: Mutate](../docs/mutations.md#mutate) | [mutations.md](mutations.md) |
| [mutations: Working Copies](../docs/mutations.md#working-copies) | [working-copies.md](working-copies.md) |
| [pagination: Page](../docs/pagination.md#page) | [pagination.md](pagination.md#read-one-page) |
| [pagination: Cursors](../docs/pagination.md#cursors) | [pagination.md](pagination.md#continue-exclusively) |
| [pagination: Iterate](../docs/pagination.md#iterate) | [iteration.md](iteration.md#begin-lazily) |
| [pagination: Materialization](../docs/pagination.md#materialization) | [iteration.md](iteration.md#observation-bounds) |
| [pagination: Read Failures](../docs/pagination.md#read-failures) | [iteration.md](iteration.md#fail-a-whole-page) |
| [relationships: Email Ownership](../docs/relationships.md#email-ownership) | [relationships.md](relationships.md#claim-an-address) |
| [relationships: Concurrent Creation](../docs/relationships.md#concurrent-creation) | [relationships.md](relationships.md#compete-from-two-connections) |
| [relationships: Swapping Owners](../docs/relationships.md#swapping-owners) | [relationships.md](relationships.md#swap-two-owners) |
| [serialization: JSON Values](../docs/serialization.md#json-values) | [serialization.md](serialization.md#round-trip-declared-types) |
| [serialization: JSON Functions](../docs/serialization.md#json-functions) | [json-engines.md](json-engines.md) |
| [serialization: Type Adapters](../docs/serialization.md#type-adapters) | [adapters.md](adapters.md#register-a-conversion) |
| [serialization: JSON Inspection](../docs/serialization.md#json-inspection) | [serialization.md](serialization.md#inspect-current-values-without-saving) |
| [serialization: Stored Schema Changes](../docs/serialization.md#stored-schema-changes) | [adapters.md](adapters.md#add-a-field-with-a-default) |
| [serialization: Index Encodings](../docs/serialization.md#index-encodings) | [index-values.md](index-values.md#define-a-custom-comparison) |
| [storage: Missing Records and Defaults](../docs/storage.md#missing-records-and-defaults) | [storage.md](storage.md#raise-for-a-missing-record) |
| [storage: Exists](../docs/storage.md#exists) | [storage.md](storage.md#check-existence) |
| [storage: Refresh](../docs/storage.md#refresh) | [storage.md](storage.md#refresh-in-place) |
| [storage: Get or Create](../docs/storage.md#get-or-create) | [storage.md](storage.md#create-or-retrieve) |
| [storage: Write Modes](../docs/storage.md#write-modes) | [write-modes.md](write-modes.md) |
| [storage: ETags and Conditional Updates](../docs/storage.md#etags-and-conditional-updates) | [write-modes.md](write-modes.md#replace-with-a-matching-version) |
| [storage: Deletion](../docs/storage.md#deletion) | [deletion.md](deletion.md) |
| [validation: Choices](../docs/validation.md#choices) | [validation.md](validation.md#reject-an-invalid-choice) |
| [validation: Validate](../docs/validation.md#validate) | [validation.md](validation.md#validate-explicitly) |
| [validation: Model Definitions](../docs/validation.md#model-definitions) | [validation.md](validation.md#reject-an-invalid-constant-default) |

## Additional Examples

[JSON Engines](json-engines.md) exercises both configured optional engines.
[Iteration](iteration.md) observes normal constructors and a real constructor
failure. [Decoding](decoding.md) observes real type adapters, endpoint loading,
and decoding failures. [Working Copies](working-copies.md) observes custom deep-copy behavior.
[Relationships](relationships.md) includes concurrent ownership claims through
independent real connections. [Retries](retries.md) and [Mutations](mutations.md)
create deterministic version conflicts without mocking storage.

## Retained Testing Requirements

The original [testing plan](README.md) remains intact. Its observation, positive
control, storage scan, concurrency, failure, cleanup, and version/engine matrix
requirements are retained in [Specifications](specifications.md) and
[Guarantees](guarantees.md). A model-load count is not proof of no storage scan.

## Verification Boundaries

- All transcripts are proposed expected behavior. There is no Comfy Chair
  implementation to run them against yet.
- Parser and Python compilation checks validate source shape, not API behavior.
- Local transcripts cover specific schedules and data, not all concurrent cases.
- Real transport failures, TLS/pool ownership, backend scan inspection, process
  boundaries, and iterator memory/request lifetime need an environment harness.
- UUID4/UUID7 generation needs Python versions on both sides of availability.
- Optional engines need their dependencies installed; do not count unavailable
  environments as passing.
- Schema loading specifies failure for missing required fields/identity without
  choosing an exception subtype not fixed by the source docs.
- JSON representation checks observe real primitive conversion without inventing
  unspecified record-envelope names, whitespace, or member order.
- Generated namespaces isolate pages. Cleanup on failure and restoration of
  process defaults require the future runner; successful transcripts clean up.

## Maintenance

Keep expected results authored and review changes as contract changes. Do not
regenerate expectations merely to match an implementation. Maintain this map
when source documentation or executable specifications change.
