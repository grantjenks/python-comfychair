# Retrying Commits

Source: [Commits](../docs/commits.md).

## Setup

    >>> from pytest import raises
    >>> from comfychair import Database, Key, Node, Commit, CommitError, ConflictError, IF_NOT_MODIFIED
    >>> db = Database()
    >>> scope = 'retry_' + str(Key())
    >>> class Task(Node):
    ...     namespace = scope
    ...     title: str
    >>> db.bind(Task)

## Construct Without Executing

    >>> calls = []
    >>> def operation(batch):
    ...     calls.append(batch)
    ...     return 'done'
    >>> commit = Commit(operation)
    >>> calls
    []

## Return the Callable's Result

    >>> commit.retry()
    'done'
    >>> len(calls)
    1

Each attempt creates an active batch, calls the operation, then captures and
commits after it returns. The successful callable's result is returned unchanged.

## Return a Resolved Handle

    >>> task = Task(title='Remove')
    >>> task.set()
    >>> def remove(batch):
    ...     current = Task.get(task.key)
    ...     return batch.delete(current, mode=IF_NOT_MODIFIED)
    >>> removed = Commit(remove).retry(attempts=3)
    >>> removed.value
    True

## Retry an Application Conflict

    >>> batches, handles = [], []
    >>> task = Task(title='Reviewed')
    >>> def save(batch):
    ...     batches.append(batch)
    ...     handle = batch.set(task)
    ...     handles.append(handle)
    ...     if len(batches) == 1:
    ...         raise ConflictError('retry')
    ...     return handle
    >>> result = Commit(save).retry(attempts=3)
    >>> len(batches)
    2
    >>> batches[0] is batches[1]
    False
    >>> result is handles[1]
    True
    >>> result.value is None
    True

## Keep Failed Handles Unresolved

    >>> with raises(CommitError):
    ...     _ = handles[0].value

Conflicts from the callable, an invoked operation, or commit execution all retry.
No provenance distinction is made. Failed attempts discard queued work and
return values; their handles cannot acquire results from later attempts.

## Reload Inside the Callable

The first attempt deliberately conflicts through a separate real connection.

    >>> competitor = Database()
    >>> task = Task(title='Original')
    >>> task.set()
    >>> seen = []
    >>> def append_review(batch):
    ...     current = Task.get(task.key, using=db)
    ...     seen.append(current.title)
    ...     if len(seen) == 1:
    ...         concurrent = Task.get(task.key, using=competitor)
    ...         concurrent.title = 'Concurrent'
    ...         concurrent.set()
    ...     current.title += ' reviewed'
    ...     _ = batch.set(current, mode=IF_NOT_MODIFIED)
    ...     return current.key
    >>> Commit(append_review, using=db).retry() == task.key
    True
    >>> seen
    ['Original', 'Concurrent']
    >>> Task.get(task.key).title
    'Concurrent reviewed'

Put repeated reads, decisions, and construction inside the callable. Retry does
not copy arguments, closures, or queued objects, or refresh closed-over records.
Explicit using routes the batch only; ordinary calls choose their own database.

## Raise the Last Conflict on Exhaustion

    >>> failures = []
    >>> def fail(batch):
    ...     error = ConflictError(str(len(failures)))
    ...     failures.append(error)
    ...     raise error
    >>> with raises(ConflictError) as caught:
    ...     Commit(fail).retry(attempts=2)
    >>> len(failures)
    2
    >>> caught.value is failures[-1]
    True

Attempts is a positive integer counting total executions, default three.
Retries are immediate without backoff. The context-manager form never retries.

## Do Not Retry Other Exceptions

    >>> calls = []
    >>> def fail_once(batch):
    ...     calls.append('called')
    ...     raise RuntimeError('stop')
    >>> with raises(RuntimeError):
    ...     Commit(fail_once).retry(attempts=3)
    >>> calls
    ['called']

Validation, KeyError, connection failures, and uncertain outcomes also propagate
without retry. No Python edits, independent writes, or external side effects
are undone. Perform external effects after success. Queue on the supplied
batch; nesting is forbidden. Pending .value access inside the callable raises
CommitError.

## Cleanup

    >>> for record in Task.iterate(batch_size=100):
    ...     _ = record.delete()
    >>> db.close()
    >>> competitor.close()
