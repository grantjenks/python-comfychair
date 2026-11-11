# Iteration and Observable Work

Source: [Pages and Cursors](../docs/pagination.md).

## Setup

The observer retains normal constructor behavior. Setup work is excluded from
measurements. Constructor observations concern model loading, not backend scans.

    >>> from pytest import raises
    >>> from comfychair import Database, Key, Node
    >>> db = Database()
    >>> scope = 'iteration_' + str(Key())
    >>> loaded = []
    >>> fail_on = set()
    >>> class Item(Node):
    ...     namespace = scope
    ...     key: str
    ...     def __init__(self, **fields):
    ...         super().__init__(**fields)
    ...         loaded.append(self.key)
    ...         if self.key in fail_on:
    ...             raise RuntimeError('constructor failure')
    >>> db.bind(Item)
    >>> for key in ('a', 'b', 'c'):
    ...     Item(key=key).set()
    >>> loaded.clear()

## Verify the Observer

    >>> item = Item.get('a')
    >>> loaded
    ['a']

## Check Existence Without Loading

    >>> loaded.clear()
    >>> Item.exists('a')
    True
    >>> loaded
    []

## Count Without Loading

    >>> loaded.clear()
    >>> Item.count()
    3
    >>> loaded
    []

## Load Only the Last Record

    >>> loaded.clear()
    >>> item = Item.last()
    >>> loaded
    ['c']

## Do Not Load Lookahead

    >>> loaded.clear()
    >>> page = Item.page(limit=2)
    >>> loaded
    ['a', 'b']

## Defer Loading Until Consumption

    >>> loaded.clear()
    >>> items = Item.iterate(batch_size=2)
    >>> loaded
    []
    >>> next(items).key
    'a'
    >>> loaded
    ['a', 'b']

## Consume the Buffered Page

    >>> next(items).key
    'b'
    >>> loaded
    ['a', 'b']

## Fetch the Next Page on Demand

    >>> next(items).key
    'c'
    >>> loaded
    ['a', 'b', 'c']
    >>> list(items)
    []

Batch size is records per request, not a total limit. Arguments otherwise follow
page: source, predicates, order_by, reverse, using, and optional initial cursor.
Validate batch_size when iterate is called, before I/O. Follow cursors through
empty pages until None. Empty queries yield nothing.

## Start at a Cursor

    >>> page = Item.page(limit=1)
    >>> [item.key for item in Item.iterate(batch_size=1, cursor=page.cursor)]
    ['b', 'c']

The iterator retains at most one page and releases references to previous pages.
It neither prefetches in the background nor counts/materializes all results.
Stopping consumption causes no further requests. The current page may already
be buffered. Ordering and live-data behavior match page().

## Refresh Through One Replacement

    >>> item = Item.get('a')
    >>> loaded.clear()
    >>> item.refresh()
    >>> loaded
    ['a']

## Fail a Whole Page

    >>> fail_on.add('b')
    >>> with raises(RuntimeError):
    ...     Item.page(limit=3)

A page returns only after all chosen records decode and construct. Fetching,
decoding, or construction failure returns no partial page and skips no corrupt
record. Concurrently deleted records may shorten a page; corruption is not absence.

## Fail at the Next Page Request

    >>> items = Item.iterate(batch_size=1)
    >>> next(items).key
    'a'
    >>> with raises(RuntimeError):
    ...     next(items)
    >>> with raises(StopIteration):
    ...     next(items)
    >>> fail_on.clear()

Earlier pages remain delivered. Nothing from a failing page is yielded. Iterators
do not retry, skip, or resume automatically after failure.

## Observation Bounds

Get decodes/constructs one returned record, or none if absent. Exists/count load
none. First/last load at most one. Page loads only returned records, each once.
Edge reads load edges only, never endpoints. Refresh loads one replacement.
Index decoding and application-created extra objects are outside these bounds.
No exact command or network-request count is promised. Backend scan guarantees
require separate observation; see [Guarantees](guarantees.md).

## Cleanup

    >>> for record in Item.iterate(batch_size=100):
    ...     _ = record.delete()
    >>> db.close()
