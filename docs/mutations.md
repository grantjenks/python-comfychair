# Retrying mutations

[README](../README.md) · [Documentation](../README.md#documentation)

Examples use the models defined in the [introductory example](example.md).

`record.mutate(callables, *, attempts=3)` immediately applies a list of
functions to one node or edge and saves with `IF_NOT_MODIFIED`. Each function
modifies its supplied record and returns `None`:

```python
def increase_priority(assignment):
    assignment.priority += 1


def reopen(assignment):
    assignment.status = "open"


assignment = Assignment.get(user.key, task.key)
assignment.mutate([increase_priority, reopen], attempts=3)
```

The first attempt starts with a working copy of the object's current fields
and ETag. Functions run in list order, followed by validation, serialization,
and a conditional write. On `ConflictError`, Comfy Chair fetches the latest
stored version and replays the functions on a fresh working copy. Failed
attempts never accumulate changes.

On confirmed success, the original object's fields and ETag are updated in
place and `mutate()` returns `None`. If it fails, the original object is left
unchanged by the operation. As with other writes, a connection failure can
leave the storage outcome unknown.

`attempts` is a positive integer counting total attempts, including the first;
the default is three. Retries are immediate, without backoff. Exhaustion raises
the last `ConflictError`. Other exceptions propagate without retrying, including
callback exceptions other than `ConflictError`, validation and connection
errors, and `KeyError` when a refresh finds the record missing. An initial
object without an ETag raises `ValueError`.

Edits made before `mutate()` participate in the first attempt, but only changes
expressed by the functions are reapplied after a conflict. Put every change
that must survive retries inside those functions. Functions may execute
multiple times; keep external side effects outside them and perform those
effects only after success.

`mutate()` is a standalone operation. There is no `batch.mutate()`, and calling
`record.mutate()` inside a commit block executes independently of that batch.
Use [a retrying commit](commits.md#retrying-a-callable) to rerun coordinated reads,
decisions, and writes.
