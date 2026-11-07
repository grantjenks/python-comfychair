# Serialization

[Documentation](../README.md#documentation)

Record values use JSON. An automatically managed ETag is stored alongside
fields. Standard values include strings, numbers, booleans, nulls, arrays, and
objects.

`Key`, UUID, and datetime fields have JSON-compatible encodings and reconstruct
their declared Python types on read. `Key` encodes as a standard UUID string.
Custom types can supply encoding and decoding adapters. Unsupported values
raise a serialization error; they are not implicitly converted to strings.

Edge endpoints use the key types declared by their endpoint models.
Serialization neither embeds nor recursively persists related nodes.

Index encodings are separate from record JSON.
[Ordered indexes](indexes.md#indexed-values) must preserve comparison
semantics, not JSON text order.
