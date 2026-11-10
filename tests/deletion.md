# Deletion

Source: [Store and Retrieve](../docs/storage.md).

## Setup

    >>> from pytest import raises
    >>> from comfychair import (Database, Edge, Field, Key, Node, Ordering,
    ...     IF_MISSING, IF_EXISTS, IF_NOT_MODIFIED, ConflictError)
    >>> db = Database()
    >>> scope = 'delete_' + str(Key())
    >>> class Task(Node):
    ...     namespace = scope
    ...     title: str
    ...     priority: int = Field(default=0)
    ...     indexes = (Ordering(priority),)
    >>> class Link(Edge[Task, Task]):
    ...     namespace = scope
    >>> db.bind(Task, Link)

## Return Whether Anything Was Deleted

    >>> task = Task(title='First')
    >>> task.set()
    >>> task.delete()
    True
    >>> task.delete()
    False

## Retain Fields and Clear the Version

    >>> task.title
    'First'
    >>> task.etag is None
    True
    >>> task.set()
    >>> Task.exists(task.key)
    True

Confirmed instance deletion retains identity and fields, clears the ETag, and
associates the selected database, even when returning False. Conditional saving
requires an ETag again. Failed conditions or unknown outcomes preserve local state.

## Delete by Key Without Updating Other Instances

    >>> task = Task(title='Direct')
    >>> task.set()
    >>> etag = task.etag
    >>> Task.delete(task.key)
    True
    >>> task.etag == etag
    True
    >>> task.delete()
    False
    >>> task.etag is None
    True

There is no identity map. Model-and-key deletion does not synchronize loaded
objects, including objects used elsewhere in the same batch.

## Require a Matching Version

    >>> task = Task(title='Conditional')
    >>> task.set()
    >>> stale = Task.get(task.key)
    >>> task.set()
    >>> with raises(ConflictError):
    ...     stale.delete(mode=IF_NOT_MODIFIED)
    >>> task.delete(mode=IF_NOT_MODIFIED)
    True

## Reject Unsupported Delete Modes

    >>> with raises(ValueError):
    ...     Task.delete(task.key, mode=IF_NOT_MODIFIED)
    >>> with raises(ValueError):
    ...     task.delete(mode=IF_MISSING)
    >>> with raises(ValueError):
    ...     task.delete(mode=IF_EXISTS)

Delete defaults to ALWAYS. Conditional deletion requires an instance with an
ETag. A missing or stale stored record conflicts. A batch can return False for
a conditional delete after an earlier delete has removed the same record.

## Clean Indexes Using Stored Values

    >>> task = Task(title='Indexed', priority=1)
    >>> task.set()
    >>> stale = Task.get(task.key)
    >>> task.priority = 2
    >>> task.set()
    >>> stale.delete()
    True
    >>> Task.count(Task.priority == 2, order_by=Task.priority)
    0

## Keep Edges When Deleting Nodes

    >>> source = Task(title='Source')
    >>> target = Task(title='Target')
    >>> source.set()
    >>> target.set()
    >>> link = Link(source=source, target=target)
    >>> link.set()
    >>> source.delete()
    True
    >>> Link.exists(source.key, target.key)
    True

## Keep Nodes When Deleting Edges

    >>> Link.delete(source.key, target.key)
    True
    >>> Task.exists(target.key)
    True

Deletion never cascades. Only the deleted record's indexes are removed.
Incoming and outgoing edges may reference missing nodes.

## Cleanup

    >>> _ = Link.delete(source.key, target.key)
    >>> for record in Task.iterate(batch_size=100):
    ...     _ = record.delete()
    >>> db.close()
