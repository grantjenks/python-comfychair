# Serialization

[README](../README.md) · [Documentation](../README.md#documentation)

Examples use the models defined in the [introductory example](example.md).

Record values are serialized as JSON, with an automatically managed ETag stored
alongside the record's fields. Standard JSON values include strings, numbers,
booleans, nulls, arrays, and objects.

Serialization also supports fields whose Python types are not native JSON
types, including `Key`, UUIDs, and datetimes. `Key` values encode as standard UUID
strings and are restored as `Key` objects. The other types use explicit
JSON-compatible encodings and are restored to their declared Python types when read. Custom
field types can supply encoding and decoding adapters. Unsupported values raise
a serialization error rather than silently converting to strings.

Edges serialize their source and target primary-key values using the types
specified by their endpoint models. Serialization does not embed or recursively
persist related nodes. Index encodings are separate from record JSON: an
ordered index must preserve the field's comparison semantics rather than sort
its JSON text.
