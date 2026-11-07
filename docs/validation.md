# Choices and Validation

[Documentation](../README.md#documentation) · [Example models](example.md)

## Choices

`Literal` defines a fixed set of values, as in `Assignment.status`. Enum fields
hold Python enum members and serialize their underlying values. Enums are
indexable when all values share a supported index type. Ordering follows
values, not declaration order.

## Validate

`record.validate()` checks field types, required values, and choices without
database access. Return `None` on success; raise `ValidationError` on invalid
data. Values are not coerced.

Storage operations additionally check required adapters and index encodings
against the selected database before issuing storage commands. These checks
do not depend on an override of `validate()`. Missing adapters raise
`ValueError`; invalid index adapter output raises `ValidationError`. See
[Errors and Arguments](errors.md).

Override for cross-field rules. Call `super().validate()` to retain built-in
checks.

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
writing. Each `mutate()` attempt validates its result. Commits validate
captured values at context exit before any writes. Field assignment alone does
not validate.

Existence and ETag conditions are checked atomically on Redis during execution,
not by local validation.

## Model Definitions

Class creation raises `TypeError` for invalid declarations, including:

- Index references to missing fields or fields belonging to another model.
- Incompatible constant defaults.
- Stored fields using reserved API names.

Default factories run and their values are checked when produced, not at class
creation. Declaration checks neither inspect existing Redis data nor migrate
records.

Custom types are not rejected merely because an adapter is absent at class
creation. A model can use multiple databases with different registrations.
Adapter availability is checked when an operation selects its database.
