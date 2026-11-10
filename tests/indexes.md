# Ordered Queries

Source: [Ordered Indexes](../docs/indexes.md).

## Setup

    >>> from pytest import raises
    >>> from comfychair import Database, Edge, Field, Key, Node, Ordering, UnsupportedQueryError
    >>> db = Database()
    >>> scope = 'indexes_' + str(Key())
    >>> class Task(Node):
    ...     namespace = scope
    ...     key: str
    ...     title: str
    ...     priority: int = Field(default=0)
    ...     status: str = Field(default='open')
    ...     indexes = (Ordering(priority), Ordering(status, priority))
    >>> db.bind(Task)
    >>> Task(key='c', title='Third', priority=3).set()
    >>> Task(key='a', title='First', priority=1).set()
    >>> Task(key='b', title='Second', priority=2, status='closed').set()

## Traverse Primary Keys Automatically

    >>> [task.key for task in Task.page(limit=20)]
    ['a', 'b', 'c']
    >>> [task.key for task in Task.page(order_by=Task.key, limit=20)]
    ['a', 'b', 'c']

Primary indexes include all records of the model and namespace and need no
declaration. Writes and deletes maintain them with records. Other orderings
must be declared and selected explicitly; there is no model default ordering.

## Reverse an Ordering

    >>> [task.priority for task in Task.page(order_by=Task.priority, reverse=True, limit=20)]
    [3, 2, 1]

## Select an Equality Prefix

    >>> page = Task.page(Task.status == 'open', order_by=Task.priority, limit=20)
    >>> [task.key for task in page]
    ['a', 'c']

## Bound the Next Field

    >>> page = Task.page(
    ...     Task.status == 'open', Task.priority >= 1, Task.priority < 3,
    ...     order_by=Task.priority, limit=20,
    ... )
    >>> [task.key for task in page]
    ['a']

Supported operators are ==, >, >=, <, and <=. Equalities constrain leading
index fields; optional lower and upper bounds constrain the next ordered field.
Either bound may be inclusive or exclusive. Positional predicates mean AND.

## Ignore Predicate Argument Order

    >>> page = Task.page(
    ...     Task.priority < 3, Task.priority >= 1, Task.status == 'open',
    ...     order_by=Task.priority, limit=20,
    ... )
    >>> [task.key for task in page]
    ['a']

## Break Ties by Primary Key

    >>> Task(key='d', title='Another third', priority=3).set()
    >>> page = Task.page(Task.priority == 3, order_by=Task.priority, limit=20)
    >>> [task.key for task in page]
    ['c', 'd']

## Read Either End

    >>> Task.first(order_by=Task.priority).key
    'a'
    >>> Task.last(order_by=Task.priority).key
    'd'
    >>> Task.first(order_by=Task.priority, reverse=True).key
    'd'
    >>> Task.last(order_by=Task.priority, reverse=True).key
    'a'

First and last use the requested order, direction, and primary-key tiebreaker.
They access an index end without fetching intervening records. They accept
source, predicates, order_by, reverse, using, and default, but not limit/cursor.

## Count Index Membership

    >>> Task.count()
    4
    >>> Task.count(Task.status == 'closed')
    1
    >>> Task.count(Task.priority >= 2, order_by=Task.priority)
    3

Counts load no records and support directly countable primary/secondary scopes
and ranges. If eligible indexes have different memberships, select order_by.
Unsupported scopes fail without a fallback scan. Counts are live observations.

## Return an Empty Contradictory Range

    >>> low, high = Task.priority >= 10, Task.priority < 5
    >>> page = Task.page(low, high, order_by=Task.priority, limit=20)
    >>> len(page)
    0
    >>> page.cursor is None
    True
    >>> Task.count(low, high, order_by=Task.priority)
    0
    >>> list(Task.iterate(low, high, order_by=Task.priority, batch_size=100))
    []

Index support and operand validity are checked before concluding a query is empty.

## Handle an Empty Endpoint

    >>> with raises(KeyError):
    ...     Task.first(low, high, order_by=Task.priority)
    >>> Task.last(low, high, order_by=Task.priority, default=None) is None
    True
    >>> sentinel = object()
    >>> Task.first(low, high, order_by=Task.priority, default=sentinel) is sentinel
    True

## Reject an Unindexed Query

    >>> with raises(UnsupportedQueryError):
    ...     Task.page(Task.title == 'First', limit=20)

Queries use one supported primary or declared index. No scans, client-side
filtering, index intersections, !=, OR, field comparisons, or arithmetic.

## Reject Predicate Truth Testing

    >>> with raises(TypeError):
    ...     bool(Task.priority >= 1)
    >>> with raises(TypeError):
    ...     1 <= Task.priority < 3

Pass separate predicates rather than Python and/or or chained comparisons.
Comparisons of instance values retain ordinary Python behavior.

## Scope Edges by Source

    >>> class Assignment(Edge[Task, Task]):
    ...     namespace = scope
    ...     priority: int = Field(default=0)
    ...     indexes = (Ordering(priority),)
    >>> db.bind(Assignment)
    >>> Assignment(source='a', target='c', priority=2).set()
    >>> Assignment(source='a', target='b', priority=1).set()
    >>> Assignment(source='d', target='b', priority=9).set()
    >>> [edge.target for edge in Assignment.page('a', limit=20)]
    ['b', 'c']
    >>> Assignment.count('a')
    2
    >>> Assignment.first('a', order_by=Assignment.priority).target
    'b'

Edge primary traversal is by target within a source. Secondary orderings are
source-scoped automatically; omit source from Ordering declarations. Explicit
order_by=Assignment.target selects primary traversal. All query methods accept
keyword-only using and share the [pagination rules](pagination.md).

## Cleanup

    >>> for source in ('a', 'd'):
    ...     for record in Assignment.iterate(source, batch_size=100):
    ...         _ = record.delete()
    >>> for record in Task.iterate(batch_size=100):
    ...     _ = record.delete()
    >>> db.close()
