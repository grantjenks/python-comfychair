# Coverage

The Markdown files specify expected behavior. They have not run against a
Comfy Chair implementation. Parsing and compilation check the examples' shape,
not their behavior.

## Important Guarantees

| Guarantee | Representative Examples |
| --- | --- |
| Stale writes cannot replace a newer version. | [Write Modes and ETags](write-modes.md), [Mutations](mutations.md). |
| Commits capture final values and ETags at exit. | [Change a Queued Object](commits.md#change-a-queued-object), [Capture the ETag at Exit](commits.md#capture-the-etag-at-exit). |
| Conditions gate the whole batch using state immediately before execution. | [Check Conditions at Execution](commit-conditions.md#check-conditions-at-execution), [Reject the Entire Batch on Conflict](commit-conditions.md#reject-the-entire-batch-on-conflict). |
| Results and local ETags follow operation order after confirmation. | [Commit Conditions and Instance State](commit-conditions.md). |
| Updates and deletes maintain index membership; rejected writes preserve it. | [Move an Index Entry on Update](indexes.md#move-an-index-entry-on-update), [Preserve Indexes on Conflict](indexes.md#preserve-indexes-on-conflict), [Deletion](deletion.md). |
| Routing honors explicit selection, associations, bindings, and the default. | [Database Selection](database.md), including conflicting destinations and equal URLs. |
| Creation returns one winner and preserves the winner's values. | [Competing Requests](storage.md#create-from-competing-requests), [Ownership Claims](relationships.md#compete-from-two-connections). |
| Cursors continue through ties without consuming lookahead. | [Pages and Cursors](pagination.md), [Iteration](iteration.md). |
| Reads load only the promised models, without loading edge endpoints. | Constructor observers in [Iteration](iteration.md), adapter observers in [Decoding](decoding.md). |
| Failed mutation attempts do not leak working-copy edits. | [Mutations](mutations.md), [Working Copies](working-copies.md). |
| Retry helpers retry conflicts, not arbitrary failures. | [Retrying Commits](retries.md), [Mutations](mutations.md). Transport uncertainty also needs the checks below. |
| Serialization and indexing use separate, configurable conversions. | [JSON Serialization](serialization.md), [JSON Engines](json-engines.md), [Adapters](adapters.md), [Indexed Values](index-values.md). |

## Documentation Areas

This is a navigation map, not a section-by-section coverage score.

| Source | Specifications |
| --- | --- |
| [Models](../docs/models.md) | [Models](models.md), [Identity](identity.md), [Decoding](decoding.md). |
| [Storage](../docs/storage.md) | [Storage](storage.md), [Write Modes](write-modes.md), [Deletion](deletion.md). |
| [Indexes](../docs/indexes.md) | [Queries](indexes.md), [Indexed Values](index-values.md). |
| [Pagination](../docs/pagination.md) | [Pages](pagination.md), [Iteration](iteration.md). |
| [Commits](../docs/commits.md) | [Commits](commits.md), [Conditions](commit-conditions.md), [Retries](retries.md). |
| [Mutations](../docs/mutations.md) | [Mutations](mutations.md), [Working Copies](working-copies.md). |
| [Relationships](../docs/relationships.md) | [Relationships](relationships.md). |
| [Database](../docs/database.md) | [Selection](database.md), [Connections](connections.md). |
| [Serialization](../docs/serialization.md) | [Serialization](serialization.md), [Engines](json-engines.md), [Adapters](adapters.md), [Decoding](decoding.md). |
| [Validation](../docs/validation.md) | [Validation](validation.md). |
| [Errors](../docs/errors.md) | [Errors](errors.md), [Failure Guarantees](guarantees.md). |
| [Example](../docs/example.md) | [Example](example.md). |

## Remaining Verification Work

The [testing plan](README.md) and [guarantees](guarantees.md) retain the following
requirements. They need real infrastructure or controlled observation beyond
these transcripts:

- Lost write confirmations: unresolved handles, unchanged local state, and no
  automatic replay, even when storage may already contain the write.
- Controlled concurrency schedules and separate processes. Starting two threads
  together does not force every relevant interleaving.
- Connection ownership, TLS, timeouts, pool behavior, and lazy I/O. A database
  that reopens after close cannot prove absence of I/O merely by being closed.
- Storage scans and requested ranges. Constructor and decoder counts establish
  model loading, not how much work the server performed.
- Iterator request and memory lifetime, including progress through empty pages
  caused by concurrent deletion.
- Supported backend, Python, and JSON-engine environments, including Python
  versions with and without UUID7. Missing dependencies are not passing results.
- Cleanup and restoration of process configuration after a failed example.
  Successful transcripts clean up their own isolated data.

## Contract Boundaries

Assert promised results and exception types. Leave JSON envelope details,
whitespace, member order, concrete descriptor classes, cursor representation,
and command sequences open where the docs leave them open. Test Python protocols
with useful examples, without excluding values that those protocols accept.

Rules without a direct transcript remain requirements. Review expected results
as contract changes; do not regenerate them merely to match an implementation.
