# Serialization

[Documentation](../README.md#documentation)

Record values use JSON. An automatically managed ETag is stored alongside
fields. JSON functions and type adapters belong to the selected
[database](database.md).

## JSON Values

Standard values include strings, signed 64-bit integers, finite floats,
booleans, nulls, arrays, and objects with string keys. Non-finite floats and
non-string object keys are rejected. Larger integers require an explicit
string representation.

| Field Type | JSON Representation |
| --- | --- |
| `Key`, `UUID` | Standard UUID string. |
| `datetime` | ISO 8601 string. |
| Enum | Underlying value. |

Reads reconstruct declared Python types. Indexed datetimes must be
timezone-aware; see [Indexed Values](indexes.md#indexed-values).

Edge endpoints use the key types declared by their endpoint models.
Serialization neither embeds nor recursively persists related nodes.

## JSON Functions

The default uses the standard library `json` module. Supply both
`json_dumps` and `json_loads` to select other functions:

```python
import orjson
from comfychair import Database

database = Database(
    json_dumps=orjson.dumps,
    json_loads=orjson.loads,
)
```

The equivalent configuration for msgspec is:

```python
import msgspec

database = Database(
    json_dumps=msgspec.json.encode,
    json_loads=msgspec.json.decode,
)
```

`json_dumps(value)` accepts JSON-compatible Python values and returns a string
or UTF-8 bytes. `json_loads(value)` accepts UTF-8 bytes and returns
JSON-compatible Python values. Installing an optional library does not select
it automatically.

Comfy Chair converts typed fields before encoding and reconstructs them after
decoding. The selected JSON library does not determine custom type
representations or extend the supported value contract.

## Type Adapters

Register custom field conversions on a database:

```python
from decimal import Decimal
from comfychair import Database, Key, Node

database = Database()
database.register_type(Decimal, encode=str, decode=Decimal)
database.set_default()


class Product(Node):
    key: Key
    price: Decimal


product = Product(price=Decimal("12.50"))
product.set()
```

Each database starts with built-in adapters. Custom registrations apply only
to that database and to the exact declared type. Duplicate registrations raise
`ValueError`; registrations do not replace existing adapters.

Encoders return JSON-compatible values. Decoders return the declared Python
type. Annotations determine decoding, including elements of typed containers.
Custom objects inside untyped containers are unsupported. Stored JSON contains
no automatic Python type tags or import paths.

Register adapters during startup, before using models. Model declarations do
not require registration. An operation checks required adapters against its
selected database before issuing storage commands. Commits check all required
adapters and encodings before any writes.

Database objects accessing the same records must use compatible adapters.
Changing an encoding does not migrate stored data.

Unsupported values and encoding or decoding failures raise
`SerializationError`. Values are not implicitly converted to strings.
Underlying exceptions are preserved as causes.

## JSON Inspection

`record.to_json()` returns a string using the same field conversion and record
structure as storage. It represents current instance values, including
identity and the current ETag. It neither fetches the saved record nor
generates a new ETag.

`Model.from_json(value)` accepts a string or UTF-8 bytes and reconstructs
identity, fields, and the ETag without saving or associating the instance with
a database. It uses the normal
[construction path](models.md#construction-and-loading), including defaults for
missing optional fields.

```python
text = product.to_json()
copy = Product.from_json(text)

assert copy.key == product.key
assert copy.price == product.price
```

Both methods perform no database I/O. Keyword-only `using=database` selects
JSON functions and adapters. Without it, selection follows the usual instance
association, model binding, and current default rules.

Whitespace and object member order are not contractual. Re-encoding a record
need not reproduce previously stored bytes.

## Stored Schema Changes

| Stored Field | Loading Behavior |
| --- | --- |
| Missing field with a default | Use the default. |
| Missing field with a factory | Call the factory for the instance. |
| Missing required field | Raise an error. |
| Missing node key or edge endpoint | Raise an error; never generate replacement identity. |
| Unknown field | Ignore it. A subsequent save drops it. |
| Invalid encoded value | Raise `SerializationError`. |

Loading defaults does not write them back or create index entries. Schema
migrations and index rebuilds are application operations.

## Index Encodings

Index encodings are separate from record JSON.
[Index Adapters](indexes.md#index-adapters) define comparison semantics rather
than JSON text order. Serialization support alone does not make a type
indexable.
