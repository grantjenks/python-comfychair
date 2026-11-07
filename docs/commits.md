# Commits

[Documentation](../README.md#documentation) · [Example models](example.md)

## Commit

`with Commit() as batch:` queues writes for one execution at normal context
exit. Commit checks mode conditions and applies writes atomically, subject to
[execution limits](#limits). Each write maintains its record and indexes
together.

Only batch methods participate. Model and instance methods, including `set()`,
`delete()`, and `mutate()`, execute independently inside the block. Pass
`batch` to helpers that queue writes.

`batch.set(record)`, `batch.add(record)`, and `batch.delete(record)` queue
operations. Set accepts all four [write modes](storage.md#write-modes); delete
accepts `ALWAYS` and `IF_NOT_MODIFIED` with the same ETag requirements as
immediate deletion. Set and delete default to `ALWAYS`.

Deletion also accepts a model and primary key without a prior fetch:

```python
with Commit() as batch:
    removed = batch.delete(Assignment, user.key, task.key)

if removed.value:
    print("Assignment removed")
```

Normal exit commits. An exception escaping the block before commit discards
queued operations. `Commit()` makes one attempt;
[the callable form](#retrying-a-callable) supports retries.

## Database Selection

`Commit(using=database)` directs every queued operation to one database.
Without `using`, resolve records through their database associations, model
bindings, then the implicit localhost default. Conflicting selections reject
the batch before writing. Unconfigured models share the implicit default.
Empty batches are no-ops.
Batch methods do not accept `using`. Ordinary calls inside the block retain
their own database selection. See [Commit Routing](database.md#commit-routing).

## Result

Each queued operation returns a `Result[T]`. Its `.value` is available only
after the whole commit succeeds and is confirmed.

| Method | Handle | Value |
| --- | --- | --- |
| `batch.set(record)` | `Result[None]` | `None`. |
| `batch.add(record)` | `Result[bool]` | `True` if inserted; `False` if the key exists at this operation. |
| `batch.delete(...)` | `Result[bool]` | `True` if deleted; `False` if absent at this operation. Conditional deletion requires an initial ETag match. |

Handles remain attached to their operations and may be returned from helpers:

```python
def remove_assignment(batch, user_key, task_key):
    return batch.delete(Assignment, user_key, task_key)


with Commit() as batch:
    saved = batch.set(task, mode=IF_NOT_MODIFIED)
    removed = remove_assignment(batch, user.key, task.key)

saved.value    # None
removed.value  # True if deleted, False if absent
```

Reading `.value` before confirmation raises `CommitError`; it does not wait or
execute operations. Discarded batches, failed commits, and unconfirmed outcomes
leave handles unresolved. The error reports that no confirmed result is
available, not that no writes occurred. The original failure propagates from
the context manager.

Converting a handle to `bool` raises `TypeError`. Use `removed.value`. Ignoring
handles is permitted.

## Capture and Object State

Queueing retains the operation, mode, and object reference. Normal context exit
captures current identities, fields, and ETags, then validates and serializes
all captured values before writing.

```python
with Commit() as batch:
    task.title = "First"
    batch.set(task)
    task.title = "Second"

# Stores "Second" and updates task.etag.
```

Repeated sets of one object all use its values at context exit. Model-and-key
deletes retain the supplied identity arguments.

Confirmed writes update object ETags in place. Repeated successful writes
through one object leave its ETag at the last such write. Writes do not return
replacement objects. Discarding a batch does not undo Python edits.

## Conditions Apply to the Whole Batch

Conditional batches check previously read versions with `IF_NOT_MODIFIED`:

```python
task = Task.get(task_key)

task.title = "Completed review"

with Commit() as batch:
    batch.set(task, mode=IF_NOT_MODIFIED)
    batch.delete(Assignment, user_key, task_key)
```

All mode conditions are checked against stored state immediately before batch
writes, not at context entry. Checks include ETags for conditional sets and
deletes, absence for `IF_MISSING`, and presence for `IF_EXISTS`.

A failed condition raises `ConflictError` before any batch write, including
unconditional writes. Condition checking and execution form one atomic Redis
operation; other clients cannot change checked records between them.

After checks pass, operations execute in call order. Results reflect preceding
operations; conditions are not rechecked.

| Operations on one key | Behavior |
| --- | --- |
| Two adds; initially absent | `True`, then `False`. |
| Set, then delete | Write, then remove. |
| Conditional set and delete with the same matching ETag | Both checks pass. The new ETag does not invalidate deletion. |
| Two sets with `IF_MISSING`; initially absent | Both checks pass. The second set replaces the first. |
| Two conditional deletes with the same matching ETag | Both checks pass. `True`, then `False`. |
| Delete, then conditional set with an initially matching ETag | Delete, then recreate. |

```python
task = Task.get(task_key)

with Commit() as batch:
    first = batch.delete(task, mode=IF_NOT_MODIFIED)
    second = batch.delete(task, mode=IF_NOT_MODIFIED)

assert first.value is True
assert second.value is False
```

`batch.add()` returns `False` if the key exists when it executes, without
cancelling the batch. Use `batch.set(..., mode=IF_MISSING)` to require initial
absence for the entire batch.

Reads execute immediately before or inside the block. They are not snapshots
and do not see queued writes; there is no local read-your-writes cache. Only
versions supplied through `IF_NOT_MODIFIED` are checked. Other reads and query
results are unprotected.

## Retrying a Callable

`Commit(operation).retry(attempts=3)` retries an operation on `ConflictError`.
Construction does not execute the callable. Each attempt creates a fresh batch,
calls `operation(batch)`, then captures and commits writes after it returns.

```python
def mark_reviewed(batch):
    task = Task.get(task_key)
    task.title = "Reviewed"
    batch.set(task, mode=IF_NOT_MODIFIED)
    return task.key


reviewed_key = Commit(mark_reviewed).retry(attempts=3)
```

Put reads, decisions, and object construction inside the callable when they
must repeat. Closed-over objects retain their existing state; retry does not
refresh them. Queue writes on the supplied batch without opening another
commit.

A `ConflictError` from the callable or commit discards that attempt's queued
work and return value, then retries with a fresh batch. `attempts` is a
positive integer counting total executions, including the first. Default:
three. Retries are immediate, without backoff.

Exhaustion raises the last `ConflictError`. Other exceptions propagate without
retry, including validation errors, `KeyError`, and connection failures with
uncertain outcomes.

Return the callable's result unchanged after confirmed commit. The result may
be a `Result` handle:

```python
def remove_assignment(batch):
    assignment = Assignment.get(user_key, task_key)
    return batch.delete(assignment, mode=IF_NOT_MODIFIED)


removed = Commit(remove_assignment).retry(attempts=3)
deleted = removed.value
```

A returned handle is resolved when `retry()` returns. Reading `.value` inside
the callable raises `CommitError`. Handles from failed attempts never acquire
results from later attempts.

Callables may execute repeatedly. Perform external side effects after success.
Immediate writes inside the callable remain independent; failed batches do not
undo them. Retrying does not undo changes to Python objects outside the batch.

The callable form also accepts an explicit database:

```python
def update_user(batch):
    user = User.get(user_key, using=archive)
    user.name = "Alice"
    batch.set(user, mode=IF_NOT_MODIFIED)


Commit(update_user, using=archive).retry(attempts=3)
```

Selecting a batch database does not change the database used by ordinary model
calls in the callable.

## Limits

Nested commits and savepoints are not supported. Entering a commit or starting
a retrying commit while a batch is active raises `CommitError`. The callable
form runs with its supplied batch active.

Atomic multi-key operations require one Redis primary or one Redis Cluster hash
slot. Cross-shard transactions are not supported.

Mode conflicts precede writes. Arbitrary execution errors do not imply rollback
or absence of writes; Redis provides no general rollback. Individual record
operations must still maintain their indexes.

Connection loss may leave commit outcome unknown. Persistence and failover
durability depend on the Redis deployment.

Execution constraints:
[Redis transactions](https://redis.io/docs/latest/develop/using-commands/transactions/),
[Redis scripting](https://redis.io/docs/latest/develop/programmability/eval-intro/),
and
[Redis Cluster specification](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/).
