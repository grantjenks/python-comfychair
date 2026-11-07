# Commits

[README](../README.md) · [Documentation](../README.md#documentation)

Examples use the models defined in the [introductory example](example.md).

Each write maintains its record and indexes together. Use `Commit` to collect
related writes in a batch. The `with Commit() as batch:` form executes once.
On successful context exit, it checks all mode conditions and applies the writes
atomically. `Commit(callable).retry()` explicitly retries an entire operation.

Only calls through `batch` participate. Ordinary calls such as `task.set()`,
`User.delete(key)`, and `record.mutate(...)` execute independently, even inside
the context block.
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

## Conditions apply to the whole batch

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
read-your-writes cache. A context-manager commit makes one attempt; retries
require the callable form below.

## Retrying a callable

`Commit(operation).retry(attempts=3)` reruns an entire operation on
`ConflictError`. Constructing `Commit(operation)` does not execute it.
Each attempt creates a fresh batch, invokes `operation(batch)`, then captures
and commits the queued writes after the callable returns:

```python
def mark_reviewed(batch):
    task = Task.get(task_key)
    task.title = "Reviewed"
    batch.set(task, mode=IF_NOT_MODIFIED)
    return task.key


reviewed_key = Commit(mark_reviewed).retry(attempts=3)
```

Keep reads, decisions, and object construction inside the callable when they
must be repeated. Closing over an object fetched before the callable runs
reuses that object's state; the retry mechanism does not refresh it for you.
The callable receives the batch and queues writes on it, without opening an
inner `Commit`.

If the callable or its commit raises `ConflictError`, the attempt's queued
work and return value are discarded and the callable runs again with a fresh
batch. `attempts` is a positive integer counting total executions, including
the first; it defaults to three. Retries are immediate, without backoff.
Exhaustion raises the last `ConflictError`. All other exceptions propagate
without retrying, including validation errors, `KeyError`, and connection
failures with uncertain outcomes.

The callable's return value is delivered unchanged only after its batch
commits successfully. A callable can return a `Result` handle:

```python
def remove_assignment(batch):
    assignment = Assignment.get(user_key, task_key)
    return batch.delete(assignment, mode=IF_NOT_MODIFIED)


removed = Commit(remove_assignment).retry(attempts=3)
deleted = removed.value
```

The returned handle is resolved when `retry()` returns. Its `.value` cannot
be read inside the callable, before the batch commits. Handles from failed
attempts do not resolve to values from a later attempt.

Callables may execute multiple times. Keep external side effects after
`retry()` succeeds. Immediate writes made inside the callable execute
independently and are not undone by a failed batch. Retrying also does not undo
changes to Python objects outside the batch.

## Limits

**Commits cannot be nested.** Entering a commit or starting a retrying commit
while another batch is active raises `CommitError`. A callable runs with its
supplied batch active. There are no savepoints.

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
