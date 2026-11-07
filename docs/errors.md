# Errors and Arguments

[Documentation](../README.md#documentation)

## Exceptions

Comfy Chair exceptions inherit from `comfychair.Error`, which inherits from
`Exception`. Built-in exceptions retain their ordinary Python types.

| Exception | Meaning |
| --- | --- |
| `TypeError` | Invalid argument usage or model declaration. |
| `ValueError` | Invalid value or combination, including missing adapters and unsupported write modes. |
| `KeyError` | Missing record or empty `first()` / `last()`, unless `default` is supplied. |
| `ValidationError` | Invalid model values or index adapter output. |
| `SerializationError` | Record encoding or decoding failure. |
| `UnsupportedQueryError` | Query cannot use a supported index. |
| `InvalidCursorError` | Malformed cursor or cursor incompatible with the query. |
| `ConflictError` | Failed write condition. |
| `StorageError` | Connection, timeout, or other backend failure. |
| `CommitError` | Invalid commit usage or access to an unconfirmed result. |

Wrapped errors preserve the original exception as their cause. Model and field
details are included where applicable. Application callback exceptions
propagate unchanged.

`default` handles absence only; it does not suppress other errors. There is no
separate configuration or closed-database exception. Invalid configuration uses
ordinary Python exceptions; closed databases reopen lazily.

## Arguments

Arguments follow ordinary Python protocols rather than exact type checks.
`limit`, `batch_size`, and `attempts` use the integer protocol (`__index__`)
and must be positive. There is no special exclusion for booleans or arbitrary
maximum. `reverse` uses ordinary truth testing.

```python
Task.page(limit=0)     # ValueError: limit must be positive.
Task.page(limit=2.0)   # TypeError: a float is not an integer argument.
```

`limit=None` does not request all records; use `iterate()`. Database objects,
field descriptors, and callables are used without upfront type or signature
inspection. Semantic constraints, such as a supported ordering for the queried
model, still apply.

Integer arguments and positivity are checked before storage access, including
at the call to `iterate()`. Query structure is checked before storage access.
Other argument errors may arise when the argument is used.

Queued operations check supported modes when queued. Identities, fields, and
ETags are captured and validated at context exit.

## Storage Failures

Backend failures raise `StorageError`, including during commits. A connection
failure may leave the write outcome unknown; the exception does not imply that
nothing was written. Unconfirmed result handles raise `CommitError` when their
`.value` is accessed.

Only `ConflictError` triggers the explicit mutation and commit retry helpers.
Storage failures are not retried automatically.
