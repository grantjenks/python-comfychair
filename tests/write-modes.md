# Write Modes and ETags

Source: [Store and Retrieve](../docs/storage.md).

## Setup

    >>> from pytest import raises
    >>> from comfychair import (Database, Key, Node, Mode, ALWAYS,
    ...     IF_MISSING, IF_EXISTS, IF_NOT_MODIFIED, ConflictError)
    >>> db = Database()
    >>> scope = 'modes_' + str(Key())
    >>> class Task(Node):
    ...     namespace = scope
    ...     title: str
    >>> db.bind(Task)

## Use an Enum or Constant

    >>> IF_NOT_MODIFIED is Mode.IF_NOT_MODIFIED
    True

## Start Without a Version

    >>> task = Task(title='First')
    >>> task.etag is None
    True
    >>> with raises(ValueError):
    ...     task.set(mode=IF_NOT_MODIFIED)

ETags are opaque random version tokens. Primary keys do not carry ETags.

## Require Absence

    >>> task = Task(title='First')
    >>> task.set(mode=IF_MISSING)
    >>> with raises(ConflictError):
    ...     task.set(mode=IF_MISSING)
    >>> Task.get(task.key).title
    'First'

## Require Presence

    >>> task = Task(title='First')
    >>> with raises(ConflictError):
    ...     task.set(mode=IF_EXISTS)
    >>> task.set()
    >>> task.title = 'Second'
    >>> task.set(mode=IF_EXISTS)
    >>> Task.get(task.key).title
    'Second'

IF_EXISTS ignores the object's ETag. ALWAYS, the default, inserts or replaces
without a condition. Every mode validates before writing.

## Assign Without Changing the ETag

    >>> task = Task(title='First')
    >>> task.set()
    >>> etag = task.etag
    >>> task.title = 'Second'
    >>> task.etag == etag
    True

## Replace With a Matching Version

    >>> task.set(mode=IF_NOT_MODIFIED)
    >>> task.etag == etag
    False
    >>> task.etag == Task.get(task.key).etag
    True

## Reject a Stale Version

    >>> stale = Task.get(task.key)
    >>> stale_etag = stale.etag
    >>> task.title = 'Third'
    >>> task.set()
    >>> stale.title = 'Lost update'
    >>> with raises(ConflictError):
    ...     stale.set(mode=IF_NOT_MODIFIED)
    >>> Task.get(task.key).title
    'Third'
    >>> stale.etag == stale_etag
    True

Comparison and record/index updates are atomic. A failed condition changes
neither storage nor indexes. Set and delete do not retry conflicts.

## Version an Unchanged Write

    >>> etag = task.etag
    >>> task.set()
    >>> task.etag == etag
    False

Every insert or replacement gets a new ETag, even with unchanged values.
Restoring old field values does not restore their earlier ETag.

## Reject a Deleted Version

    >>> stale = Task.get(task.key)
    >>> Task.delete(task.key)
    True
    >>> with raises(ConflictError):
    ...     stale.set(mode=IF_NOT_MODIFIED)

Conditions compare the token in the selected database, regardless of which
Database object read it. In a commit, all conditions use the initial stored
state; fields and ETags are captured at context exit. See
[Commit Conditions](commit-conditions.md).

## Cleanup

    >>> for record in Task.iterate(batch_size=100):
    ...     _ = record.delete()
    >>> db.close()
