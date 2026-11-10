# JSON Engines

Source: [Serialization](../docs/serialization.md).

This page requires both optional packages, orjson and msgspec. Installing them
does not change database defaults. The examples configure each engine explicitly.

## Setup

    >>> from decimal import Decimal
    >>> import orjson
    >>> import msgspec
    >>> from comfychair import Database, Key, Node
    >>> scope = 'engines_' + str(Key())
    >>> class Product(Node):
    ...     namespace = scope
    ...     price: Decimal
    >>> standard = Database()
    >>> _ = standard.register_type(Decimal, encode=str, decode=Decimal)

## Use orjson

    >>> fast = Database(json_dumps=orjson.dumps, json_loads=orjson.loads)
    >>> _ = fast.register_type(Decimal, encode=str, decode=Decimal)
    >>> product = Product(price=Decimal('12.50'))
    >>> product.set(using=fast)
    >>> Product.get(product.key, using=fast).price
    Decimal('12.50')

## Inspect JSON Through a Bytes Encoder

to_json returns a string even when the underlying dumps returns bytes.

    >>> text = product.to_json(using=fast)
    >>> isinstance(text, str)
    True
    >>> Product.from_json(text, using=fast).price
    Decimal('12.50')

## Read With the Standard Library

Compatible field adapters preserve meaning across JSON engines.

    >>> Product.get(product.key, using=standard).price
    Decimal('12.50')

## Use msgspec

    >>> compact = Database(json_dumps=msgspec.json.encode, json_loads=msgspec.json.decode)
    >>> _ = compact.register_type(Decimal, encode=str, decode=Decimal)
    >>> another = Product(price=Decimal('3.75'))
    >>> another.set(using=compact)
    >>> Product.get(another.key, using=compact).price
    Decimal('3.75')

## Read With orjson

    >>> Product.get(another.key, using=fast).price
    Decimal('3.75')

Comfy Chair handles field conversion before encoding and type reconstruction
after decoding. Engine-specific representations must not extend or alter that
contract. Whitespace/member order and exact re-encoded bytes are not contractual.

## Cleanup

    >>> product.delete(using=fast)
    True
    >>> another.delete(using=compact)
    True
    >>> standard.close()
    >>> fast.close()
    >>> compact.close()
