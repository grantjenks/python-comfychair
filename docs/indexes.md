# Ordered Indexes

[Documentation](../README.md#documentation) · [Example models](example.md)

Queries accept keyword-only `using` to select a database. See
[Database Configuration](database.md).

## Ordering

Declare indexes with field descriptors. Duplicate values are supported.
Equality predicates constrain leading fields; the next field supplies traversal
order and optional range bounds.

| Declaration | Query |
| --- | --- |
| `Ordering(created_at)` | `Task.page(order_by=Task.created_at, limit=20)` |
| `Ordering(priority)` on an edge | `Assignment.page(user, order_by=Assignment.priority, reverse=True, limit=20)` |
| `Ordering(status, created_at)` on an edge | `Assignment.page(user, Assignment.status == "open", order_by=Assignment.created_at, reverse=True, limit=20)` |

Edge indexes are scoped to the source automatically. Omit the source from
`Ordering` declarations.

## Automatic Primary Traversal

Nodes have a primary index on `key`; edges have one on `(source, target)`.
Writes and deletes maintain these indexes with the record. No declaration is
required.

| Model | Scope | Ascending order |
| --- | --- | --- |
| Node | All records of the model and namespace | Primary key |
| Edge | Outgoing edges of the model for one source | Target key |

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

`page()`, `iterate()`, `first()`, and `last()` default to primary traversal.
`order_by=User.key` or `order_by=Assignment.target` selects it explicitly.
`reverse=True` reverses traversal. Other fields require a declared `Ordering`
selected by `order_by`; model-level default ordering is not supported.

Key type determines order: `Key` and UUID use unsigned 128-bit numeric order;
strings use Unicode code-point order. Key order is deterministic, not
necessarily chronological. See [generated keys](models.md#generated-keys).

Primary indexes include every record in scope. `User.count()` counts all users
of that model and namespace; `Assignment.count(user)` counts all its outgoing
edges for that source. Counts do not load model instances.

Primary and declared indexes use the same [pagination rules](pagination.md).

## Predicates and Ranges

`page()`, `iterate()`, `first()`, `last()`, and `count()` accept positional
predicates. Multiple predicates mean AND; argument order is irrelevant.

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

| Expression | Meaning |
| --- | --- |
| `field == value` | Exact match |
| `field > value` | Exclusive lower bound |
| `field >= value` | Inclusive lower bound |
| `field < value` | Exclusive upper bound |
| `field <= value` | Inclusive upper bound |

Queries must fit a primary index or declared ordering. Equality predicates
constrain leading fields. Range predicates constrain the next field, also
selected by `order_by` for traversal. Lower and upper bounds are independently
optional and may be inclusive or exclusive. `order_by` accepts a field
descriptor; `reverse=True` selects descending order.

Unsupported queries raise `UnsupportedQueryError`. Scans, Python-side filtering, index
intersections, `!=`, OR, field-to-field comparisons, and arithmetic expressions
are not supported.

Converting a predicate to `bool` raises `TypeError`. Pass predicates
separately; Python `and`, `or`, and chained comparisons such as
`start <= Assignment.created_at < end` are invalid. Comparisons of instance
values, such as `reservation.starts_at < reservation.ends_at`, retain ordinary
Python behavior.

## Contradictory Filters

Contradictory predicates on a supported index produce an empty result:

```python
page = Assignment.page(
    user,
    Assignment.priority >= 10,
    Assignment.priority < 5,
    order_by=Assignment.priority,
)

assert len(page) == 0
assert page.cursor is None
```

`iterate()` yields nothing and `count()` returns zero. `first()` and `last()`
raise `KeyError` or return the supplied `default`. Index support and operand
validity are checked before treating a query as empty.

## First and Last

Return the first or last record in the requested order, including direction and
primary-key tiebreaker. Access the corresponding index end without fetching
intervening records.

Arguments follow `page()` for source, predicates, `order_by`, and `reverse`.
Neither limit nor cursor is accepted. Raise `KeyError` on an empty scope, or
return the explicit keyword-only `default`. See
[missing records](storage.md#missing-records-and-defaults).

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

## Count

Return index membership count without loading model instances. Return zero for
an empty scope.

```python
open_count = Assignment.count(user, Assignment.status == "open")
```

Source and predicates must identify a directly countable primary or declared
index scope, including supported ranges. Optional `order_by` selects the
traversal index:

```python
open_count = Assignment.count(
    user,
    Assignment.status == "open",
    order_by=Assignment.created_at,
)
```

Primary indexes include every record in scope. Secondary indexes omit entries
containing `None`. If eligible indexes have different memberships, select one
explicitly. Unsupported scopes raise `UnsupportedQueryError` without a fallback
scan. Counts
reflect current storage and may change before a later page request.

## Indexed Values

| Field type | Equality and ordering |
| --- | --- |
| `str` | Exact, case-sensitive equality; Unicode code-point order. No locale collation or normalization. |
| `bool` | Exact equality; `False` before `True`. |
| `int` | Exact equality and numeric order within the signed 64-bit range. |
| `float` | Finite IEEE 754 binary64 values in numeric order. Negative and positive zero compare equal. NaN and infinities are rejected. |
| `Key`, `UUID` | Exact equality and unsigned 128-bit numeric order. |
| `datetime` | Timezone-aware values normalized to UTC with microsecond precision; ordered by instant. Naive values are rejected. |

`None` serializes as JSON null and is omitted from secondary indexes. Any
`None` field excludes the whole composite index entry. Primary keys cannot be
`None`.

JSON arrays and objects have no built-in index encoding. Custom types require
an index adapter separate from their serialization adapter.

Unsupported index types and out-of-range indexed values raise before writing,
at context exit for queued writes. Application code supplies normalization,
such as lowercasing email addresses.

## Index Adapters

Register an adapter on the database that stores the index:

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Money:
    cents: int


database.register_type(
    Money,
    encode=lambda value: value.cents,
    decode=Money,
)
database.register_index(
    Money,
    encode=lambda value: value.cents,
    as_type=int,
)
```

`register_index(type, encode=callable, as_type=scalar_type)` maps a custom
declared type to a supported index scalar. Comfy Chair supplies the
storage-specific ordered encoding.

The same adapter converts stored values and query operands. Its output defines
equality and ordering. Conversion must be deterministic and return the declared
scalar type within its supported range. Changing the adapter requires
rebuilding affected indexes.

Registrations belong to one database and apply to the exact declared type.
Duplicate registrations raise `ValueError`. Configure adapters during startup.
Databases accessing the same indexes must use compatible adapters.

Class creation checks index structure. Adapter availability is checked against
the selected database when an operation needs it, before storage commands.
Commits check all required adapters and encodings before any writes.
