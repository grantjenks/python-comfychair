# Store and retrieve

[README](../README.md) · [Documentation](../README.md#documentation)

Examples use the models defined in the [introductory example](example.md).

Constructing or changing a Python object performs no I/O. Persistence is explicit.
The methods below execute immediately. To queue a set, add, or delete in a
`Commit`, call its counterpart on the batch object.

| Method | Behavior |
| --- | --- |
| `Model.get(...)` | Return the matching record; raise `KeyError` if absent unless an explicit `default` is supplied. |
| `Model.exists(...)` | Return whether a record with the primary key exists. |
| `record.refresh()` | Replace fields and ETag in place with the stored version; return `None`, or raise `KeyError` if absent. |
| `Model.get_or_create(...)` | Return `(record, created)`, retrieving a match or inserting a new record. |
| `record.set()` | Insert or replace the record and update its indexes. |
| `record.mutate(callables, attempts=3)` | Apply changes to one record, retry conditional writes on conflicts, and update the object in place; return `None`. |
| `record.add()` | Insert only; return `True` if inserted, `False` if its primary key already exists. |
| `record.delete()` or `Model.delete(key)` | Remove the record and its index entries; return `True` if a record was deleted, `False` if absent. |

## Missing records and defaults

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

## Existence and refresh

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

## Get or create

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

## Write modes

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
at its position in that order. See [batch conditions](commits.md#conditions-apply-to-the-whole-batch).

## ETags and conditional updates

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

An ordinary `set()` or `delete()` does not retry on conflict. Fetch the current
record and decide whether to reapply the change, or use `mutate()` to replay
changes to one record.

## Deletion

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
