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

    status: str = "open"
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

## Serialization

Record values are serialized as JSON. Standard JSON values include strings,
numbers, booleans, nulls, arrays, and objects.

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
The return values below apply outside a transaction; queued operations produce
their results when the transaction executes.

| Method | Behavior |
| --- | --- |
| `Model.get(...)` | Return the matching record, or `None` if absent. |
| `record.set()` | Insert or replace the record and update its indexes. |
| `record.add()` | Insert only; return `True` if inserted, `False` if its primary key already exists. |
| `record.delete()` or `Model.delete(key)` | Remove the record and its index entries; return `True` if a record was deleted, `False` if absent. |

A conflicting secondary unique lookup raises `IntegrityError`. It never
replaces another record.

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

Each write maintains its record and indexes together. Use a transaction to group
related writes:

```python
task = Task(title="Review the README")
assignment = Assignment(source=user, target=task)

with transaction() as tx:
    task.set()
    assignment.set()
```

Transactions follow Redis `MULTI`/`EXEC` semantics. Operations join the active
transaction and are queued, not executed immediately. A normal exit executes
them in order; an exception escaping the block before execution discards them.
Other clients cannot interleave commands during execution. Aborting persistence
does not undo changes to Python objects.

Queued calls return `None`. After successful execution, `tx.results` contains
one result per model operation, in call order. In particular, `add()` and
`delete()` produce their boolean results only after execution:

```python
with transaction() as tx:
    Assignment.delete(user.key, task.key)

deleted = tx.results[0]
```

A queued read cannot supply a value for Python code to branch on inside the
block. Redis executes it in sequence and returns its result with the other
results. There is no local read-your-writes cache, automatic conflict detection,
or automatic retry. Reading before entering the transaction does not protect
that read from concurrent changes.

**Transactions cannot be nested.** Entering `transaction()` while one is active
raises `TransactionError`. There are no savepoints. Helpers called inside a
transaction perform ordinary writes and let their caller own the transaction
boundary.

Atomic multi-key work requires one Redis primary, or keys in one Redis Cluster
hash slot. Transactions across cluster shards are not supported.

Redis `MULTI`/`EXEC` does not roll back previously executed commands when
another command fails during execution. Other queued commands still execute.
Each record operation must maintain its own indexes and enforce uniqueness, but
an error in one operation does not undo other operations in the transaction.
A raised execution error therefore does not mean that nothing was written.

A lost connection during execution can leave the outcome unknown. Persistence
and failover durability depend on the Redis deployment.

See [Redis transactions](https://redis.io/docs/latest/develop/using-commands/transactions/)
and the [Redis Cluster specification](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/)
for the underlying constraints.
