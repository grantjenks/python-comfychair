# Comfy Chair

> “Cardinal Fang! Fetch…THE COMFY CHAIR!”
>
> Monty Python, [The Spanish Inquisition](https://python.mzonline.com/sketches/spanishinquisition/)

Typed nodes, edges, and indexes for Python, backed by Redis.

Comfy Chair stores records by key and relationships as records with their own
metadata. Type annotations define fields. Explicit indexes provide unique
lookups and ordered traversal. Namespaces keep independent collections of data
separate.

The interface centers on five concepts:

- **Node:** a record identified by a key, with a generated UUID by default.
- **Edge:** a directed relationship identified by its source and target keys.
- **Lookup:** a unique index for exact matches.
- **Ordering:** an index for ordered traversal, optionally within an equality filter.
- **Page:** a materialized sequence of results with a continuation cursor.

**Development status:** design stage. The API below specifies the intended
behavior; an implementation is not yet available. Examples assume a configured
Redis connection.

## Models and namespaces

Declare stored fields with Python type annotations. Constructors accept keyword
arguments, and defaults use dataclass-style declarations.

A model's optional `namespace` separates its records and indexes from those in
other namespaces. Omit it to use no namespace; there is no implicit namespace
prefix. Model names distinguish record types with or without a namespace. The
same primary key can exist in different models or namespaces without a collision.
Related models share a namespace when one is used; edge references include the
identity of their endpoint models.

For example, a small task application could define:

```python
from dataclasses import field
from datetime import datetime, timezone
from uuid import UUID, uuid4
from typing import Literal

from comfychair import Edge, Lookup, Node, Ordering, transaction


def now():
    return datetime.now(timezone.utc)


class User(Node):
    namespace = "example"

    key: UUID = field(default_factory=uuid4)
    name: str
    email: str

    indexes = (
        Lookup("email"),
    )


class Task(Node):
    namespace = "example"

    title: str
    created_at: datetime = field(default_factory=now)

    indexes = (
        Ordering("created_at"),
    )


class Assignment(Edge[User, Task]):
    namespace = "example"

    status: Literal["open", "closed"] = "open"
    priority: int = 0
    created_at: datetime = field(default_factory=now)

    indexes = (
        Ordering("created_at"),
        Ordering("priority"),
        Ordering("status", "created_at"),
    )
```

`Node` supplies a `key: UUID` field with a generated UUID by default. The
`User` declaration shows it explicitly; `Task` inherits it. Names and titles
are ordinary fields. Primary lookup by key is automatic.

`Edge[User, Task]` supplies typed `source` and `target` endpoints. Its default
key is `(source.key, target.key)`, scoped to the edge model and namespace. There
is one edge per endpoint pair for a given edge model. Different edge models can
connect the same pair independently.

Edges store endpoint references and their own fields, not copies of endpoint
records. Fetching an edge does not fetch either endpoint. Relationships are
directed; outgoing indexes do not imply an incoming index.

Namespaces organize storage. They are not authorization boundaries and do not
by themselves place Redis Cluster keys in the same hash slot.

Model names and namespaces are part of storage identity. Renaming a model or
changing its namespace changes where it reads and writes; existing data is not
renamed or migrated automatically. Changing index declarations likewise does
not rebuild existing indexes. Take care when changing these declarations for
models that already have stored data.

## Choices and validation

Use `Literal` annotations for a fixed set of choices, as in the `Assignment`
model's `status` field. Python enum types are also supported: fields hold enum
members in Python and serialize their underlying values. An enum is indexable
when its values share a supported index type; ordering follows those values,
not member declaration order.

Call `record.validate()` to check field types, required values, choices, and
supported index encodings without accessing Redis. It returns `None` on success
and raises `ValidationError` on invalid data. Validation does not silently
coerce values to other types.

Override `validate()` for rules involving multiple fields, and call
`super().validate()` to retain the built-in checks:

```python
from comfychair import ValidationError


class Reservation(Node):
    starts_at: datetime
    ends_at: datetime

    def validate(self):
        super().validate()
        if self.ends_at <= self.starts_at:
            raise ValidationError("ends_at must be later than starts_at")
```

`set()`, `add()`, and creation through `get_or_create()` validate before
writing. Transaction writes validate before being queued. Mutating an object's
fields alone does not trigger validation. Local validation does not establish
uniqueness, existence, or ETag matches; those checks happen atomically on Redis
during the operation.

Model definitions are checked at class creation. Invalid declarations raise
`TypeError`, including unknown index fields, multiple primary indexes,
unsupported indexed field types, incompatible constant defaults, and stored
fields that collide with reserved API names. Default factories are checked when
their values are produced, not invoked during class creation. These checks do
not inspect existing Redis data or migrate stored records.

## Serialization

Record values are serialized as JSON, with an automatically managed ETag stored
alongside the record's fields. Standard JSON values include strings, numbers,
booleans, nulls, arrays, and objects.

Serialization also supports fields whose Python types are not native JSON
types, including UUIDs and datetimes. These values use explicit JSON-compatible
encodings and are restored to their declared Python types when read. Custom
field types can supply encoding and decoding adapters. Unsupported values raise
a serialization error rather than silently converting to strings.

Serialization preserves field values; it does not recursively persist related
nodes. Index encodings are separate from record JSON: an ordered index must
preserve the field's comparison semantics rather than sort its JSON text.

The built-in indexed field types have deliberately limited rules:

| Field type | Lookup and ordering |
| --- | --- |
| `str` | Exact, case-sensitive lookup; Unicode code-point order, without locale collation or normalization. |
| `bool` | Exact lookup; `False` sorts before `True`. |
| `int` | Exact lookup and numeric order within the signed 64-bit range. |
| `float` | Finite IEEE 754 binary64 values in numeric order; negative and positive zero compare equal. NaN and infinities are rejected. |
| `UUID` | Exact lookup and unsigned 128-bit numeric order. |
| `datetime` | Timezone-aware values normalized to UTC, with microsecond precision, ordered by instant. Naive datetimes are rejected. |

`None` is stored as JSON null but omitted from indexes. If any field in a
composite index is `None`, the record has no entry in that index. Multiple
records may therefore have `None` in an otherwise unique field. Primary-key
fields cannot be `None`.

JSON arrays and objects can be stored but have no built-in index encoding.
A custom serialization adapter does not automatically make its type indexable.
Custom types need a separate, deterministic lookup encoding, and an
order-preserving encoding for use in `Ordering`. Unsupported index types and
out-of-range indexed values raise an error before the write is queued or sent.
Application-specific normalization, such as lowercasing email addresses, belongs
in application code.

## Store and retrieve

```python
user = User(name="Alice", email="alice@example.com")
task = Task(title="Write the introduction")

user.set()
task.set()

assignment = Assignment(source=user, target=task, priority=3)
assignment.set()

same_user = User.get(user.key)
by_email = User.get(email="alice@example.com")
same_assignment = Assignment.get(user, task)

task.title = "Write the README introduction"
task.set()
```

Constructing or changing a Python object performs no I/O. Persistence is explicit.
The methods below execute immediately. To queue a write in a transaction, call
its counterpart on the transaction object.

| Method | Behavior |
| --- | --- |
| `Model.get(...)` | Return the matching record, or `None` if absent. |
| `Model.exists(...)` | Return whether a primary key or unique lookup has a matching record. |
| `record.refresh()` | Replace fields and ETag in place with the stored version; return `None`, or raise `KeyError` if absent. |
| `Model.get_or_create(...)` | Return `(record, created)`, retrieving a match or inserting a new record. |
| `record.set()` | Insert or replace the record and update its indexes. |
| `record.set(mode=IF_NOT_MODIFIED)` | Replace the record only if its stored ETag matches the version held by this object; otherwise raise `ConflictError`. |
| `record.add()` | Insert only; return `True` if inserted, `False` if its primary key already exists. |
| `record.delete()` or `Model.delete(key)` | Remove the record and its index entries; return `True` if a record was deleted, `False` if absent. |

A conflicting secondary unique lookup raises `IntegrityError`. It never
replaces another record.

### Existence and refresh

`exists()` accepts the same primary-key and unique-lookup arguments as `get()`.
It returns a boolean without loading a model instance:

```python
has_user = User.exists(user.key)
has_email = User.exists(email="alice@example.com")
has_assignment = Assignment.exists(user.key, task.key)
```

`refresh()` explicitly reloads an existing object:

```python
task.refresh()
```

It replaces the object's stored fields and ETag, discarding unsaved edits.
If the stored record no longer exists, it raises `KeyError` and leaves the
object unchanged. Refreshing does not load related nodes. Like other reads,
existence checks and refreshes reflect current storage and do not reserve or
lock the record for a later write.

### Get or create

Use `get_or_create()` when a record should be retrieved if present and inserted
otherwise:

```python
user, created = User.get_or_create(
    email="alice@example.com",
    defaults={"name": "Alice"},
)
```

Lookup arguments must identify a primary key or a complete declared unique
`Lookup`. Primary-key arguments follow `get()`, including the two endpoints
for an edge. `defaults` supplies additional fields only when creating; normal
field defaults and factories supply the remaining values.

If a match exists, return it with `False` and leave its fields and ETag unchanged.
Otherwise, validate and insert the new record and return it with `True`.
The lookup and insertion are coordinated atomically: concurrent calls for the
same lookup cannot create duplicates. A conflict with a different unique index
raises `IntegrityError`. Defaults cannot override fields supplied as lookup arguments.

`get_or_create()` executes immediately and is not a queued transaction method.

### Write modes

`set()` accepts a mode controlling whether the write is allowed:

| Mode | Behavior |
| --- | --- |
| `ALWAYS` | Insert or replace, regardless of existence or ETag. The default. |
| `IF_NOT_MODIFIED` | Replace only if the record exists and its ETag matches the object's ETag. |
| `IF_MISSING` | Insert only if the primary key does not exist. |
| `IF_EXISTS` | Replace only if the primary key exists, regardless of its ETag. |

Modes are members of the `Mode` enum and are also exported as module-level
constants. `IF_NOT_MODIFIED` and `Mode.IF_NOT_MODIFIED` are the same value.

```python
from comfychair import ALWAYS, IF_NOT_MODIFIED, IF_MISSING, IF_EXISTS, Mode

new_task = Task(title="Write the installation guide")
new_task.set(mode=IF_MISSING)

new_task.title = "Revise the installation guide"
new_task.set(mode=IF_EXISTS)
```

A failed mode condition raises `ConflictError` without changing the record or
its indexes. Every mode still enforces validation and unique secondary indexes.
Unsupported modes raise `ValueError` before the operation is sent or queued.

`add()` remains the insert-only convenience method: it returns `False` when
the primary key already exists. `set(mode=IF_MISSING)` instead requires the
insertion to succeed and raises `ConflictError` if that key exists.

### ETags and conditional updates

Every successful insert or replacement generates a fresh, opaque ETag.
Fetching a record retains its ETag automatically, available as `record.etag`.
Changing Python fields does not change the ETag.

Use `mode=IF_NOT_MODIFIED` to protect a read-modify-write operation:

```python
task = Task.get(task_key)
if task is None:
    raise KeyError(task_key)

task.title = "Updated title"
task.set(mode=IF_NOT_MODIFIED)
```

The write succeeds only if the stored record still has the ETag that was read.
A mismatch or a record that has since been deleted raises `ConflictError`
without changing the record or its indexes. The comparison and update execute
atomically on Redis. A successful write refreshes the object's ETag.

Plain `set()` writes regardless of the previous ETag, subject to validation and
unique indexes. It also generates a new ETag, even if the field values have not
changed. ETags are random version tokens rather than content hashes; restoring
old field values does not restore an old ETag.

Newly constructed objects and key-only references have no ETag until a successful
write or an explicit fetch provides one. Requesting `mode=IF_NOT_MODIFIED`
without an ETag raises `ValueError` before the operation is sent or queued.

A conflict does not retry the operation automatically. Fetch the current record
and decide whether to reapply the change.

### Typed references

Construct a typed reference from a key without fetching the record:

```python
user_ref = User.ref(user.key)
task_ref = Task.ref(task.key)

assignment = Assignment(source=user_ref, target=task_ref, priority=3)
assignment.set()

assert assignment.source.key == user.key
loaded_user = User.get(assignment.source.key)
```

A reference exposes identity, not stored field values. Accessing an unloaded
field raises an error; it never triggers a request. Constructing a reference
does not check whether the record exists.

### Deletion

Delete through an instance or directly by primary key:

```python
deleted = Assignment.delete(user.key, task.key)  # True if the edge existed.
deleted_again = assignment.delete()             # False after removal.
user_deleted = User.delete(user.key)
```

The caller does not need to fetch an object first. Index cleanup uses the stored
record, not possibly stale field values on an instance. For composite node keys,
pass the key tuple to `Model.delete(key)`.

Deletion supports exactly two modes:

| Mode | Behavior |
| --- | --- |
| `ALWAYS` | Delete if present; return `True` if deleted and `False` if absent. The default. |
| `IF_NOT_MODIFIED` | Delete only if the record exists and its ETag matches the object's ETag; return `True` on success, otherwise raise `ConflictError`. |

```python
task = Task.get(task_key)
if task is None:
    raise KeyError(task_key)

deleted = task.delete(mode=IF_NOT_MODIFIED)
```

Conditional deletion compares the ETag and removes the record and its index
entries atomically. It requires an object carrying an ETag; a key-only reference
or model-and-key call cannot supply that version and raises `ValueError` if
used with `IF_NOT_MODIFIED`. `IF_MISSING` and `IF_EXISTS` are unsupported for
deletion and raise `ValueError`.

There are no cascading deletes. Deleting a node leaves its incoming and outgoing
edges intact. Deleting an edge leaves both endpoint nodes intact. Only index
entries belonging to the deleted record are removed automatically.

Applications explicitly remove any related records they no longer need. An
edge can therefore refer to a missing node; fetching that node returns `None`.

## Lookups and orderings

`Lookup` enforces uniqueness and supports exact matches. Every field in a
composite lookup must be supplied. It returns at most one record and provides no
ordered traversal.

`Ordering` supports traversal with duplicate values. Equality filters match
leading fields in the declaration; the following field supplies the ordering.

| Declaration | Example access |
| --- | --- |
| `Lookup("email")` | `User.get(email="alice@example.com")` |
| `Ordering("created_at")` | `Task.page(order_by="created_at", limit=20)` |
| `Ordering("priority")` on an edge | `Assignment.page(user, order_by="priority", reverse=True, limit=20)` |
| `Ordering("status", "created_at")` on an edge | `Assignment.page(user, where={"status": "open"}, order_by="created_at", reverse=True, limit=20)` |

Edge indexes are scoped to their source automatically. An
`Ordering("created_at")` indexes outgoing edges for each source.
A `Lookup("external_id")` on an edge makes that field unique within each source's
outgoing edges. The source does not need to appear in either declaration.

Queries must match a declared index. Unsupported filters or orderings raise an
error instead of scanning the data.

### Primary keys

Nodes automatically have a primary lookup on `key`; edges automatically have
one on their source and target. An explicit primary lookup replaces the default:

```python
class Document(Node):
    namespace = "archive"

    collection: str
    external_id: str
    title: str

    indexes = (
        Lookup("collection", "external_id", primary=True),
    )


document = Document(
    collection="manuals",
    external_id="installation",
    title="Installation guide",
)
document.set()

found = Document.get(collection="manuals", external_id="installation")
```

Here, `.key` is the composite `(collection, external_id)` identity, with no
additional UUID identity. Edge models can also declare an alternative primary
lookup.

Be careful not to change a record's key, fields used in a primary lookup, or
an edge's endpoints. Identity is not enforced as immutable. A subsequent write
uses the new identity and can leave the old record and references behind;
it does not rename or move the stored record.

### First, last, and count

`first()` and `last()` accept the same source, `where`, `order_by`, and
`reverse` arguments as `page()`, but no limit or cursor. They return one
record or `None` if the query is empty:

```python
oldest = Assignment.first(
    user,
    where={"status": "open"},
    order_by="created_at",
)
newest = Assignment.last(
    user,
    where={"status": "open"},
    order_by="created_at",
)
```

First and last refer to the requested ordering, including `reverse` and the
primary-key tiebreaker. They access the corresponding end of the index without
fetching the intervening records. Ordering is explicit; there is no model-level
default ordering.

`count()` returns the number of entries in a supported index scope without
loading model instances:

```python
open_count = Assignment.count(user, where={"status": "open"})
```

The source and equality filters must identify a scope that a declared index can
count directly. An optional `order_by` selects the same index used for ordered
traversal when needed:

```python
open_count = Assignment.count(
    user,
    where={"status": "open"},
    order_by="created_at",
)
```

An empty scope returns zero. Counts follow index membership, including the
omission of entries containing `None`. If eligible indexes would count different
memberships, the query must select an index explicitly. Unsupported scopes raise
an error; there is no fallback scan or automatically maintained global counter.
Counts describe current storage and may change before a subsequent page request.

## Pages and cursors

```python
page = Assignment.page(
    user,
    where={"status": "open"},
    order_by="created_at",
    reverse=True,
    limit=20,
)

for assignment in page:
    print(assignment.target.key, assignment.priority)

if page.cursor is not None:
    next_page = Assignment.page(
        user,
        where={"status": "open"},
        order_by="created_at",
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
ordering, and direction. Omitting the cursor starts at the beginning. A cursor
cannot be reused for a different query or an incompatible index definition.

Pagination continues exclusively after the last ordering value and primary-key
tiebreaker. Equal timestamps or priorities therefore have a deterministic order.

Pages read live data, not a snapshot. Inserts and deletes may change later pages.
Changing an ordering or filtering field between requests can cause a record to
be skipped or seen again.

## Transactions

Each write maintains its record and indexes together. Use explicit transaction
methods to group related writes:

```python
task = Task(title="Review the README")
assignment = Assignment(source=user, target=task)

with transaction() as tx:
    tx.set(task)
    tx.set(assignment)
```

Only calls through `tx` participate. Ordinary calls such as `task.set()` or
`User.delete(key)` execute independently, even inside the context block.
Pass the transaction object to helpers that need to contribute writes.

`tx.set(record)`, `tx.add(record)`, and `tx.delete(record)` queue their
operations. `tx.set()` accepts all four write modes; `tx.delete()` accepts
`ALWAYS` and `IF_NOT_MODIFIED`, with the same ETag requirements as immediate
deletion. Both default to `ALWAYS`. Deletion also accepts a model and primary
key, without fetching an object first:

```python
with transaction() as tx:
    tx.delete(Assignment, user.key, task.key)

deleted = tx.results[0]
```

Queued calls return `None`. A normal context exit commits the queued operations;
an exception escaping the block before commit discards them. After successful
execution, `tx.results` contains one result per operation in call order.
`add()` and `delete()` produce their boolean results at that point.

Queueing a write captures its mode, the record's values, and any expected ETag.
Later Python mutations do not change the queued operation.
Successful writes refresh their objects' ETags after commit is confirmed.
Discarding the transaction does not undo changes to Python objects.

### Conditions apply to the whole transaction

Use `tx.set(record, mode=IF_NOT_MODIFIED)` to make the transaction conditional
on a previously read version:

```python
task = Task.get(task_key)
if task is None:
    raise KeyError(task_key)

task.title = "Completed review"

with transaction() as tx:
    tx.set(task, mode=IF_NOT_MODIFIED)
    tx.delete(Assignment, user_key, task_key)
```

At commit, **all mode conditions are checked before any queued write is applied**.
This includes ETag matches for conditional updates and deletions, absence for
`IF_MISSING`, and presence for `IF_EXISTS`. If any condition fails, commit raises
`ConflictError` and none of the transaction's writes execute. In this example,
a conflict leaves both the task and the assignment unchanged by this transaction.

Condition checking and applying the writes form one atomic operation on Redis;
another client cannot change a checked record between those steps. Mode
conditions compare against stored state before the transaction's writes begin.
Unconditional writes in the same transaction are also withheld if any mode
condition fails.

`tx.add()` retains its boolean result: an existing primary key produces `False`
and does not cancel the transaction. Use `tx.set(record, mode=IF_MISSING)` when
insertion must succeed for the transaction to proceed.

Reads use ordinary model methods and execute immediately. They may occur before
or inside the block, but are not snapshot reads and do not see queued writes.
Only versions explicitly supplied through `mode=IF_NOT_MODIFIED` are checked;
other reads and query results are not protected. There is no local
read-your-writes cache or automatic retry.

### Limits

**Transactions cannot be nested.** Entering `transaction()` while one is active
raises `TransactionError`. There are no savepoints, and the public API does not
require `watch()` or `multi()` calls.

Atomic multi-key work requires one Redis primary, or keys in one Redis Cluster
hash slot. Transactions across cluster shards are not supported.

Mode-condition conflicts are detected before writes begin. This guarantee does
not imply general rollback after arbitrary execution errors. Redis does not provide that
rollback, and a raised execution error must not be taken to mean that nothing
was written. Individual record operations must still enforce uniqueness and
maintain their indexes.

A lost connection during commit can leave the outcome unknown. Persistence and
failover durability depend on the Redis deployment.

See [Redis transactions](https://redis.io/docs/latest/develop/using-commands/transactions/),
[Redis scripting](https://redis.io/docs/latest/develop/programmability/eval-intro/),
and the [Redis Cluster specification](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/)
for the underlying execution constraints.
