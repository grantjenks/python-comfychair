# Choices and validation

[README](../README.md) · [Documentation](../README.md#documentation)

Examples use the models defined in the [introductory example](example.md).

Use `Literal` annotations for a fixed set of choices, as in the `Assignment`
model's `status` field. Python enum types are also supported: fields hold enum
members in Python and serialize their underlying values. An enum is indexable
when its values share a supported index type; ordering follows those values,
not member declaration order.

Call `record.validate()` to check field types, required values, choices, and
supported index encodings without accessing Redis. It returns `None` on success
and raises `ValidationError` on invalid data. Validation does not silently
coerce values to other types.

Override `validate()` for rules involving multiple fields, and call
`super().validate()` to retain the built-in checks:

```python
from comfychair import ValidationError


class Reservation(Node):
    starts_at: datetime
    ends_at: datetime

    def validate(self):
        super().validate()
        if self.ends_at <= self.starts_at:
            raise ValidationError("ends_at must be later than starts_at")
```

`set()`, `add()`, and creation through `get_or_create()` validate before
writing. Each `mutate()` attempt validates its resulting record before writing.
Writes in a `Commit` validate the captured object values at context exit, before any writes. Mutating an object's fields alone does not trigger
validation. Local validation does not establish existence or ETag matches;
those checks happen atomically on Redis during the operation.

Model definitions are checked at class creation. Invalid declarations raise
`TypeError`, including index fields that do not belong to the model,
unsupported indexed field types, incompatible constant defaults, and stored
fields that collide with reserved API names. Default factories are checked when
their values are produced, not invoked during class creation. These checks do
not inspect existing Redis data or migrate stored records.
