# Comfy Chair

> “Cardinal Fang! Fetch…THE COMFY CHAIR!”
>
> Monty Python, [The Spanish Inquisition](https://python.mzonline.com/sketches/spanishinquisition/)

Typed nodes, edges, and indexes for Python, backed by Redis.

Comfy Chair stores records by key and relationships as records with their own
metadata. Type annotations define fields. Primary keys identify records;
explicit indexes provide ordered traversal. Namespaces keep independent
collections of data separate.

Edges are records too: they store source and target keys along with fields such
as a role, status, or priority. Their indexes support queries over one source's
outgoing edges, including ordered traversal and pagination.

**Development status:** design stage. The API below specifies the intended
behavior; an implementation is not yet available. Examples assume a configured
Redis connection.

## Example

Define nodes and a typed relationship between them:

```python
from datetime import datetime, timezone
from typing import Literal

from comfychair import Commit, Edge, Field, Node, Ordering


def now():
    return datetime.now(timezone.utc)


class User(Node):
    name: str


class Task(Node):
    title: str
    created_at: datetime = Field(default_factory=now)

    indexes = (
        Ordering(created_at),
    )


class Assignment(Edge[User, Task]):
    status: Literal["open", "closed"] = Field(default="open")
    priority: int = Field(default=0)
    created_at: datetime = Field(default_factory=now)

    indexes = (
        Ordering(created_at),
        Ordering(priority),
        Ordering(status, created_at),
    )
```

Create records and store related writes together:

```python
user = User(name="Alice")
task = Task(title="Write the introduction")
assignment = Assignment(source=user, target=task, priority=3)

with Commit() as batch:
    batch.set(user)
    batch.set(task)
    batch.set(assignment)

same_user = User.get(user.key)
task.title = "Write the README introduction"
task.set()
```

List a user's open assignments, newest first:

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
```

An edge stores its endpoint keys. Reading an edge does not fetch either node.
Every query uses an automatic primary index or a declared ordering.

## Models and fields

Declare stored fields with Python type annotations. Constructors accept keyword
arguments. Use `Field` for defaults, factories, and fields referenced in index
declarations.

The annotation supplies the value type; `Field` supplies configuration and
query behavior. `Field()` without a default declares a required value.

Plain annotations and ordinary defaults remain shorthand:

```python
class Note(Node):
    title: str
    priority: int = 0
```

Comfy Chair creates descriptors for these fields during class creation. Use an
explicit `Field()` when configuring a field or referencing it within the class
body, as in `Ordering(created_at)`. On an instance, field access returns the
stored Python value; on a class, it returns the descriptor:

```python
Task.created_at                  # Field descriptor
task.created_at                  # datetime value on a Task instance
Task.created_at >= start         # Query predicate
```

`Node` supplies a `key: UUID` field with a generated UUID by default.
Declare it explicitly to configure its default:

```python
from uuid import UUID, uuid4


class Account(Node):
    key: UUID = Field(default_factory=uuid4)
    name: str
```

Primary lookup by key is automatic.

`Edge[User, Task]` supplies `source` and `target` fields whose value types
come from the endpoint models' primary keys. Both are UUIDs in this example.
An edge's `key` is the pair `(source, target)`, scoped to its model and
namespace. There is one edge per endpoint pair for a given edge model.
Different edge models can connect the same pair independently.

Edges store endpoint keys and their own fields, not copies of endpoint
records. The generic parameters identify the endpoint models. Fetching an
edge does not fetch either node. Relationships are directed; outgoing indexes
do not imply an incoming index.

### Primary keys

Nodes are identified by their `key` field; edges by their source and target
primary keys. Declare a node's `key` explicitly to use a supplied identifier
instead of a generated UUID:

```python
class Document(Node):
    key: str
    title: str


document = Document(key="installation", title="Installation guide")
document.set()
found = Document.get("installation")
```

`get()` and `exists()` accept primary keys, not secondary field filters.
Use an `Ordering` query to find records by other fields. Primary keys cannot
be `None`.

Be careful not to change a record's key or an edge's endpoints. Identity is not
enforced as immutable. A subsequent write uses the new identity and can leave
the old record and edges behind; it does not rename or move the stored
record.

### Namespaces and storage identity

Set an optional `namespace` on each model to prefix its storage names:

```python
class Note(Node):
    namespace = "notes_plugin"
    title: str
```

Namespaces let separate packages define models with the same class name in one
Redis database without collisions. Omit the attribute to use no namespace
prefix. Model name and namespace together distinguish record types.

Edges may connect models in different namespaces. The edge declaration
identifies each endpoint model and its namespace. A namespace is a naming
prefix, not an authorization boundary or a Redis Cluster hash-slot assignment.

Model names and namespaces are part of storage identity. Renaming a model or
changing its namespace changes where it reads and writes; existing data is not
renamed or migrated automatically. Changing index declarations likewise does
not rebuild existing indexes. Take care when changing these declarations for
models that already have stored data.

## Store and retrieve

Constructing or changing a Python object performs no I/O. Persistence is explicit.
The methods below execute immediately. To queue a write in a `Commit`, call
its counterpart on the batch object.

| Method | Behavior |
| --- | --- |
| `Model.get(...)` | Return the matching record; raise `KeyError` if absent unless an explicit `default` is supplied. |
| `Model.exists(...)` | Return whether a record with the primary key exists. |
| `record.refresh()` | Replace fields and ETag in place with the stored version; return `None`, or raise `KeyError` if absent. |
| `Model.get_or_create(...)` | Return `(record, created)`, retrieving a match or inserting a new record. |
| `record.set()` | Insert or replace the record and update its indexes. |
| `record.add()` | Insert only; return `True` if inserted, `False` if its primary key already exists. |
| `record.delete()` or `Model.delete(key)` | Remove the record and its index entries; return `True` if a record was deleted, `False` if absent. |

### Missing records and defaults

`get()`, `first()`, and `last()` share one absence policy: omitting
`default` raises `KeyError` when no record matches. Supplying the keyword-only
`default` returns that exact value instead.

```python
task = Task.get(task_key)                # KeyError if absent.
optional_task = Task.get(task_key, default=None)

missing = object()
result = User.get(user_key, default=missing)
if result is missing:
    print("No matching user")
```

An explicit default applies only to absence. Invalid queries, connection
failures, validation failures, and decoding errors still raise; they are not
converted to the default value.

### Existence and refresh

`exists()` accepts the same primary-key arguments as `get()`.
It returns a boolean without loading a model instance:

```python
has_user = User.exists(user.key)
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

Use `get_or_create()` to retrieve a record by primary key or insert it if absent:

```python
task, created = Task.get_or_create(
    task_key,
    defaults={"title": "Write the introduction"},
)
```

Identity arguments follow `get()`, including the two endpoints for an edge.
`defaults` supplies fields only when creating; normal
field defaults and factories supply the remaining values. Defaults cannot
override the supplied identity, and `None` is not a valid primary key.

If a match exists, return it with `False` and leave its fields and ETag unchanged.
Otherwise, validate and insert the new record and return it with `True`.
Concurrent calls for the same primary key cannot create duplicate records.

`get_or_create()` executes immediately and is not a queued `Commit` method.

### Write modes

For an immediate `set()`, the mode controls whether the write is allowed:

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
its indexes. Every mode still enforces validation.
Unsupported modes raise `ValueError` before the operation is sent or queued.

For an immediate write, `add()` returns `False` when the primary key already
exists; `set(mode=IF_MISSING)` raises `ConflictError` instead.

Inside a `Commit`, modes are preconditions on the stored state before any
batch writes. They are not rechecked between operations. Once all conditions
pass, sets insert or replace in call order, while `add()` remains insert-only
at its position in that order. See [batch conditions](#conditions-apply-to-the-whole-batch).

### ETags and conditional updates

Every successful insert or replacement generates a fresh, opaque ETag.
Fetching a record retains its ETag automatically, available as `record.etag`.
Changing Python fields does not change the ETag.

Use `mode=IF_NOT_MODIFIED` to protect a read-modify-write operation:

```python
task = Task.get(task_key)

task.title = "Updated title"
task.set(mode=IF_NOT_MODIFIED)
```

The write succeeds only if the stored record still has the ETag that was read.
A mismatch or a record that has since been deleted raises `ConflictError`
without changing the record or its indexes. The comparison and update execute
atomically on Redis. A successful write refreshes the object's ETag.

Plain `set()` writes regardless of the previous ETag, subject to validation.
It also generates a new ETag, even if the field values have not changed.
ETags are random version tokens rather than content hashes; restoring
old field values does not restore an old ETag.

Newly constructed records have no ETag until a successful write provides one;
fetched records carry the stored ETag. Primary-key values do not carry ETags.
Requesting `mode=IF_NOT_MODIFIED` without a record ETag raises
`ValueError` before writing. For queued operations, the record's ETag is read
and checked when the context exits.

A conflict does not retry the operation automatically. Fetch the current record
and decide whether to reapply the change.

### Edge endpoints

Model relationships are stored as edges. Their `source` and `target` attributes
are ordinary primary-key values, used directly with model methods:

```python
assignment = Assignment(source=user.key, target=task.key, priority=3)
assignment.set()

loaded_user = User.get(assignment.source)
loaded_task = Task.get(assignment.target)
```

An edge constructor also accepts instances of the declared endpoint models
as shorthand and extracts their keys:

```python
assignment = Assignment(source=user, target=task)
assert assignment.source == user.key
assert assignment.target == task.key
```

Edge methods accept endpoint keys or the corresponding model instances;
source-scoped queries accept either form for the source. Passing a key never
fetches the endpoint or checks its existence. There are no reference wrappers,
embedded model fields, or implicit fetches.

### Deletion

Delete through an instance or directly by primary key:

```python
deleted = Assignment.delete(user.key, task.key)  # True if the edge existed.
deleted_again = assignment.delete()             # False after removal.
user_deleted = User.delete(user.key)
```

The caller does not need to fetch an object first. Index cleanup uses the stored
record, not possibly stale field values on an instance.

Immediate deletion supports exactly two modes:

| Mode | Behavior |
| --- | --- |
| `ALWAYS` | Delete if present; return `True` if deleted and `False` if absent. The default. |
| `IF_NOT_MODIFIED` | Delete only if the record exists and its ETag matches the object's ETag; return `True` on success, otherwise raise `ConflictError`. |

```python
task = Task.get(task_key)

deleted = task.delete(mode=IF_NOT_MODIFIED)
```

Conditional deletion compares the ETag and removes the record and its index
entries atomically. It requires an object carrying an ETag; a model-and-key
call cannot supply that version and raises `ValueError` if
used with `IF_NOT_MODIFIED`. `IF_MISSING` and `IF_EXISTS` are unsupported for
deletion and raise `ValueError`.

In a `Commit`, conditional deletion checks the initial stored ETag along with
the other batch preconditions. Its result then reflects whether that operation
actually deleted a record, so it can return `False` if an earlier operation
in the batch already removed the record.

There are no cascading deletes. Deleting a node leaves its incoming and outgoing
edges intact. Deleting an edge leaves both endpoint nodes intact. Only index
entries belonging to the deleted record are removed automatically.

Applications explicitly remove any related records they no longer need. An
edge can therefore refer to a missing node.

## Ordered indexes

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

### Automatic edge traversal

Every edge model has a primary index on `(source, target)`. Supplying a source
supports `page()`, `first()`, `last()`, and `count()` without an `Ordering`
declaration. Traversal uses ascending target-key order by default; use
`reverse=True` to reverse it. The target's key type determines its ordering.

```python
page = Assignment.page(user, limit=20)
first = Assignment.first(user, default=None)
last = Assignment.last(user, default=None)
total = Assignment.count(user)
```

For edge `page()`, `first()`, and `last()`, omitting `order_by` selects this
primary traversal. `order_by=Assignment.target` selects it explicitly. Pagination uses the same
`Page` and cursor rules as declared orderings. A source-only count includes
every outgoing edge for that model and source.

Filters on other fields must fit a declared `Ordering`, selected with
`order_by` for traversal. Node traversal requires an explicit ordering.

### Predicates and ranges

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

`page()`, `first()`, `last()`, and `count()` accept positional predicates.
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

### First, last, and count

`first()` and `last()` accept the same source, positional predicates, `order_by`,
and `reverse` arguments as `page()`, but no limit or cursor. They follow the
[missing-record policy](#missing-records-and-defaults):
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
fetching the intervening records. Edge queries without `order_by` use the
fixed primary order by target; other traversal requires an explicit ordering.
There is no configurable model-level default ordering.

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

An empty scope returns zero. Counts follow index membership, including the
omission of entries containing `None`. If eligible indexes would count different
memberships, the query must select an index explicitly. Unsupported scopes raise
an error; there is no fallback scan or automatically maintained global counter.
Counts describe current storage and may change before a subsequent page request.

### Indexed values

The built-in indexed field types have deliberately limited rules:

| Field type | Equality and ordering |
| --- | --- |
| `str` | Exact, case-sensitive equality; Unicode code-point order, without locale collation or normalization. |
| `bool` | Exact equality; `False` sorts before `True`. |
| `int` | Exact equality and numeric order within the signed 64-bit range. |
| `float` | Finite IEEE 754 binary64 values in numeric order; negative and positive zero compare equal. NaN and infinities are rejected. |
| `UUID` | Exact equality and unsigned 128-bit numeric order. |
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

## Pages and cursors

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

Pagination continues exclusively after the last ordering value and primary-key
tiebreaker. Equal timestamps or priorities therefore have a deterministic order.

Pages read live data, not a snapshot. Inserts and deletes may change later pages.
Changing an ordering or filtering field between requests can cause a record to
be skipped or seen again.

## Commits

Each write maintains its record and indexes together. Use `Commit` to collect
related writes in a batch. On successful context exit, it checks all mode
conditions and applies the writes atomically.

Only calls through `batch` participate. Ordinary calls such as `task.set()` or
`User.delete(key)` execute independently, even inside the context block.
Pass the batch object to helpers that need to contribute writes.

`batch.set(record)`, `batch.add(record)`, and `batch.delete(record)` queue their
operations. `batch.set()` accepts all four write modes; `batch.delete()` accepts
`ALWAYS` and `IF_NOT_MODIFIED`, with the same ETag requirements as immediate
deletion. Both default to `ALWAYS`. Deletion also accepts a model and primary
key, without fetching an object first:

```python
with Commit() as batch:
    removed = batch.delete(Assignment, user.key, task.key)

if removed.value:
    print("Assignment removed")
```

Each queued call returns a `Result[T]` handle. Its `.value` becomes available
only after the whole commit succeeds and is confirmed. A normal context exit
commits the queued operations; an exception escaping the block before commit
discards them.

| Queued method | Handle | Value after successful commit |
| --- | --- | --- |
| `batch.set(record)` | `Result[None]` | `None`, matching immediate `set()`. |
| `batch.add(record)` | `Result[bool]` | `True` if inserted, `False` if the primary key already exists. |
| `batch.delete(...)` | `Result[bool]` | `True` if deleted at this point in the operation order, `False` if absent. Conditional deletion requires an initial ETag match. |

Results stay attached to their operations and can be returned from helpers:

```python
def remove_assignment(batch, user_key, task_key):
    return batch.delete(Assignment, user_key, task_key)


with Commit() as batch:
    saved = batch.set(task, mode=IF_NOT_MODIFIED)
    removed = remove_assignment(batch, user.key, task.key)

saved.value    # None
removed.value  # True if deleted, False if absent
```

Reading `.value` before commit raises `CommitError`; it never waits or triggers
execution. If the block is discarded or commit fails, including an unconfirmed
outcome after connection loss, `.value` raises `CommitError` explaining that no
confirmed result is available. This does not imply that nothing was written.
The original failure still propagates from the context manager.

Converting a result handle to a boolean raises `TypeError`. Write
`if removed.value:`, not `if removed:`. Handles may be ignored when the caller
does not need the result.

Queueing records the operation, mode, and object reference. On normal context
exit, Comfy Chair captures the objects' current identities, field values, and
ETags, then validates and serializes the captured values before writing:

```python
with Commit() as batch:
    task.title = "First"
    batch.set(task)
    task.title = "Second"

# Stores "Second" and updates task.etag.
```

Multiple queued sets of the same object all use its values at context exit.
Model-and-key deletes retain their supplied identity arguments. Successful
writes update the objects' ETags internally after commit is confirmed; they do
not return replacement objects. For repeated successful writes through the same
object, its ETag reflects the last such write. Discarding the batch does not
undo Python edits.

### Conditions apply to the whole batch

Use `batch.set(record, mode=IF_NOT_MODIFIED)` to make the batch conditional
on a previously read version:

```python
task = Task.get(task_key)

task.title = "Completed review"

with Commit() as batch:
    batch.set(task, mode=IF_NOT_MODIFIED)
    batch.delete(Assignment, user_key, task_key)
```

At commit, **all mode conditions are checked before any queued write is applied**.
This includes ETag matches for conditional updates and deletions, absence for
`IF_MISSING`, and presence for `IF_EXISTS`. If any condition fails, commit raises
`ConflictError` and none of the batch's writes execute. In this example,
a conflict leaves both the task and the assignment unchanged by this batch.

**Conditions gate the batch using its starting state; operation results reflect
execution order.** Starting state means the stored state immediately before
the batch's writes, not when the Python context was entered.

Condition checking and applying the writes form one atomic operation on Redis;
another client cannot change a checked record between those steps. Unconditional
writes in the same batch are also withheld if any mode condition fails.

A primary key present when a `batch.add()` executes produces `False` without
cancelling the batch. Earlier writes in the batch affect that result. Use
`batch.set(record, mode=IF_MISSING)` when initial absence is a required
condition for the batch to proceed.

After checking preconditions, operations execute in call order. Their ordinary
effects and boolean results reflect preceding operations in the batch: two
`add()` calls for an initially absent key yield `True`, then `False`; a set
followed by a delete writes and then removes the record. ETag conditions always
compare against the stored state before any batch writes. A conditional set
and delete carrying the same matching ETag therefore both pass their checks;
the set's new ETag does not invalidate the delete.

Two `set(..., mode=IF_MISSING)` calls for the same initially absent key both
pass their preconditions. Both sets execute, and the second replaces the first.
Similarly, two conditional deletes of the same initial version both pass their
ETag checks, but only the first removes a record:

```python
task = Task.get(task_key)

with Commit() as batch:
    first = batch.delete(task, mode=IF_NOT_MODIFIED)
    second = batch.delete(task, mode=IF_NOT_MODIFIED)

assert first.value is True
assert second.value is False
```

A conditional set can also recreate a record deleted earlier in the same
batch, provided its precondition passed against the initial stored state.

Reads use ordinary model methods and execute immediately. They may occur before
or inside the block, but are not snapshot reads and do not see queued writes.
Only versions explicitly supplied through `mode=IF_NOT_MODIFIED` are checked;
other reads and query results are not protected. There is no local
read-your-writes cache or automatic retry.

### Limits

**Commits cannot be nested.** Entering `Commit()` while another is active
raises `CommitError`. There are no savepoints.

Atomic multi-key work requires one Redis primary, or keys in one Redis Cluster
hash slot. Transactions across cluster shards are not supported.

Mode-condition conflicts are detected before writes begin. This guarantee does
not imply general rollback after arbitrary execution errors. Redis does not provide that
rollback, and a raised execution error must not be taken to mean that nothing
was written. Individual record operations must still maintain their indexes.

A lost connection during commit can leave the outcome unknown. Persistence and
failover durability depend on the Redis deployment.

See [Redis transactions](https://redis.io/docs/latest/develop/using-commands/transactions/),
[Redis scripting](https://redis.io/docs/latest/develop/programmability/eval-intro/),
and the [Redis Cluster specification](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/)
for the underlying execution constraints.

## Relationships and exclusive ownership

Edges identify pairs, so several targets can share one source. A rule such as
one owner per email is maintained by application workflows, not enforced by
the edge type or an ordered index.

Use an email node as the claim on an address and an edge for its owner.
The `User` model from the example stores a name, with no duplicated email
field or embedded relationship:

```python
class Email(Node):
    key: str


class EmailOwner(Edge[Email, User]):
    pass
```

An `EmailOwner` has a string `source` and UUID `target`. Find the owner using
only the email address through automatic source traversal:

```python
ownership = EmailOwner.first("alice@example.com")
user = User.get(ownership.target)
```

No `Ordering` declaration is needed. `first()` raises `KeyError` if no edge
exists, unless an explicit default is supplied. It selects one edge; the
application's ownership workflow ensures that there is only one.

### Concurrent creation

Create the email claim, user, and edge together. Addresses in this example
have already been normalized according to the application's rules.

```python
def create_user(name, address):
    user = User(name=name)
    email = Email(key=address)
    ownership = EmailOwner(source=email, target=user)

    with Commit() as batch:
        batch.set(email, mode=IF_MISSING)
        batch.set(user, mode=IF_MISSING)
        batch.set(ownership, mode=IF_MISSING)

    return user
```

Concurrent requests for the same address contend on the same `Email` key.
One commit succeeds; another finds the email claim present and raises
`ConflictError` before any of its writes. The losing request creates no user
or edge. Use `set(mode=IF_MISSING)` for this requirement: a failed `add()`
would produce `False` without cancelling the other writes.

This workflow reserves the address. Creating additional edges directly would
bypass the one-owner rule. Deleting an ownership edge alone does not release
the email claim; applications manage that lifecycle explicitly.

### Swapping owners

Given two distinct addresses and users with consistent existing ownership,
replace the two edges. Neither email nor user node changes:

```python
def swap_emails(email_a_key, user_a_key, email_b_key, user_b_key):
    old_a = EmailOwner.get(email_a_key, user_a_key)
    old_b = EmailOwner.get(email_b_key, user_b_key)

    new_a = EmailOwner(source=email_a_key, target=user_b_key)
    new_b = EmailOwner(source=email_b_key, target=user_a_key)

    with Commit() as batch:
        batch.delete(old_a, mode=IF_NOT_MODIFIED)
        batch.delete(old_b, mode=IF_NOT_MODIFIED)
        batch.set(new_a, mode=IF_MISSING)
        batch.set(new_b, mode=IF_MISSING)
```

The old edges' ETags must still match, and the new edges must be absent.
All four conditions are checked before any writes; a conflict leaves the
stored relationships unchanged by this batch. Ownership-changing workflows
must preserve the one-owner rule. These examples have the same Redis primary
or Cluster hash-slot requirement as other commits.

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
writing. Writes in a `Commit` validate the captured object values at context
exit, before any writes. Mutating an object's fields alone does not trigger
validation. Local validation does not establish existence or ETag matches;
those checks happen atomically on Redis during the operation.

Model definitions are checked at class creation. Invalid declarations raise
`TypeError`, including index fields that do not belong to the model,
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

Edges serialize their source and target primary-key values using the types
specified by their endpoint models. Serialization does not embed or recursively
persist related nodes. Index encodings are separate from record JSON: an
ordered index must preserve the field's comparison semantics rather than sort
its JSON text.
