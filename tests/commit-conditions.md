# Commit Conditions and Instance State

Source: [Commits](../docs/commits.md).

## Setup

    >>> from pytest import raises
    >>> from comfychair import (Database, Key, Node, Commit, CommitError,
    ...     ConflictError, IF_MISSING, IF_NOT_MODIFIED)
    >>> db = Database()
    >>> scope = 'conditions_' + str(Key())
    >>> class Task(Node):
    ...     namespace = scope
    ...     title: str
    >>> db.bind(Task)

## Add Twice

    >>> task = Task(title='New')
    >>> with Commit() as batch:
    ...     first = batch.add(task)
    ...     second = batch.add(task)
    >>> first.value
    True
    >>> second.value
    False

Add returning False does not cancel other operations. To gate the batch on
absence, use set(mode=IF_MISSING).

## Check Initial Absence Twice

    >>> first = Task(title='First')
    >>> second = Task(key=first.key, title='Second')
    >>> with Commit() as batch:
    ...     _ = batch.set(first, mode=IF_MISSING)
    ...     _ = batch.set(second, mode=IF_MISSING)
    >>> Task.get(first.key).title
    'Second'

All mode conditions use storage immediately before batch writes, not context
entry. ALWAYS has no condition; IF_MISSING requires absence; IF_EXISTS requires
presence; IF_NOT_MODIFIED requires presence and a matching ETag.

## Check Conditions at Execution

The key is absent when the operation is queued. An independent write before
context exit makes the condition fail. The immediate write remains; neither
queued write takes effect.

    >>> queued = Task(title='Queued')
    >>> extra = Task(title='Also queued')
    >>> with raises(ConflictError):
    ...     with Commit() as batch:
    ...         pending = batch.set(extra)
    ...         _ = batch.set(queued, mode=IF_MISSING)
    ...         Task(key=queued.key, title='Immediate').set()
    >>> Task.get(queued.key).title
    'Immediate'
    >>> Task.exists(extra.key)
    False
    >>> queued.etag is None and extra.etag is None
    True
    >>> with raises(CommitError):
    ...     _ = pending.value

## Set Then Delete With the Same ETag

    >>> task = Task(title='Before')
    >>> task.set()
    >>> with Commit() as batch:
    ...     task.title = 'After'
    ...     _ = batch.set(task, mode=IF_NOT_MODIFIED)
    ...     removed = batch.delete(task, mode=IF_NOT_MODIFIED)
    >>> removed.value
    True
    >>> Task.exists(task.key)
    False
    >>> task.etag is None
    True

Checks and writes form one atomic Redis operation. After all checks pass,
execute in call order without rechecking conditions. Only explicitly supplied
conditions protect reads; other reads and query results are unprotected.

## Delete Twice With the Same ETag

    >>> task = Task(title='Twice')
    >>> task.set()
    >>> with Commit() as batch:
    ...     first = batch.delete(task, mode=IF_NOT_MODIFIED)
    ...     second = batch.delete(task, mode=IF_NOT_MODIFIED)
    >>> first.value
    True
    >>> second.value
    False
    >>> task.etag is None
    True

## Delete Then Recreate

    >>> task = Task(title='Recreated')
    >>> task.set()
    >>> before = task.etag
    >>> with Commit() as batch:
    ...     _ = batch.delete(task)
    ...     _ = batch.set(task, mode=IF_NOT_MODIFIED)
    >>> Task.get(task.key).title
    'Recreated'
    >>> task.etag == before
    False
    >>> task.etag == Task.get(task.key).etag
    True

## Keep the Last Written ETag

    >>> task = Task(title='Twice')
    >>> with Commit() as batch:
    ...     _ = batch.set(task)
    ...     _ = batch.set(task)
    >>> task.etag == Task.get(task.key).etag
    True

Confirmed local updates follow execution order. Insert/replacement assigns its
new ETag. Instance delete clears it even when returning False. Writes do not
return replacement objects. Local updates do not affect initial-state checks.

## Reject the Entire Batch on Conflict

    >>> task = Task(title='Original')
    >>> task.set()
    >>> stale = Task.get(task.key)
    >>> stale_etag = stale.etag
    >>> task.set()
    >>> extra = Task(title='Must not be written')
    >>> with raises(ConflictError):
    ...     with Commit() as batch:
    ...         pending = batch.set(extra)
    ...         _ = batch.set(stale, mode=IF_NOT_MODIFIED)
    >>> Task.exists(extra.key)
    False
    >>> extra.etag is None
    True
    >>> stale.etag == stale_etag
    True
    >>> with raises(CommitError):
    ...     _ = pending.value

A failed condition precedes every write, including unconditional writes.
Failed conditions and unknown outcomes preserve local ETags and associations.

## Leave Other Instances Alone

    >>> task = Task(title='Original')
    >>> task.set()
    >>> other = Task.get(task.key)
    >>> other_etag = other.etag
    >>> with Commit() as batch:
    ...     task.title = 'Updated'
    ...     _ = batch.set(task)
    >>> other.title
    'Original'
    >>> other.etag == other_etag
    True

## Do Not Apply Key Deletion to an Instance

    >>> with Commit() as batch:
    ...     _ = batch.set(task)
    ...     _ = batch.delete(Task, task.key)
    >>> Task.exists(task.key)
    False
    >>> task.etag is None
    False

Only instances passed to operations receive local updates. Separate instances
for the same identity are never synchronized. Set accepts all four modes;
delete accepts ALWAYS/IF_NOT_MODIFIED, defaulting to ALWAYS. Conditional delete
requires an instance carrying an ETag, as for immediate deletion.

## Cleanup

    >>> for record in Task.iterate(batch_size=100):
    ...     _ = record.delete()
    >>> db.close()
