# Pages and Cursors

[Documentation](../README.md#documentation) · [Example models](example.md)

`page()` and `iterate()` accept keyword-only `using`. See
[Database Configuration](database.md).

## Page

`page()` returns a materialized, read-only sequence. Iteration, indexing,
slicing, and `len(page)` require no further requests. Length counts fetched
records, not all query matches. `limit` must be a positive integer; see
[Arguments](errors.md#arguments).

Return at most `limit` records. A page may inspect one additional index entry
for continuation, without fetching or decoding that entry's record.

Primary traversal:

```python
page = User.page(limit=100)

while True:
    for user in page:
        print(user.key, user.name)

    if page.cursor is None:
        break

    page = User.page(limit=100, cursor=page.cursor)
```

Declared ordering:

```python
page = Assignment.page(
    user,
    Assignment.status == "open",
    order_by=Assignment.created_at,
    reverse=True,
    limit=20,
)

for assignment in page:
    print(assignment.target, assignment.priority)

if page.cursor is not None:
    next_page = Assignment.page(
        user,
        Assignment.status == "open",
        order_by=Assignment.created_at,
        reverse=True,
        limit=20,
        cursor=page.cursor,
    )
```

## Cursors

`page.cursor` is an opaque, serializable continuation token; `None` indicates
exhaustion. Omitting `cursor` or passing `None` starts at the beginning.

Continuation requires the same namespace, model, source, filters, ordering, and
direction. Equivalent predicates identify the same query regardless of object
identity or argument order. A cursor is invalid for a different query or
incompatible index definition. Malformed or incompatible cursors raise
`InvalidCursorError`.

Continuation is exclusive: after the page's last examined primary key for
nodes, target key within a source for edges, or ordering value and primary-key
tiebreaker for declared indexes. The lookahead entry is not consumed. Ties have
deterministic order. Pagination uses neither offsets nor full scans.

Pages read live data without a snapshot. Inserts and deletes may change
subsequent pages. New keys before the cursor are excluded from later primary
traversal; keys after it may appear. Changes to ordering or filtering fields
may skip or repeat records.

A cursor indicates that more index entries were observed, not that the next
page is guaranteed to contain records. Concurrent deletion can produce short
or empty pages with a cursor. Continuation advances even when a page is empty.

## Iterate

`iterate()` yields records and follows page cursors automatically.

```python
for user in User.iterate(batch_size=100):
    print(user.key, user.name)
```

Arguments follow `page()` for source, predicates, `order_by`, `reverse`,
`using`, and optional starting `cursor`. `batch_size` replaces `limit`: a
positive integer controlling records fetched per request, not total results.

```python
for assignment in Assignment.iterate(
    user,
    Assignment.status == "open",
    order_by=Assignment.created_at,
    reverse=True,
    batch_size=100,
):
    print(assignment.target, assignment.priority)
```

`batch_size` is checked when `iterate()` is called, without storage access.
The first request for an item fetches and materializes a page. Another page is
fetched only after the current page is consumed and another item is requested.

With at least three users and no concurrent changes:

```python
users = User.iterate(batch_size=2)  # No storage access.
first = next(users)                # Fetch and construct two users.
second = next(users)               # Use the buffered page.
third = next(users)                # Fetch the next page.
```

The iterator retains at most one page of records internally and releases its
references to previous pages. It does not prefetch in the background. Stopping
consumption makes no further requests, although the current page may already
be buffered.

Iteration follows cursors until `None`, including across empty pages. An empty
query yields nothing.

`iterate()` does not count or materialize the full result set. Ordering,
filtering, and live-data behavior match `page()`. Use `page()` for explicit
page boundaries and continuation tokens.

## Materialization

Successful reads have these record-decoding and model-construction bounds:

| Operation | Records Decoded and Constructed |
| --- | --- |
| `get()` | One matching record, or none if absent. |
| `exists()`, `count()` | None. |
| `first()`, `last()` | At most one returned record. |
| `page()` | Only returned records, each once. |
| Edge Reads | Edge records only; no endpoint nodes. |
| `refresh()` | One replacement instance, then update the original. |

Index decoding is separate from record decoding. Reading keys, ordering values,
or index membership does not load a model. Constructions performed by
application callbacks or constructor overrides are outside these bounds.

Unsupported queries raise rather than loading records for Python-side
filtering. No particular number of backend commands or network requests is
guaranteed.

## Read Failures

`page()` returns after all selected records are decoded and constructed.
A fetching, decoding, or construction failure raises from the call. It does
not return a partial page or silently skip malformed records.

Returned pages remain usable without a connection. Iteration, indexing,
slicing, and length are local operations, including after the selected database
is closed.

For `iterate()`, a page failure raises from the `next()` call that needs that
page. Earlier pages remain delivered; no records from the failing page are
yielded. The iterator terminates without retrying, skipping the failure, or
resuming automatically.

Records removed concurrently may shorten a page. This does not turn
serialization or construction failures into missing records.
