# Decoding and Loading

Sources: [Models](../docs/models.md), [Pages and Cursors](../docs/pagination.md),
and [Serialization](../docs/serialization.md).

## Setup

A real decoder records each conversion. It still reconstructs the declared type.

    >>> from dataclasses import dataclass
    >>> from pytest import raises
    >>> from comfychair import Database, Edge, Key, Node, SerializationError
    >>> db = Database()
    >>> scope = 'decoding_' + str(Key())
    >>> decoded = []
    >>> failures = set()
    >>> @dataclass(frozen=True)
    ... class Label:
    ...     value: str
    >>> def decode(value):
    ...     decoded.append(value)
    ...     if value in failures:
    ...         raise ValueError('invalid label')
    ...     return Label(value)
    >>> _ = db.register_type(Label, encode=lambda label: label.value, decode=decode)
    >>> class Item(Node):
    ...     namespace = scope
    ...     key: str
    ...     label: Label
    >>> class Link(Edge[Item, Item]):
    ...     namespace = scope
    >>> db.bind(Item, Link)
    >>> Item(key='a', label=Label('first')).set()
    >>> Item(key='b', label=Label('second')).set()
    >>> Item(key='c', label=Label('third')).set()
    >>> Link(source='a', target='b').set()

## Verify the Decoder Observer

    >>> decoded.clear()
    >>> item = Item.get('a')
    >>> item.label.value
    'first'
    >>> decoded
    ['first']

## Avoid Decoding for Existence

    >>> decoded.clear()
    >>> Item.exists('a')
    True
    >>> decoded
    []

## Avoid Decoding for Count

    >>> Item.count()
    3
    >>> decoded
    []

## Avoid Decoding Endpoints

    >>> link = Link.get('a', 'b')
    >>> link.target
    'b'
    >>> decoded
    []

## Decode Only Returned Records

    >>> page = Item.page(limit=2)
    >>> decoded
    ['first', 'second']

## Keep a Failed Refresh Unchanged

    >>> item = Item.get('b')
    >>> item.label = Label('unsaved')
    >>> etag = item.etag
    >>> failures.add('second')
    >>> with raises(SerializationError):
    ...     item.refresh()
    >>> item.label.value
    'unsaved'
    >>> item.etag == etag
    True

## Do Not Treat a Decoder Failure as Absence

    >>> with raises(SerializationError) as caught:
    ...     Item.get('b', default=None)
    >>> isinstance(caught.value.__cause__, ValueError)
    True

## Fail the Entire Page

    >>> with raises(SerializationError):
    ...     Item.page(limit=3)

## Stop an Iterator at the Failing Page

    >>> records = Item.iterate(batch_size=1)
    >>> next(records).key
    'a'
    >>> with raises(SerializationError):
    ...     next(records)
    >>> with raises(StopIteration):
    ...     next(records)

Nothing from a failing page is returned, while previously yielded pages remain
delivered. Serialization failures are not missing-record events. No retry,
skipping, or automatic resume follows the failure.

## Cleanup

    >>> failures.clear()
    >>> Link.delete('a', 'b')
    True
    >>> for record in Item.iterate(batch_size=100):
    ...     _ = record.delete()
    >>> db.close()
