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

    >>> present = Task(title='Present')
    >>> present.set()
    >>> Task.exists(present.key)
    True
    >>> Task.exists(Key())
    False

Exists accepts the same identity arguments as get. It neither decodes a record
nor constructs a model. Reads reserve no records or locks for later writes.

## Refresh in Place

    >>> task = Task(title='Saved')
    >>> task.set()
    >>> task.title = 'Unsaved'
    >>> same = task
    >>> task.refresh() is None
    True
    >>> task is same
    True
    >>> task.title
    'Saved'

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

## Create From Competing Requests

Two connections request the same key with different defaults. Either may win;
both must receive the stored winner. The barrier starts competing requests but
does not force a particular server interleaving.

    >>> from concurrent.futures import ThreadPoolExecutor
    >>> from threading import Barrier
    >>> clients = [Database(), Database()]
    >>> ready = Barrier(2)
    >>> key = Key()
    >>> def request(index):
    ...     ready.wait(timeout=10)
    ...     return Task.get_or_create(
    ...         key, defaults={'title': ('First request', 'Second request')[index]},
    ...         using=clients[index],
    ...     )
    >>> with ThreadPoolExecutor(max_workers=2) as pool:
    ...     outcomes = list(pool.map(request, (0, 1)))
    >>> sorted(created for record, created in outcomes)
    [False, True]
    >>> winner = next(record for record, created in outcomes if created)
    >>> stored = Task.get(key)
    >>> all(record.key == key and record.title == stored.title
    ...     and record.etag == stored.etag for record, created in outcomes)
    True
    >>> stored.title == winner.title
    True
    >>> stored.title in ('First request', 'Second request')
    True
    >>> for client in clients:
    ...     client.close()

## Cleanup

    >>> for record in Task.iterate(batch_size=100):
    ...     _ = record.delete()
    >>> db.close()
