# Pages and Cursors

[Documentation](../README.md#documentation) · [Example models](example.md)

`page()` and `iterate()` accept keyword-only `using`. See
[Database Configuration](database.md).

## Page

`page()` returns a materialized, read-only sequence. Iteration, indexing,
slicing, and `len(page)` require no further requests. Length counts fetched
records, not all query matches.

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
exhaustion. Omitting `cursor` starts at the beginning.

Continuation requires the same namespace, model, source, filters, ordering, and
direction. Equivalent predicates identify the same query regardless of object
identity or argument order. A cursor is invalid for a different query or
incompatible index definition.

Continuation is exclusive: after the last primary key for nodes, the last
target key within a source for edges, or the last ordering value and
primary-key tiebreaker for declared indexes. Ties have deterministic order.
Pagination uses neither offsets nor full scans.

Pages read live data without a snapshot. Inserts and deletes may change
subsequent pages. New keys before the cursor are excluded from later primary
traversal; keys after it may appear. Changes to ordering or filtering fields
may skip or repeat records.

## Iterate

`iterate()` yields records and follows page cursors automatically.

```python
for user in User.iterate(batch_size=100):
    print(user.key, user.name)
```

Arguments follow `page()` for source, predicates, `order_by`, `reverse`,
`using`, and optional starting `cursor`. `batch_size` replaces `limit`: a positive integer
controlling records fetched per request, not total results.

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

Iteration is lazy. The first fetch occurs when iteration starts; subsequent
pages are fetched as needed until exhaustion. An empty query yields nothing.
Stopping iteration makes no further requests, although the current page may
already be buffered.

`iterate()` does not count or materialize the full result set. Ordering,
filtering, and live-data behavior match `page()`. Use `page()` for explicit
page boundaries and continuation tokens.
