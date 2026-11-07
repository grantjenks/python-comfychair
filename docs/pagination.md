# Pages and cursors

[README](../README.md) · [Documentation](../README.md#documentation)

Examples use the models defined in the [introductory example](example.md).

Iterate all users in primary-key order:

```python
page = User.page(limit=100)

while True:
    for user in page:
        print(user.key, user.name)

    if page.cursor is None:
        break

    page = User.page(limit=100, cursor=page.cursor)
```

The same pagination API applies to declared orderings:

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

`Page` is a materialized, read-only sequence. Iteration, indexing, slicing, and
`len(page)` operate on fetched records without further requests. Its length is
the number of records in this page, not the total number of matches.

`page.cursor` is an opaque, serializable continuation token, or `None` when the
query is exhausted. Pass it back with the same namespace, model, source, filters,
ordering, and direction. Equivalent predicates identify the same query even
when reconstructed as new Python objects or supplied in a different argument
order. Omitting the cursor starts at the beginning. A cursor cannot be reused
for a different query or an incompatible index definition.

Primary traversal continues exclusively after the last primary key for nodes,
or the last target key within the source for edges. Declared orderings continue
after the last ordering value and primary-key tiebreaker. Equal timestamps or
priorities therefore have a deterministic order. Pagination requires no offsets
or full scans.

Pages read live data, not a snapshot. Inserts and deletes may change later pages.
In primary traversal, a new key before the cursor will not appear later in that
traversal; one after it may appear. Changing an ordering or filtering field
between requests can cause a record to be skipped or seen again.

## Automatic iteration

Use `iterate()` to yield individual records and follow page cursors automatically:

```python
for user in User.iterate(batch_size=100):
    print(user.key, user.name)
```

It accepts the same source, predicates, `order_by`, `reverse`, and optional
starting `cursor` as `page()`. The `batch_size` parameter replaces `limit`
and must be a positive integer. It controls records fetched per request, not
the total number yielded.

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

`iterate()` returns a lazy iterator. The first fetch happens when iteration
begins; subsequent pages are fetched as needed until the cursor is exhausted.
An empty query yields no records. Stopping iteration makes no further requests,
though records from the current page may already have been fetched. The iterator
does not compute a total count or materialize the full result set.

Iteration uses the same ordering, filtering, and live-data semantics as
`page()`; it does not provide a snapshot. Use `page()` directly when the caller
needs page boundaries or continuation cursors.
