# Mutations

[Documentation](../README.md#documentation) · [Example models](example.md)

## Mutate

`record.mutate(callables, *, attempts=3, using=None)` applies an iterable of
functions to a node or edge and saves with `IF_NOT_MODIFIED`. Each function
modifies its argument and returns `None`.

The iterable is materialized once and reused across attempts. An empty iterable
raises `ValueError`. Callables are invoked without signature inspection; a
return value other than `None` raises `TypeError` and stops the operation.

```python
def increase_priority(assignment):
    assignment.priority += 1


def reopen(assignment):
    assignment.status = "open"


assignment = Assignment.get(user.key, task.key)
assignment.mutate([increase_priority, reopen], attempts=3)
```

`using` selects the database for the conditional write and retry reads.
Omission uses the instance's database, then the model binding, then the
current default database. Conditional writes compare the working copy's
ETag with the stored record in that database; the reading `Database` object's
identity does not restrict the comparison. See
[Database Configuration](database.md).

The first attempt uses a working copy of the object's current fields and ETag.
Functions run in supplied order, followed by validation, serialization, and
conditional write. On `ConflictError`, fetch the current stored record and
replay the functions on a fresh working copy. Failed attempts do not accumulate
changes.

Confirmed success updates the original object's fields and ETag in place and
returns `None`. Failure leaves the original object unchanged by the operation.
Backend failures raise `StorageError`; connection failure may leave the storage
outcome unknown.

`attempts` is a positive integer counting total attempts, including the first.
Default: three. Retries are immediate, without backoff. Exhaustion raises the
last `ConflictError`. See [Arguments](errors.md#arguments).

Only `ConflictError` triggers retry, including when raised by a callback. Other
callback, validation, and connection errors propagate. A missing record during
refresh raises `KeyError`. An initial object without an ETag raises
`ValueError`.

Unsaved edits participate only in the first attempt. Express changes in
callbacks to reapply them after conflicts. Callbacks may run repeatedly;
perform external side effects only after success.

`mutate()` executes independently, including inside a commit block. There is no
`batch.mutate()`. Use
[Commit(operation).retry()](commits.md#retrying-a-callable) for coordinated
reads, decisions, and writes.
