# Mutation Working Copies

Source: [Mutations](../docs/mutations.md).

## Setup

    >>> from pytest import raises
    >>> from comfychair import Database, Field, Key, Node
    >>> db = Database()
    >>> scope = 'copies_' + str(Key())
    >>> class Basket(Node):
    ...     namespace = scope
    ...     items: list[str] = Field(default_factory=list)
    ...     metadata: dict[str, list[str]] = Field(default_factory=dict)
    >>> db.bind(Basket)

## Isolate Nested Mutable Values

    >>> basket = Basket(items=['apple'], metadata={'notes': ['first']})
    >>> basket.set()
    >>> def fail(working):
    ...     working.items.append('pear')
    ...     working.metadata['notes'].append('second')
    ...     assert basket.items == ['apple']
    ...     assert basket.metadata == {'notes': ['first']}
    ...     raise RuntimeError('stop')
    >>> with raises(RuntimeError):
    ...     basket.mutate([fail])
    >>> basket.items
    ['apple']
    >>> basket.metadata
    {'notes': ['first']}

## Transfer Confirmed Fields

    >>> def add_pear(working):
    ...     working.items.append('pear')
    >>> same = basket
    >>> basket.mutate([add_pear])
    >>> basket is same
    True
    >>> basket.items
    ['apple', 'pear']

## Preserve Aliases Inside the Copy

Declared fields are copied together, not independently.

    >>> class Pair(Node):
    ...     namespace = scope
    ...     left: list[str]
    ...     right: list[str]
    >>> db.bind(Pair)
    >>> pair = Pair(left=[], right=[])
    >>> pair.set()
    >>> shared = ['one']
    >>> pair.left = shared
    >>> pair.right = shared
    >>> def check_alias(working):
    ...     assert working.left is working.right
    ...     assert working.left is not shared
    ...     working.left.append('two')
    >>> pair.mutate([check_alias])
    >>> pair.left
    ['one', 'two']
    >>> shared
    ['one']

## Use the Custom Deep-Copy Protocol

    >>> copies = []
    >>> class Box:
    ...     def __init__(self, value):
    ...         self.value = value
    ...     def __deepcopy__(self, memo):
    ...         copies.append(self.value)
    ...         result = Box(self.value)
    ...         memo[id(self)] = result
    ...         return result
    >>> _ = db.register_type(Box, encode=lambda box: box.value, decode=Box)
    >>> class Parcel(Node):
    ...     namespace = scope
    ...     box: Box
    >>> db.bind(Parcel)
    >>> parcel = Parcel(box=Box(1))
    >>> parcel.set()
    >>> copies.clear()
    >>> def increment(working):
    ...     working.box.value += 1
    >>> parcel.mutate([increment])
    >>> copies
    [1]
    >>> parcel.box.value
    2

Immutable custom values may be shared. Mutable types must provide independent
copies. Copying uses Python's deep-copy protocol, not a JSON round trip or a
second adapter registry. Connections and adapter registries are not copied.

## Fail Before Callbacks When Copying Fails

    >>> class Uncopyable:
    ...     def __deepcopy__(self, memo):
    ...         raise RuntimeError('cannot copy')
    >>> _ = db.register_type(Uncopyable, encode=lambda value: None, decode=lambda value: Uncopyable())
    >>> class Held(Node):
    ...     namespace = scope
    ...     value: Uncopyable
    >>> db.bind(Held)
    >>> held = Held(value=Uncopyable())
    >>> held.set()
    >>> callbacks = []
    >>> def observe(working):
    ...     callbacks.append('called')
    >>> with raises(RuntimeError):
    ...     held.mutate([observe])
    >>> callbacks
    []

Working instances use the normal constructor and relevant ETag/database
association. A conflict discards the working copy and reloads storage before
making another. Successful transfer does not rerun callbacks or constructors.
Isolation covers the supplied working instance, not globals, closures, or
external storage. See [Mutations](mutations.md) for retry observations.

## Cleanup

    >>> for model in (Held, Parcel, Pair, Basket):
    ...     for record in model.iterate(batch_size=100):
    ...         _ = record.delete()
    >>> db.close()
