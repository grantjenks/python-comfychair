# Executable Specifications

Small examples specify the public API. Each Markdown page is also a doctest
file. Expected results are requirements, not results observed from an
implementation. Comfy Chair is not implemented yet.

## Start Here

[Example](example.md) shows models, storage, relationships, and queries together.
The pages below isolate individual behaviors in smaller examples.

## Models and Storage

- [Models and Fields](models.md)
- [Keys and Namespaces](identity.md)
- [Store and Retrieve](storage.md)
- [Write Modes and ETags](write-modes.md)
- [Deletion](deletion.md)
- [Relationships and Ownership](relationships.md)
- [Choices and Validation](validation.md)
- [Errors and Arguments](errors.md)

## Queries and Traversal

- [Ordered Queries](indexes.md)
- [Indexed Values and Adapters](index-values.md)
- [Pages and Cursors](pagination.md)
- [Iteration and Observable Work](iteration.md)

## Commits and Mutations

- [Commits](commits.md)
- [Commit Conditions and Instance State](commit-conditions.md)
- [Retrying Commits](retries.md)
- [Mutations](mutations.md)
- [Mutation Working Copies](working-copies.md)

## Connections and Serialization

- [Database Selection](database.md)
- [Connection Lifetime](connections.md)
- [JSON Serialization](serialization.md)
- [JSON Engines](json-engines.md)
- [Custom Types and Schema Changes](adapters.md)
- [Decoding and Loading](decoding.md)

## Observation and Coverage

- [Concurrency and Failure Guarantees](guarantees.md)
- [Coverage](coverage.md)

## Read and Run

Examples in a page execute in order in one namespace. Pages supply their own
imports, model definitions, and data. Individual sections may reuse the page's
setup; copy that setup when running a section separately. Different pages do
not depend on each other. Markdown's indented code blocks preserve interpreter
prompts and expected output without a custom parser.

Once an implementation is available, install it and pytest, start a dedicated
Redis primary, and run a page with `python -m doctest -v tests/storage.md`.
`pytest.raises` checks exception types without fixing incidental message text.
No examples use mocks, patched internals, or an in-memory storage substitute.

Examples use localhost Redis database 0; routing examples also use database 1.
Generated model namespaces isolate runs. Successful pages remove their data
and close connections they create. A failed run may leave its isolated data;
the future runner must provide reliable final cleanup and restore process-wide
configuration on failure. Do not run against application storage.

Pages requiring a JSON engine say so. Run those pages in environments with that
engine installed. Do not silently skip requirements because a backend or
dependency is unavailable.

## Render

Render the same Markdown as HTML. Keep each input beside its expected output.
A future Copy Python control should extract doctest input, preserve indentation,
and omit prompts and expected output. Copy Transcript should preserve both.
No renderer, custom parser, test harness, or implementation is introduced here.

## Scope

Examples protect documented behavior while leaving implementation choices open.
Use ordinary Python protocols, representative inputs, and meaningful failure
cases. Avoid fixing incidental representations, exact messages, or internal
classes. Add an example when it explains a useful behavior or catches a plausible
regression, rather than to enumerate every argument combination.

Passing a transcript demonstrates its particular observations. Brief rules
cover behavior requiring concurrency, transport failures, storage inspection,
or other environments. Those rules remain requirements; they are not silently
treated as passing tests. [Coverage](coverage.md) records these distinctions.

The existing [testing plan](README.md) still applies: positive controls for
observers, public APIs, real storage, separate backend-specific checks, and
explicit coverage gaps. Run applicable examples across supported Python
versions, JSON engines, and backends. UUID generation needs environments both
with and without standard-library UUID7.
