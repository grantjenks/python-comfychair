# Pages and Cursors

Source: [Pages and Cursors](../docs/pagination.md).

## Setup

    >>> from pytest import raises
    >>> from comfychair import Database, Field, Key, Node, Ordering, InvalidCursorError
    >>> db = Database()
    >>> scope = 'pages_' + str(Key())
    >>> class Item(Node):
    ...     namespace = scope
    ...     key: str
    ...     rank: int = Field(default=0)
    ...     indexes = (Ordering(rank),)
    >>> db.bind(Item)
    >>> for key in ('a', 'b', 'c', 'd'):
    ...     Item(key=key, rank=1).set()

## Read One Page

    >>> page = Item.page(limit=2)
    >>> [item.key for item in page]
    ['a', 'b']
    >>> len(page)
    2
    >>> page[0].key
    'a'
    >>> [item.key for item in page[1:]]
    ['b']

Pages are materialized read-only sequences. Length counts fetched records, not
all matches. Iteration, indexing, slicing, and length do no further I/O.

## Continue Exclusively

    >>> next_page = Item.page(limit=2, cursor=page.cursor)
    >>> [item.key for item in next_page]
    ['c', 'd']
    >>> next_page.cursor is None
    True

None means exhaustion. Omitting cursor or supplying None starts at the beginning.
A cursor is opaque and serializable; its concrete representation is unspecified.

## Continue Through Ties

    >>> page = Item.page(order_by=Item.rank, limit=2)
    >>> [item.key for item in page]
    ['a', 'b']
    >>> page = Item.page(order_by=Item.rank, limit=2, cursor=page.cursor)
    >>> [item.key for item in page]
    ['c', 'd']

Continuation is after the last examined entry: node key, target within an edge
source, or ordering value and primary-key tiebreaker. Lookahead is not consumed.
No offsets or full scans. At most limit records are returned; one extra index
entry may be inspected without fetching or decoding its record.

## Reject a Different Query

    >>> page = Item.page(limit=1)
    >>> with raises(InvalidCursorError):
    ...     Item.page(limit=1, cursor=page.cursor, reverse=True)

Cursors bind namespace, model, source, filters, ordering, direction, and
compatible index definition. Malformed or incompatible cursors raise
InvalidCursorError. Equivalent predicate objects and argument order are accepted.

## Reorder Equivalent Predicates

    >>> page = Item.page(Item.rank >= 0, Item.rank < 2, order_by=Item.rank, limit=1)
    >>> page = Item.page(
    ...     Item.rank < 2, Item.rank >= 0, order_by=Item.rank,
    ...     limit=1, cursor=page.cursor,
    ... )
    >>> [item.key for item in page]
    ['b']

## Change the Page Size

The cursor records a position, not the size of the next request.

    >>> page = Item.page(limit=1)
    >>> page = Item.page(limit=2, cursor=page.cursor)
    >>> [item.key for item in page]
    ['b', 'c']
    >>> page = Item.page(limit=1, cursor=page.cursor)
    >>> [item.key for item in page]
    ['d']
    >>> page.cursor is None
    True

## Read Live Data

    >>> page = Item.page(limit=1)
    >>> Item(key='0').set()
    >>> Item(key='z').set()
    >>> rest = Item.page(limit=20, cursor=page.cursor)
    >>> [item.key for item in rest]
    ['b', 'c', 'd', 'z']

There is no snapshot. New keys before the cursor are excluded; later keys may
appear. Changes to filtering or ordering values can skip or repeat records.
Concurrent deletion can produce short or empty pages with a cursor; continuation
must advance. A cursor reports observed index entries, not a guaranteed next record.

## Keep an Already Read Page

A later storage update does not change the objects already materialized in a
page. A new query observes the update.

    >>> page = Item.page(Item.rank == 1, order_by=Item.rank, limit=1)
    >>> changed = Item.get('a')
    >>> changed.rank = 9
    >>> changed.set()
    >>> [(item.key, item.rank) for item in page]
    [('a', 1)]
    >>> Item.get('a').rank
    9

## Read After Closing the Connection

The page remains usable. Since Database can reopen, this example alone does
not establish the no-I/O guarantee; that needs [transport observation](guarantees.md#observe-storage-work).

    >>> page = Item.page(limit=2)
    >>> db.close()
    >>> [item.key for item in page]
    ['0', 'a']
    >>> len(page)
    2

## Follow Cursors Explicitly

    >>> keys = []
    >>> page = Item.page(limit=2)
    >>> while True:
    ...     keys.extend(item.key for item in page)
    ...     if page.cursor is None:
    ...         break
    ...     page = Item.page(limit=2, cursor=page.cursor)
    >>> keys
    ['0', 'a', 'b', 'c', 'd', 'z']

Page and iterate accept keyword-only using. Edge and secondary-index pagination
use these same rules. Limits use positive integer arguments; see [Arguments](errors.md).

## Cleanup

    >>> for record in Item.iterate(batch_size=100):
    ...     _ = record.delete()
    >>> db.close()
