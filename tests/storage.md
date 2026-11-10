# Store and Retrieve

Source: [Store and Retrieve](../docs/storage.md).

## Setup

    >>> from pytest import raises
    >>> from comfychair import Database, Key, Node
    >>> db = Database()
    >>> scope = 'storage_' + str(Key())
    >>> class Task(Node):
    ...     namespace = scope
    ...     title: str
    >>> db.bind(Task)

## Construct Without Saving

    >>> task = Task(title='First')
    >>> Task.exists(task.key)
    False

## Save a Record

    >>> task = Task(title='First')
    >>> task.set() is None
    True
    >>> Task.get(task.key).title
    'First'

## Save an Edit

    >>> task.title = 'Second'
    >>> Task.get(task.key).title
    'First'
    >>> task.set()
    >>> Task.get(task.key).title
    'Second'

Model and instance operations execute immediately. Construction and assignment
perform no I/O. All storage methods accept keyword-only using; see
[Database Selection](database.md).

## Raise for a Missing Record

    >>> missing_key = Key()
    >>> with raises(KeyError):
    ...     Task.get(missing_key)

## Return a Supplied Default

    >>> Task.get(missing_key, default=None) is None
    True
    >>> sentinel = object()
    >>> Task.get(missing_key, default=sentinel) is sentinel
    True

The same rule applies to first() and last(). Defaults handle absence only;
query, connection, validation, and decoding errors still propagate.

## Check Existence

    >>> Task.exists(task.key)
    True
    >>> Task.exists(missing_key)
    False

Exists accepts the same identity arguments as get. It neither decodes a record
nor constructs a model. Reads reserve no records or locks for later writes.

## Refresh in Place

    >>> task.title = 'Unsaved'
    >>> same = task
    >>> task.refresh() is None
    True
    >>> task is same
    True
    >>> task.title
    'Second'

Refresh loads one replacement normally, then copies its fields and ETag.
Related nodes are not loaded. Any loading failure leaves the original intact.

## Preserve Local State When Refresh Fails

    >>> task = Task(title='Local')
    >>> with raises(KeyError):
    ...     task.refresh()
    >>> task.title
    'Local'
    >>> task.etag is None
    True

## Add Only When Absent

    >>> task = Task(title='First')
    >>> task.add()
    True
    >>> task.title = 'Second'
    >>> task.add()
    False
    >>> Task.get(task.key).title
    'First'

## Create or Retrieve

    >>> key = Key()
    >>> task, created = Task.get_or_create(key, defaults={'title': 'First'})
    >>> created
    True
    >>> task.title
    'First'

## Preserve an Existing Record

    >>> etag = task.etag
    >>> again, created = Task.get_or_create(key, defaults={'title': 'Ignored'})
    >>> created
    False
    >>> again.title
    'First'
    >>> again.etag == etag
    True

Get-or-create uses primary identity, including both endpoints for edges.
Defaults supply creation fields, with model defaults/factories for the rest;
they cannot override identity. None is not a primary key. Existing records are
unchanged. Creation validates before insertion. Concurrent callers cannot
create duplicates. It executes immediately and is not a batch method.

## Cleanup

    >>> for record in Task.iterate(batch_size=100):
    ...     _ = record.delete()
    >>> db.close()
