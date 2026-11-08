# Tests

Testing plan for the [Comfy Chair API](../README.md#documentation).
The pytest suite is not implemented yet.

Use real storage and public APIs. Keep portable conformance tests separate
from backend-specific observation and failure tests. Do not use mocks, canned
responses, patched Comfy Chair internals, or a substitute in-memory database.

## Observation

Use existing extension points; no additional public testing API is required.

| Work | Observer |
| --- | --- |
| Model Construction | Override `__init__`, call `super()`, and record constructions. |
| Default Factories | Record factory calls while producing normal values. |
| Validation | Override `validate()` and retain normal validation. |
| Serialization | Register real encoders and decoders that also record inputs. |
| JSON Encoding and Decoding | Wrap configured functions and delegate to their real implementations. |
| Storage Requests | Instrument a supplied real client or transport without replacing execution. |
| Mutation Attempts | Record callback inputs and invocation counts. |

Measure only the operation under test, excluding setup. Verify that observers
preserve results by comparing instrumented and uninstrumented execution.

## Positive Controls

Before asserting that work did not occur, demonstrate that the observer detects
that work. For example:

1. Save a record.
2. Fetch it and confirm that constructor and decoder observers detect the load.
3. Reset observations.
4. Call `exists()` and assert that neither observer records a load.

Use equivalent controls for factories, validation, storage requests, and retry
attempts.

## Storage Scans

Construction and decoding counts establish Python-side work, not the absence
of storage scans. Backend-specific checks observe actual commands, requested
ranges, and server-side execution where scripts are involved.

Run the same small query against increasingly large datasets and check that
unrelated records are not fetched. Avoid asserting exact command sequences:
implementations may combine requests or change index storage.

Timing and transferred data volume are supporting evidence, not proof.
Document gaps where backend visibility cannot establish whether a scan
occurred.

## Concurrency and Failures

Use separate database objects connected to the same real storage. Coordinate
concurrent work with barriers or events rather than sleeps.

Cover competing primary-key creation, stale ETags, whole-batch condition
failure, repeated writes and deletes, successful conflict retries, retry
exhaustion, and callback-raised conflicts. Include connection failures and
failures in decoders, constructors, validators, and callbacks.

A deterministic mutation conflict can be produced by updating the stored
record through a second database object during the first callback invocation.
The helper's conditional write must conflict and retry using the new value.

For an unconfirmed commit, check the error, unresolved results, and absence of
automatic replay. Inspect storage independently; do not assume that the write
failed.

## Conformance Coverage

Map each documented guarantee to a test or an explicit coverage gap. The
following areas organize that mapping; unchecked items are planned work.

- [ ] [Database Configuration](../docs/database.md): defaults, selection,
  binding, ownership, lazy connections, closing, contexts, and concurrency.
- [ ] [Models and Fields](../docs/models.md): declarations, defaults, factories,
  keys, namespaces, construction, endpoint typing, and mutable identity.
- [ ] [Store and Retrieve](../docs/storage.md): reads, missing-record defaults,
  refresh, get-or-create, write modes, ETags, deletion, and index maintenance.
- [ ] [Ordered Indexes](../docs/indexes.md): primary traversal, ordering,
  predicates, ranges, contradictory filters, scalar encodings, adapters, and
  unsupported queries.
- [ ] [Pages and Cursors](../docs/pagination.md): continuation, lookahead,
  materialization, concurrent changes, buffering, and read failure timing.
- [ ] [Commits](../docs/commits.md): initial-state conditions, execution order,
  capture timing, instance state, result handles, nesting restrictions, routing,
  and callable retries.
- [ ] [Mutations](../docs/mutations.md): independent nested values, custom copy
  behavior, unsaved edits, callback order, conflicts, exhaustion, and failures.
- [ ] [Relationships and Exclusive Ownership](../docs/relationships.md):
  directed edges, ownership claims, competing creation, swaps, and no cascades
  or automatic reverse traversal.
- [ ] [Choices and Validation](../docs/validation.md): required values, choices,
  overrides, validation timing, and class-definition errors.
- [ ] [Serialization](../docs/serialization.md): JSON functions, built-in and
  custom adapters, registration scope, inspection, and stored schema changes.
- [ ] [Errors and Arguments](../docs/errors.md): exception boundaries, causes,
  Python argument protocols, and unconfirmed storage outcomes.
- [ ] [Example](../docs/example.md): documented model declarations and workflows.

Run applicable coverage across supported Python versions and JSON engines.
Test UUID7 generation where available, UUID4 fallback where unavailable, and
reading both key forms.

Use isolated test data and reliable cleanup. Restore changed process-wide
configuration. Tests must run independently and in any order.
