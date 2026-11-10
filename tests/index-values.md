# Indexed Values and Adapters

Source: [Ordered Indexes](../docs/indexes.md).

## Setup

    >>> from dataclasses import dataclass
    >>> from pytest import raises
    >>> from comfychair import Database, Field, Key, Node, Ordering, ValidationError
    >>> db = Database()
    >>> scope = 'index_values_' + str(Key())

## Order Strings Exactly

    >>> class Label(Node):
    ...     namespace = scope
    ...     key: str
    >>> db.bind(Label)
    >>> for key in ('a', 'Z', 'A'):
    ...     Label(key=key).set()
    >>> [label.key for label in Label.page(limit=10)]
    ['A', 'Z', 'a']

String comparison is case-sensitive Unicode code-point order, without locale
collation or normalization. Applications normalize values such as email addresses.

## Omit Null Secondary Values

    >>> class Item(Node):
    ...     namespace = scope
    ...     key: str
    ...     group: str = Field(default='open')
    ...     rank: int | None = Field(default=None)
    ...     indexes = (Ordering(rank), Ordering(group, rank))
    >>> db.bind(Item)
    >>> Item(key='a', rank=1).set()
    >>> Item(key='b').set()
    >>> Item.count()
    2
    >>> Item.count(order_by=Item.rank)
    1
    >>> Item.count(Item.group == 'open', order_by=Item.rank)
    1

Any None component excludes the entire composite secondary entry. It still
serializes as null. Primary identity cannot be None.

## Define a Custom Comparison

    >>> @dataclass(frozen=True)
    ... class Money:
    ...     cents: int
    >>> _ = db.register_type(Money, encode=lambda value: value.cents, decode=Money)
    >>> _ = db.register_index(Money, encode=lambda value: value.cents, as_type=int)
    >>> class Product(Node):
    ...     namespace = scope
    ...     key: str
    ...     price: Money = Field()
    ...     indexes = (Ordering(price),)
    >>> db.bind(Product)
    >>> Product(key='high', price=Money(200)).set()
    >>> Product(key='low', price=Money(100)).set()
    >>> Product.first(order_by=Product.price).key
    'low'

## Encode Query Operands the Same Way

    >>> page = Product.page(Product.price >= Money(150), order_by=Product.price, limit=10)
    >>> [product.key for product in page]
    ['high']

The adapter's output defines equality and order. It must be deterministic and
return its declared supported scalar in range. Comfy Chair supplies the
storage-specific ordered encoding. JSON adapters do not imply index support.

## Reject Duplicate Registration

    >>> with raises(ValueError):
    ...     db.register_index(Money, encode=lambda value: value.cents, as_type=int)

Registrations are per database and exact declared type. Configure at startup.
Databases sharing indexes need compatible adapters. Changing an adapter requires
an application-managed index rebuild. Class creation checks index structure;
adapter availability and encodings are checked before storage commands, for
every operation in a commit before any write.

## Built-In Comparisons

| Type | Contract |
| --- | --- |
| bool | Exact equality; False before True. |
| int | Exact signed 64-bit values, numeric order. |
| float | Finite IEEE 754 binary64, numeric order; negative and positive zero equal. |
| Key, UUID | Exact equality, unsigned 128-bit numeric order. |
| datetime | Aware values normalized to UTC microseconds, ordered by instant. |
| Enum | Homogeneous supported underlying values, ordered by value. |

## Order Booleans

    >>> class Flag(Node):
    ...     namespace = scope
    ...     value: bool = Field()
    ...     indexes = (Ordering(value),)
    >>> db.bind(Flag)
    >>> Flag(value=True).set()
    >>> Flag(value=False).set()
    >>> [flag.value for flag in Flag.page(order_by=Flag.value, limit=10)]
    [False, True]

## Treat Both Floating Zeros Equally

    >>> class Sample(Node):
    ...     namespace = scope
    ...     value: float = Field()
    ...     indexes = (Ordering(value),)
    >>> db.bind(Sample)
    >>> Sample(value=-0.0).set()
    >>> Sample(value=0.0).set()
    >>> Sample.count(Sample.value == 0.0, order_by=Sample.value)
    2

## Compare Datetimes by Instant

    >>> from datetime import datetime, timedelta, timezone
    >>> class Event(Node):
    ...     namespace = scope
    ...     at: datetime = Field()
    ...     indexes = (Ordering(at),)
    >>> db.bind(Event)
    >>> instant = datetime(2026, 1, 1, 12, tzinfo=timezone.utc)
    >>> same_instant = datetime(2026, 1, 1, 4, tzinfo=timezone(timedelta(hours=-8)))
    >>> Event(at=instant).set()
    >>> Event.count(Event.at == same_instant, order_by=Event.at)
    1

Naive indexed datetimes, non-finite floats, unsupported index types, and
out-of-range values fail before writing. Queued values fail at context exit.
JSON arrays and objects have no built-in index encoding. Missing adapters raise
ValueError; invalid index adapter output raises ValidationError. These checks
remain required when application validate() is overridden.

## Check Adapters Independently of Validation

Adapter checks remain required even when application validation does nothing.

    >>> invalid = Database()
    >>> _ = invalid.register_type(Money, encode=lambda value: value.cents, decode=Money)
    >>> _ = invalid.register_index(Money, encode=lambda value: 'invalid', as_type=int)
    >>> class Unchecked(Node):
    ...     namespace = scope
    ...     price: Money = Field()
    ...     indexes = (Ordering(price),)
    ...     def validate(self):
    ...         pass
    >>> unchecked = Unchecked(price=Money(100))
    >>> with raises(ValidationError):
    ...     unchecked.set(using=invalid)
    >>> Unchecked.exists(unchecked.key, using=invalid)
    False
    >>> invalid.close()

## Cleanup

    >>> for model in (Event, Sample, Flag, Product, Item, Label):
    ...     for record in model.iterate(batch_size=100):
    ...         _ = record.delete()
    >>> db.close()
