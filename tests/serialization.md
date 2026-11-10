# JSON Serialization

Source: [Serialization](../docs/serialization.md).

## Setup

    >>> import json
    >>> from datetime import datetime, timezone
    >>> from enum import Enum
    >>> from uuid import UUID
    >>> from pytest import raises
    >>> from comfychair import Database, Key, Node, SerializationError
    >>> db = Database()
    >>> scope = 'json_' + str(Key())
    >>> class Status(Enum):
    ...     OPEN = 'open'
    >>> class Record(Node):
    ...     namespace = scope
    ...     title: str
    ...     count: int
    ...     ratio: float
    ...     active: bool
    ...     optional: str | None
    ...     tags: list[str]
    ...     details: dict[str, int]
    ...     created: datetime
    ...     external: UUID
    ...     status: Status
    >>> db.bind(Record)
    >>> record = Record(
    ...     title='Example', count=2, ratio=1.5, active=True, optional=None,
    ...     tags=['one'], details={'two': 2},
    ...     created=datetime(2026, 1, 1, tzinfo=timezone.utc),
    ...     external=UUID('00000000-0000-4000-8000-000000000001'), status=Status.OPEN,
    ... )

## Round Trip Declared Types

    >>> record.set()
    >>> loaded = Record.get(record.key)
    >>> loaded.tags
    ['one']
    >>> loaded.details
    {'two': 2}
    >>> loaded.created == record.created
    True
    >>> loaded.external == record.external
    True
    >>> loaded.status is Status.OPEN
    True

Standard JSON values are strings, signed 64-bit integers, finite floats,
booleans, nulls, arrays, and string-keyed objects. Key/UUID encode as standard
UUID strings, datetime as ISO 8601, and Enum as its underlying value. Reads
reconstruct declared types. Indexed datetimes must be timezone-aware.

## Inspect Current Values Without Saving

    >>> etag = record.etag
    >>> record.title = 'Unsaved'
    >>> text = record.to_json()
    >>> isinstance(text, str)
    True
    >>> copy = Record.from_json(text)
    >>> copy.title
    'Unsaved'
    >>> copy.key == record.key
    True
    >>> copy.etag == etag
    True
    >>> Record.get(record.key).title
    'Example'

To-json uses storage's conversion and record structure, including identity and
current ETag. It does not fetch storage or generate an ETag. Whitespace and
member order are unspecified; re-encoding need not reproduce saved bytes.

## Accept UTF-8 Bytes

    >>> copy = Record.from_json(text.encode('utf-8'))
    >>> copy.key == record.key
    True

From-json calls the normal constructor without saving or associating a database.
Both methods do no database I/O. Using selects adapters; omission follows
instance association, binding, then default.

## Observe Primitive Conversion

These real JSON functions record their inputs and delegate to the standard
library. They do not replace storage or decoding with canned responses.

    >>> encoded, decoded = [], []
    >>> def dumps(value):
    ...     encoded.append(value)
    ...     return json.dumps(value)
    >>> def loads(value):
    ...     decoded.append(value)
    ...     return json.loads(value)
    >>> observed = Database(json_dumps=dumps, json_loads=loads)
    >>> text = record.to_json(using=observed)
    >>> def leaves(value):
    ...     if isinstance(value, dict):
    ...         return [leaf for child in value.values() for leaf in leaves(child)]
    ...     if isinstance(value, list):
    ...         return [leaf for child in value for leaf in leaves(child)]
    ...     return [value]
    >>> str(record.key) in leaves(encoded[-1])
    True
    >>> str(record.external) in leaves(encoded[-1])
    True
    >>> 'open' in leaves(encoded[-1])
    True
    >>> copy = Record.from_json(text, using=observed)
    >>> isinstance(decoded[-1], bytes)
    True

No wire-envelope field names are assumed. JSON functions receive already
converted primitives; type reconstruction follows decoding. dumps returns str
or UTF-8 bytes; loads accepts UTF-8 bytes and returns primitives. Supply both
functions together. Defaults use the standard library.

## Reject Non-Finite Floats

    >>> record.ratio = float('nan')
    >>> with raises(SerializationError):
    ...     record.to_json()
    >>> record.ratio = 1.5

Non-string object keys and out-of-range integers are also rejected. Larger
integers require an explicit string representation. Unsupported values are not
silently stringified. Encoder/decoder failures preserve their original cause.

## Reject Oversized Integers

    >>> record.count = 2 ** 63
    >>> with raises(SerializationError):
    ...     record.to_json()
    >>> record.count = 2

## Reject Non-String Object Keys

    >>> record.details = {1: 2}
    >>> with raises(SerializationError):
    ...     record.to_json()
    >>> record.details = {'two': 2}

## Select Optional Engines

See [JSON Engines](json-engines.md) for orjson and msgspec examples. Installing
an engine does not select it. Neither engine extends the supported value
contract or decides custom type representations. Run the same round-trip and
representation observations in each selected engine environment.

Edges encode endpoint key types without embedding or recursively persisting
nodes. Index encodings are separate from record JSON; serialization alone does
not make a type indexable.

## Cleanup

    >>> for item in Record.iterate(batch_size=100):
    ...     _ = item.delete()
    >>> db.close()
    >>> observed.close()
