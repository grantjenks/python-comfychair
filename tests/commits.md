# Commits

Source: [Commits](../docs/commits.md).

## Setup

    >>> from pytest import raises
    >>> from comfychair import Database, Key, Node, Commit, CommitError, IF_NOT_MODIFIED
    >>> db = Database()
    >>> scope = 'commits_' + str(Key())
    >>> class Task(Node):
    ...     namespace = scope
    ...     title: str
    >>> db.bind(Task)

## Queue a Write

    >>> task = Task(title='Introduction')
    >>> with Commit() as batch:
    ...     saved = batch.set(task)
    >>> Task.get(task.key).title
    'Introduction'

Normal context exit executes once. Empty batches are no-ops. Only batch methods
participate; pass the batch to helpers that queue writes.

## Read Before Execution

    >>> task = Task(title='New')
    >>> with Commit() as batch:
    ...     saved = batch.set(task)
    ...     print(Task.exists(task.key))
    False
    >>> Task.exists(task.key)
    True

Reads are immediate, without snapshots or a local read-your-writes cache.

## Change a Queued Object

    >>> task = Task(title='First')
    >>> with Commit() as batch:
    ...     saved = batch.set(task)
    ...     task.title = 'Second'
    >>> Task.get(task.key).title
    'Second'

Queueing retains operation, mode, and object reference. Context exit captures
current identities, fields, and ETags, then validates and serializes everything
before writes. Repeated sets through one object use its final values.
Model-and-key deletes retain supplied identity arguments. Unsupported modes
are rejected when queued.

## Read the Result

    >>> saved.value is None
    True

Set returns Result[None]. Add and delete return Result[bool], indicating actual
insertion/deletion at the operation's position. Ignoring a handle is permitted.

## Capture the ETag at Exit

An independent save updates the queued object's ETag. The batch uses that final
token, along with its final fields.

    >>> task = Task(title='Original')
    >>> task.set()
    >>> with Commit() as batch:
    ...     saved = batch.set(task, mode=IF_NOT_MODIFIED)
    ...     task.title = 'Immediate'
    ...     task.set()
    ...     immediate_etag = task.etag
    ...     task.title = 'Queued'
    >>> saved.value is None
    True
    >>> Task.get(task.key).title
    'Queued'
    >>> task.etag == Task.get(task.key).etag
    True
    >>> task.etag == immediate_etag
    False

## Reject a Mode When Queued

    >>> with Commit() as batch:
    ...     with raises(ValueError):
    ...         batch.set(task, mode='unsupported')

## Validate All Captured Values Before Writes

    >>> from comfychair import ValidationError
    >>> good = Task(title='Valid')
    >>> bad = Task(title='Initially valid')
    >>> with raises(ValidationError):
    ...     with Commit() as batch:
    ...         _ = batch.set(good)
    ...         _ = batch.set(bad)
    ...         bad.title = 1
    >>> Task.exists(good.key)
    False
    >>> Task.exists(bad.key)
    False

## Reject Early Result Access

    >>> task = Task(title='Pending')
    >>> with Commit() as batch:
    ...     saved = batch.set(task)
    ...     with raises(CommitError):
    ...         _ = saved.value
    >>> saved.value is None
    True

Access neither waits nor executes work. Values resolve only after the entire
commit is confirmed.

## Reject Handle Truth Testing

    >>> with raises(TypeError):
    ...     bool(saved)

Use the resolved .value rather than the handle's truth value.

## Return a Handle From a Helper

    >>> def remove_task(batch, key):
    ...     return batch.delete(Task, key)
    >>> with Commit() as batch:
    ...     removed = remove_task(batch, task.key)
    >>> removed.value
    True

Direct deletion needs no fetch. Edge deletion supplies both endpoint keys:
batch.delete(Assignment, source_key, target_key).

## Discard Queued Work on an Exception

    >>> task = Task(title='Unsaved')
    >>> with raises(RuntimeError):
    ...     with Commit() as batch:
    ...         abandoned = batch.set(task)
    ...         raise RuntimeError('cancel')
    >>> Task.exists(task.key)
    False
    >>> with raises(CommitError):
    ...     _ = abandoned.value

## Keep Python Edits on Discard

    >>> task = Task(title='First')
    >>> task.set()
    >>> with raises(RuntimeError):
    ...     with Commit() as batch:
    ...         task.title = 'Second'
    ...         _ = batch.set(task)
    ...         raise RuntimeError('cancel')
    >>> task.title
    'Second'
    >>> Task.get(task.key).title
    'First'

## Keep Independent Writes on Discard

    >>> independent = Task(title='Immediate')
    >>> with raises(RuntimeError):
    ...     with Commit() as batch:
    ...         independent.set()
    ...         raise RuntimeError('cancel')
    >>> Task.exists(independent.key)
    True

Ordinary set, delete, mutate, and get_or_create execute independently in a block.
Their effects are not undone by batch failure.

## Reject Nesting

    >>> with Commit() as batch:
    ...     with raises(CommitError):
    ...         with Commit():
    ...             pass
    ...     with raises(CommitError):
    ...         Commit(lambda nested: None).retry()

No nested commits or savepoints. The retry callable also runs with an active
batch. See [Retrying Commits](retries.md).

## Select One Database

    >>> with Commit(using=db) as batch:
    ...     _ = batch.set(task)

Without explicit using, resolve each operation through instance association,
model binding, then default. All selections must be the same Database object,
even for equal URLs. Conflicts reject before writing. Unbound, unassociated
models share the current default. Batch methods accept no using argument.
Ordinary calls retain their own routing. Token checks do not require the same
Database object that originally read the ETag.

## Failure Boundaries

Conditions gate the whole batch; see [Commit Conditions](commit-conditions.md).
Each record operation maintains its indexes. Atomic multi-key execution requires
one Redis primary or cluster hash slot; cross-shard transactions are unsupported.

Execution errors do not promise rollback. Backend failures raise StorageError;
connection loss can leave outcome unknown. Failed/discarded/unconfirmed handles
remain unresolved, and reading them raises CommitError rather than implying no
writes occurred. The original failure propagates from context exit. Persistence
and failover durability depend on Redis. See [Guarantees](guarantees.md).

## Cleanup

    >>> for record in Task.iterate(batch_size=100):
    ...     _ = record.delete()
    >>> db.close()
