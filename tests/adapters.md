# Custom Types and Schema Changes

Source: [Serialization](../docs/serialization.md).

## Setup

    >>> from decimal import Decimal
    >>> from pytest import raises
    >>> from comfychair import Database, Field, Key, Node, SerializationError
    >>> db = Database()
    >>> scope = 'adapters_' + str(Key())

## Declare Before Registering

    >>> class Product(Node):
    ...     namespace = scope
    ...     price: Decimal
    >>> db.bind(Product)
    >>> product = Product(price=Decimal('12.50'))

Model declaration requires no database registration. Availability is checked
when an operation selects a database, before storage commands.

## Register a Conversion

    >>> _ = db.register_type(Decimal, encode=str, decode=Decimal)
    >>> product.set()
    >>> Product.get(product.key).price
    Decimal('12.50')

Encoders return JSON-compatible values; decoders return the declared type.
Each database starts with built-in adapters. Configure registrations at startup.

## Reject Duplicate Registration

    >>> with raises(ValueError):
    ...     db.register_type(Decimal, encode=str, decode=Decimal)

Registrations do not replace existing adapters. They apply to the exact declared
type, only on that database. Stores accessed through multiple Database objects
require compatible adapters. Changes do not migrate existing data.

## Keep Registrations Local to the Database

    >>> unconfigured = Database()
    >>> with raises(ValueError):
    ...     product.set(using=unconfigured)

## Reconstruct Typed Container Elements

    >>> class Invoice(Node):
    ...     namespace = scope
    ...     prices: list[Decimal]
    >>> db.bind(Invoice)
    >>> invoice = Invoice(prices=[Decimal('1.25'), Decimal('2.50')])
    >>> invoice.set()
    >>> Invoice.get(invoice.key).prices
    [Decimal('1.25'), Decimal('2.50')]

Custom objects in untyped containers are unsupported. Stored JSON has no
automatic Python import paths or type tags. Annotations determine reconstruction.

## Preserve an Encoding Failure's Cause

    >>> class Broken:
    ...     pass
    >>> failure = RuntimeError('encoder failed')
    >>> def encode_broken(value):
    ...     raise failure
    >>> _ = db.register_type(Broken, encode=encode_broken, decode=lambda value: Broken())
    >>> class BrokenRecord(Node):
    ...     namespace = scope
    ...     value: Broken
    >>> db.bind(BrokenRecord)
    >>> with raises(SerializationError) as caught:
    ...     BrokenRecord(value=Broken()).to_json()
    >>> caught.value.__cause__ is failure
    True

## Add a Field With a Default

The class name and namespace intentionally stay the same across schema versions.

    >>> class Article(Node):
    ...     namespace = scope
    ...     title: str
    >>> db.bind(Article)
    >>> article = Article(title='First')
    >>> article.set()
    >>> key = article.key
    >>> etag = article.etag
    >>> class Article(Node):
    ...     namespace = scope
    ...     title: str
    ...     priority: int = 0
    >>> db.bind(Article)
    >>> loaded = Article.get(key)
    >>> loaded.priority
    0
    >>> loaded.etag == etag
    True

## Apply a Factory to Missing Data

    >>> factories = []
    >>> def default_tags():
    ...     factories.append('called')
    ...     return []
    >>> class Article(Node):
    ...     namespace = scope
    ...     title: str
    ...     tags: list[str] = Field(default_factory=default_tags)
    >>> db.bind(Article)
    >>> loaded = Article.get(key)
    >>> factories
    ['called']
    >>> loaded.tags
    []

## Do Not Write Defaults During Loading

Loading the same old record still needs its missing field's factory.

    >>> factories.clear()
    >>> loaded = Article.get(key)
    >>> factories
    ['called']
    >>> loaded.set()
    >>> factories.clear()
    >>> loaded = Article.get(key)
    >>> factories
    []

## Drop Unknown Fields on Save

    >>> class Article(Node):
    ...     namespace = scope
    ...     title: str
    >>> db.bind(Article)
    >>> loaded = Article.get(key)
    >>> loaded.set()
    >>> class Article(Node):
    ...     namespace = scope
    ...     title: str
    ...     tags: list[str] = Field(default_factory=default_tags)
    >>> db.bind(Article)
    >>> factories.clear()
    >>> loaded = Article.get(key)
    >>> factories
    ['called']

Missing required fields fail. Missing node keys or edge endpoints fail rather
than generating new identity. Invalid encoded values raise SerializationError.
Unknown fields are ignored and lost on the next save. Loading defaults neither
writes them back nor creates index entries. Migrations and index rebuilds belong
to the application. From-json follows these same construction rules.

## Cleanup

    >>> for model in (Product, Invoice, Article):
    ...     for record in model.iterate(batch_size=100):
    ...         _ = record.delete()
    >>> db.close()
    >>> unconfigured.close()
