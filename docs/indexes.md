# Ordered indexes

[README](../README.md) · [Documentation](../README.md#documentation)

Examples use the models defined in the [introductory example](example.md).

`Ordering` declarations take field descriptors, not field-name strings.
`Ordering` supports traversal with duplicate values. Equality predicates match
leading fields in the declaration; the following field supplies
the traversal order and may have range bounds.

| Declaration | Example access |
| --- | --- |
| `Ordering(created_at)` | `Task.page(order_by=Task.created_at, limit=20)` |
| `Ordering(priority)` on an edge | `Assignment.page(user, order_by=Assignment.priority, reverse=True, limit=20)` |
| `Ordering(status, created_at)` on an edge | `Assignment.page(user, Assignment.status == "open", order_by=Assignment.created_at, reverse=True, limit=20)` |

Edge indexes are scoped to their source automatically. An
`Ordering(created_at)` indexes outgoing edges for each source.
The source does not need to appear in the declaration.

## Automatic primary traversal

Every node model has an ordered primary index on `key`; every edge model has
one on `(source, target)`. Record writes and deletes maintain these indexes
alongside the records. No `Ordering` declaration is required.

| Model | Scope | Ascending order |
| --- | --- | --- |
| Node | All records of the model and namespace | Primary key |
| Edge | All outgoing edges of the model for one source | Target key |

```python
page = User.page(limit=100)
first = User.first()
last = User.last()
total = User.count()

outgoing = Assignment.page(user, limit=20)
first_edge = Assignment.first(user, default=None)
last_edge = Assignment.last(user, default=None)
edge_count = Assignment.count(user)
```

For `page()`, `iterate()`, `first()`, and `last()`, omitting `order_by` selects the
primary traversal. Use `order_by=User.key` or
`order_by=Assignment.target` to select it explicitly, and `reverse=True`
for descending traversal. `first()` and `last()` use the normal missing-record
policy, raising `KeyError` on an empty scope unless a default is supplied.

The key type determines ordering: `Key` and UUID values use unsigned 128-bit
numeric order, and strings use Unicode code-point order. Key order is
deterministic but does not guarantee creation order; see
[generated keys](models.md#generated-keys).

A node's primary index includes every stored record, so `User.count()` counts
all users of that model and namespace. A source-only edge count includes every
outgoing edge for that model and source. Counts use index membership without
loading model instances.

Primary traversal uses the same `Page` and cursor rules as declared orderings.
Filters on other fields must fit a declared `Ordering`, selected with
`order_by` for traversal.

## Predicates and ranges

Pass filters as positional predicates. Multiple predicates mean AND:

```python
start = datetime(2026, 1, 1, tzinfo=timezone.utc)
end = datetime(2026, 2, 1, tzinfo=timezone.utc)

page = Assignment.page(
    user,
    Assignment.status == "open",
    Assignment.created_at >= start,
    Assignment.created_at < end,
    order_by=Assignment.created_at,
    reverse=True,
    limit=20,
)
```

Supported comparisons are:

| Expression | Meaning |
| --- | --- |
| `field == value` | Exact match |
| `field > value` | Exclusive lower bound |
| `field >= value` | Inclusive lower bound |
| `field < value` | Exclusive upper bound |
| `field <= value` | Inclusive upper bound |

`page()`, `iterate()`, `first()`, `last()`, and `count()` accept positional
predicates.
`order_by` takes a field descriptor, such as `Task.created_at`. Use
`reverse=True` for descending traversal.

Every query must fit an automatic primary index or a declared ordering.
Equality predicates constrain its leading
fields; range predicates constrain the next field, which is also the traversal
field selected by `order_by`. A range can have a lower bound, an upper bound,
or both, with inclusive or exclusive endpoints. Predicate argument order does
not matter.

Unsupported queries raise an error. There are no implicit scans, Python-side
filtering, or index intersections. Inequality (`!=`), OR, field-to-field
comparisons, and arithmetic expressions are not supported.

Predicate objects raise `TypeError` when converted to a boolean. Supply separate
positional predicates instead of using Python `and`, `or`, or a chained
comparison such as `start <= Assignment.created_at < end`. Comparisons between
ordinary instance values, such as `reservation.starts_at < reservation.ends_at`,
remain normal Python comparisons.

## First, last, and count

`first()` and `last()` accept the same source, positional predicates, `order_by`,
and `reverse` arguments as `page()`, but no limit or cursor. They follow the
[missing-record policy](storage.md#missing-records-and-defaults):
return a record, raise `KeyError` if absent, or return an explicit `default`:

```python
oldest = Assignment.first(
    user,
    Assignment.status == "open",
    order_by=Assignment.created_at,
)
newest = Assignment.last(
    user,
    Assignment.status == "open",
    order_by=Assignment.created_at,
    default=None,
)
```

First and last refer to the requested ordering, including `reverse` and the
primary-key tiebreaker. They access the corresponding end of the index without
fetching the intervening records. Without `order_by`, nodes use primary-key
order and edges use target-key order within the source. Other traversal requires
an explicit ordering; there is no configurable model-level default ordering.

`count()` returns the number of entries in a supported index scope without
loading model instances:

```python
open_count = Assignment.count(user, Assignment.status == "open")
```

The source and predicates must identify a scope that an automatic primary
index or declared ordering can count directly, including supported range
bounds. An optional `order_by` selects the same index used for ordered
traversal when needed:

```python
open_count = Assignment.count(
    user,
    Assignment.status == "open",
    order_by=Assignment.created_at,
)
```

An empty scope returns zero. Counts follow index membership. Primary indexes
include every record in their scope; secondary orderings omit entries
containing `None`. If eligible indexes would count different memberships, the query must select an index explicitly. Unsupported scopes raise
an error; there is no fallback scan.
Counts describe current storage and may change before a subsequent page request.

## Indexed values

The built-in indexed field types have deliberately limited rules:

| Field type | Equality and ordering |
| --- | --- |
| `str` | Exact, case-sensitive equality; Unicode code-point order, without locale collation or normalization. |
| `bool` | Exact equality; `False` sorts before `True`. |
| `int` | Exact equality and numeric order within the signed 64-bit range. |
| `float` | Finite IEEE 754 binary64 values in numeric order; negative and positive zero compare equal. NaN and infinities are rejected. |
| `Key`, `UUID` | Exact equality and unsigned 128-bit numeric order. |
| `datetime` | Timezone-aware values normalized to UTC, with microsecond precision, ordered by instant. Naive datetimes are rejected. |

`None` is stored as JSON null but omitted from indexes. If any field in a
composite index is `None`, the record has no entry in that index. Primary-key
fields cannot be `None`.

JSON arrays and objects can be stored but have no built-in index encoding.
A custom serialization adapter does not automatically make its type indexable.
Custom types need a separate, deterministic, order-preserving encoding for
use in `Ordering`. Unsupported index types and out-of-range indexed values
raise an error before writing, at context exit for queued writes.
Application-specific normalization, such as lowercasing email addresses, belongs
in application code.
