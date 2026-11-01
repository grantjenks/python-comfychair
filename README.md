# Comfychair

> “Cardinal Fang! Fetch…THE COMFY CHAIR!”
>
> Monty Python, [The Spanish Inquisition](https://python.mzonline.com/sketches/spanishinquisition/)

Typed nodes, edges, and indexes for Python, backed by Redis.

Comfychair stores objects by key and relationships as records of their own. A
relationship has a source, a target, and ordinary fields: when a user was assigned
a task, its priority, or its status. Retrieve one record by key, find it through a
unique lookup, or page through a node's relationships in a declared order.

**Status: API design proposal.** This README describes the intended interface for
review. There is no implementation or installable release in this repository yet.
Examples assume models have been bound to a Redis connection; connection
configuration remains to be designed.

## Define records

```python
from dataclasses import field
from datetime import datetime, timezone
from uuid import UUID, uuid4

from comfychair import Edge, Lookup, Node, Ordering, transaction


def now():
    return datetime.now(timezone.utc)


class User(Node):
    key: UUID = field(default_factory=uuid4)
    name: str
    email: str

    indexes = (
        Lookup("email"),
    )


class Task(Node):
    title: str
    created_at: datetime = field(default_factory=now)

    indexes = (
        Ordering("created_at"),
    )


class Assignment(Edge[User, Task]):
    status: str = "open"
    priority: int = 0
    created_at: datetime = field(default_factory=now)

    indexes = (
        Ordering("created_at"),
        Ordering("priority"),
        Ordering("status", "created_at"),
    )
```

Field annotations define stored data. Constructors accept keyword arguments;
defaults and factories use dataclass-style declarations.

`Node` supplies a `key: UUID` field with a generated UUID by default. `User` spells
out that declaration for clarity; `Task` inherits it. Names and titles are ordinary
mutable fields. Primary lookup by key is automatic.

`Edge[User, Task]` supplies typed `source` and `target` endpoints. An assignment's
default key is the pair `(source.key, target.key)`, scoped to the `Assignment`
model. There is one assignment per user/task pair. Another edge model may connect
the same pair independently.

Edges store endpoint keys and their own fields, not copies of the endpoint
records. Fetching an edge does not fetch its user and task. Relationships are
directed; a source's outgoing list does not imply an index of incoming edges.

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

assignment.delete()
```

Constructing or changing a Python object performs no I/O. Call `set()` to insert
or replace the stored record. Replacing a record also updates its indexes.
`get()` returns a record or `None` when no match exists. `delete()` removes the
record and its index entries.

Use `add()` for insert-only behavior. Outside a transaction it returns `True`
when inserted and `False` when the primary key already exists. A conflicting
secondary unique lookup raises `IntegrityError`; it never replaces the other
record. Conditional-operation results inside a transaction still need a precise
contract and are not illustrated here.

Persistence methods belong to the record or its model. There is no dictionary
interface for storing records and no separate `db.get(User, ...)` interface.

## Lookups and orderings

An index declaration names stored fields. Two index types cover different access
patterns:

| Declaration | Purpose | Access |
| --- | --- | --- |
| `Lookup("email")` | Unique exact match | `User.get(email="alice@example.com")` |
| `Ordering("created_at")` | Ordered traversal | `Task.page(order_by="created_at", limit=20)` |
| `Ordering("priority")` on an edge | Ordered traversal for one source | `Assignment.page(user, order_by="priority", reverse=True, limit=20)` |
| `Ordering("status", "created_at")` on an edge | Match a leading field, then traverse in order | `Assignment.page(user, where={"status": "open"}, order_by="created_at", reverse=True, limit=20)` |

`Lookup` enforces uniqueness and returns at most one record. A lookup with
multiple fields requires an exact value for every field. It does not provide
ordered traversal.

`Ordering` allows duplicate values. Equality filters match leading fields in
the declaration; the following field supplies the requested ordering. Queries
must match a declared index. Unsupported filters or orderings raise an error
instead of silently scanning the data.

Edge indexes are scoped to the source automatically. Declare
`Ordering("created_at")`, not `Ordering("source", "created_at")`. Likewise,
`Lookup("external_id")` on an edge would make `external_id` unique within each
source's outgoing edges.

Default primary lookups require no declaration: a node uses its key, and an edge
uses its source and target. An explicit primary lookup can replace the default:

```python
class ExternalTask(Node):
    tenant: UUID
    external_id: str
    title: str

    indexes = (
        Lookup("tenant", "external_id", primary=True),
    )


external = ExternalTask(
    tenant=user.key,
    external_id="TASK-42",
    title="Review the proposal",
)
external.set()

found = ExternalTask.get(tenant=user.key, external_id="TASK-42")
```

Here, `.key` is the composite `(tenant, external_id)` identity; there is no
additional generated UUID identity. Edge models may also declare an alternative
primary lookup. Primary identity and source-scoped secondary indexes are distinct
concerns; the exact rules for custom edge identities remain part of the design.

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
`len(page)` operate on the fetched records without further requests. Its length
is the number of records in this page, not the total number of matching records.

`page.cursor` is an opaque, serializable continuation token, or `None` when the
query is exhausted. Pass it back with the same source, filters, ordering, and
direction. Omitting the cursor starts at the beginning. A cursor cannot be reused
for a different query or an incompatible index definition.

Pagination continues exclusively after the last ordering value and primary-key
tiebreaker. Equal timestamps or priorities therefore have a deterministic order.
Pages read live data, not a snapshot: inserts and deletes may change later pages,
and changing an ordering or filtering field between requests can cause a record
to be skipped or seen again.

## Transactions

Use a transaction to group related writes:

```python
task = Task(title="Review the README")
assignment = Assignment(source=user, target=task)

with transaction():
    task.set()
    assignment.set()
```

Writes join the active transaction. A normal exit commits the queued work; an
exception escaping the block discards it before commit. `set()` captures the
record's state when called, so later Python mutations do not silently alter an
already queued write. Aborting persistence does not undo changes to Python
objects.

Each individual write must maintain its record and indexes together. An explicit
transaction groups those writes so other clients cannot observe intermediate
steps of a successful commit.

**Transactions cannot be nested.** Entering `transaction()` while one is active
raises `TransactionError`. There are no savepoints or independently rolled-back
inner scopes. Helpers called inside a transaction should perform ordinary writes
and let their caller own the transaction boundary.

These guarantees must fit Redis's execution model:

- Atomic multi-key work requires a single Redis primary, or keys in one Redis
  Cluster hash slot. Arbitrary transactions across cluster shards are outside
  the initial scope.
- Redis `MULTI`/`EXEC` does not undo commands that have already run when another
  command fails during execution. Comfychair must validate operations and unique
  constraints before applying writes; transaction syntax alone cannot provide
  SQL-style rollback.
- Isolation does not imply durable replication. Persistence and failover behavior
  depend on the Redis deployment.

The commit protocol, conflict detection, reads within a transaction, and handling
of ambiguous outcomes after connection loss still require implementation design.
This proposal does not promise serializable read/modify/write transactions or
automatic retries of a Python `with` block.

See [Redis transactions](https://redis.io/docs/latest/develop/using-commands/transactions/)
and the [Redis Cluster specification](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/)
for the underlying constraints.

## Bulk operations

Bulk methods are a proposed convenience for reducing round trips:

```python
users = User.get_many([user.key, another_user_key])
User.set_many(changed_users)
User.delete_many(obsolete_user_keys)
```

`get_many()` returns results in input order, with `None` for missing records.
Bulk writes preserve the consistency of each record and its indexes, but are not
atomic as a group unless enclosed in an explicit transaction. Pipelining is an
implementation detail. There is no separate `batch()` context or deferred-result
API in this proposal.

## Scope and remaining design

Comfychair targets applications that know their access patterns: retrieve a
record, follow a typed relationship, or list one node's relationships in a useful
order. It does not propose joins, arbitrary graph traversal, a query language, or
automatic full scans.

Before implementation, the remaining design work includes:

- Connection configuration, model registration, and key namespaces.
- Field validation, serialization, and supported Python types.
- The representation of typed endpoint references and explicit endpoint loading.
- Custom edge identities and alternative index scopes, including incoming edges.
- Transaction reads, conflict reporting, conditional results, and commit failures.
- Node deletion behavior, dangling edges, and any explicit cascade operations.
- Schema changes, index creation, and index backfills for existing records.
- Async support, packaging, and licensing.

The examples establish the proposed API shape. They are not performance claims
or a statement that these behaviors have already been implemented.
