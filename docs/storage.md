# Store and Retrieve

[Documentation](../README.md#documentation) · [Example models](example.md)

Construction and field assignment perform no I/O. Model and instance methods
execute immediately. Queued writes use [Commit](commits.md) batch methods.

| Method | Result |
| --- | --- |
| `Model.get(...)` | Matching record. |
| `Model.exists(...)` | Whether the primary key exists. |
| `record.refresh()` | Replace fields and ETag in place; return `None`. |
| `Model.get_or_create(...)` | `(record, created)`. |
| `record.set()` | Insert or replace the record and its indexes; return `None`. |
| `record.mutate(callables, attempts=3)` | [Apply and retry mutations](mutations.md); return `None`. |
| `record.add()` | `True` if inserted; `False` if the primary key exists. |
| `record.delete()` or `Model.delete(key)` | `True` if deleted; `False` if absent. |

All storage methods accept keyword-only `using`. Selection uses the explicit
database, then the instance's remembered database, then the model binding,
then the current default database.
See [Database Configuration](database.md).

## Missing Records and Defaults

`get()`, `first()`, and `last()` raise `KeyError` when no record matches. The
keyword-only `default` argument returns the supplied object instead.

```python
task = Task.get(task_key)                # KeyError if absent.
optional_task = Task.get(task_key, default=None)

missing = object()
result = User.get(user_key, default=missing)
if result is missing:
    print("No matching user")
```

`default` applies only to absence. Query, connection, validation, and decoding
errors propagate.

## Exists

Return whether a primary key exists, without loading a model instance.
Arguments follow `get()`.

```python
has_user = User.exists(user.key)
has_assignment = Assignment.exists(user.key, task.key)
```

## Refresh

Load a replacement through the normal
[construction path](models.md#construction-and-loading), then copy its stored
fields and ETag into the existing instance. Discard unsaved edits and return
`None`. Raise `KeyError` if absent. Any loading failure leaves the existing
instance unchanged.

```python
task.refresh()
```

Related nodes are not loaded. Reads, including `exists()` and `refresh()`,
observe current storage without reserving or locking records for subsequent
writes.

## Get or Create

Retrieve a record by primary key or insert it if absent. Return
`(record, created)`.

```python
task, created = Task.get_or_create(
    task_key,
    defaults={"title": "Write the introduction"},
)
```

Identity arguments follow `get()`, including both endpoints for edges.
`defaults` supplies creation fields; field defaults and factories supply
remaining values. `defaults` cannot override identity. Primary keys cannot be
`None`.

An existing record returns with `created=False`; its fields and ETag remain
unchanged. Otherwise, validate and insert the record, returning `created=True`.
Concurrent calls for one primary key cannot create duplicates.

`get_or_create()` executes immediately. It is not a batch method.

## Write Modes

`set(mode=ALWAYS)` inserts or replaces a record, subject to the selected
condition:

| Mode | Condition |
| --- | --- |
| `ALWAYS` | None. Default. |
| `IF_NOT_MODIFIED` | Record exists and stored ETag matches the object's ETag. |
| `IF_MISSING` | Primary key is absent. |
| `IF_EXISTS` | Primary key exists; ETag is ignored. |

Modes are `Mode` enum members, also exported as module constants.
`IF_NOT_MODIFIED` and `Mode.IF_NOT_MODIFIED` are identical.

```python
from comfychair import ALWAYS, IF_NOT_MODIFIED, IF_MISSING, IF_EXISTS, Mode

new_task = Task(title="Write the installation guide")
new_task.set(mode=IF_MISSING)

new_task.title = "Revise the installation guide"
new_task.set(mode=IF_EXISTS)
```

A failed condition raises `ConflictError` without changing the record or
indexes. All modes validate. Unsupported modes raise `ValueError` before
execution or queueing.

`add()` returns `False` for an existing key; `set(mode=IF_MISSING)` raises
`ConflictError`.

In a batch, conditions apply to the stored state before all writes and are not
rechecked between operations. Sets then insert or replace in call order;
`add()` remains insert-only at its position. See
[batch conditions](commits.md#conditions-apply-to-the-whole-batch).

## ETags and Conditional Updates

`record.etag` holds an opaque random version token. Every successful insert or
replacement generates a new ETag, including writes of unchanged values. Reads
retain the stored ETag; field assignments do not change it. Restoring old field
values does not restore an old ETag.

```python
task = Task.get(task_key)

task.title = "Updated title"
task.set(mode=IF_NOT_MODIFIED)
```

`IF_NOT_MODIFIED` atomically compares the stored ETag and writes the record and
indexes on Redis. A stale ETag or deleted record raises `ConflictError` without
changes. Confirmed writes update the object's ETag. Plain `set()` ignores the
previous ETag but still validates.

New records have no ETag until written. Primary keys do not carry ETags.
`IF_NOT_MODIFIED` without an object ETag raises `ValueError` before writing.
Queued operations capture and check ETags at context exit.

`set()` and `delete()` do not retry conflicts. Reload before reapplying
changes, or use [mutate()](mutations.md).

Conditional writes compare the object's ETag with the record in the selected
database, without restricting which `Database` object read the ETag. See
[Instance State](database.md#instance-state).

## Deletion

Delete by instance or primary key. Direct deletion does not require a prior
fetch. Index cleanup uses stored values, including when the supplied instance
is stale.

```python
deleted = Assignment.delete(user.key, task.key)  # True if the edge existed.
deleted_again = assignment.delete()             # False after removal.
user_deleted = User.delete(user.key)
```

| Mode | Behavior |
| --- | --- |
| `ALWAYS` | Delete if present. Return `True` if deleted, otherwise `False`. Default. |
| `IF_NOT_MODIFIED` | Delete if present with a matching ETag. Return `True`; otherwise raise `ConflictError`. |

```python
task = Task.get(task_key)

deleted = task.delete(mode=IF_NOT_MODIFIED)
```

Conditional deletion atomically compares the ETag and removes the record and
index entries. It requires an instance carrying an ETag. Model-and-key calls
with `IF_NOT_MODIFIED` raise `ValueError`. Deletion with `IF_MISSING` or
`IF_EXISTS` also raises `ValueError`.

Batch deletion checks conditions against the initial stored state. Its result
reflects execution order and may be `False` after an earlier deletion in the
batch.

After a confirmed instance deletion, keep its key and field values, clear
`etag` to `None`, and associate it with the selected database. This also
applies when `ALWAYS` returns `False` because the record is absent.

```python
deleted = task.delete()
assert task.etag is None
```

An ordinary `set()` can recreate the record. `IF_NOT_MODIFIED` requires an
ETag again. A failed condition or unconfirmed storage outcome leaves local
state unchanged.

Model-and-key deletion does not update loaded instances. There is no identity
map; those instances retain their previous fields and ETags. Batch operations
follow the same [instance state rules](commits.md#capture-and-object-state).

Deletion does not cascade. Node deletion preserves incoming and outgoing edges;
edge deletion preserves endpoint nodes. Only the deleted record's index entries
are removed. Applications remove related records explicitly; edges may
reference missing nodes.
