# Choices and Validation

Source: [Choices and Validation](../docs/validation.md).

## Setup

    >>> from enum import Enum
    >>> from typing import Literal
    >>> from pytest import raises
    >>> from comfychair import Database, Field, Key, Node, Ordering, ValidationError
    >>> db = Database()
    >>> scope = 'validation_' + str(Key())
    >>> class Task(Node):
    ...     namespace = scope
    ...     status: Literal['open', 'closed'] = 'open'
    >>> db.bind(Task)

## Validate Explicitly

    >>> task = Task()
    >>> task.validate() is None
    True

## Reject an Invalid Choice

Assignment alone does not validate.

    >>> task.status = 'unknown'
    >>> with raises(ValidationError):
    ...     task.validate()
    >>> with raises(ValidationError):
    ...     task.set()

Validation checks field types, required values, and choices, without I/O or
coercion. Set, add, get-or-create creation, each mutation attempt, and captured
commit writes validate. Existence/ETag conditions are separate atomic store checks.

## Do Not Coerce a Value

    >>> class Count(Node):
    ...     namespace = scope
    ...     value: int
    >>> count = Count(value=1)
    >>> count.value = '1'
    >>> with raises(ValidationError):
    ...     count.validate()
    >>> count.value
    '1'

## Add a Cross-Field Rule

    >>> class Interval(Node):
    ...     namespace = scope
    ...     start: int
    ...     stop: int
    ...     def validate(self):
    ...         super().validate()
    ...         if self.stop <= self.start:
    ...             raise ValidationError('stop must follow start')
    >>> Interval(start=1, stop=2).validate()
    >>> with raises(ValidationError):
    ...     Interval(start=2, stop=1).validate()

## Observe Validation Timing

    >>> validations = []
    >>> class Observed(Node):
    ...     namespace = scope
    ...     title: str
    ...     def validate(self):
    ...         super().validate()
    ...         validations.append(self.title)
    >>> db.bind(Observed)
    >>> item = Observed(title='First')
    >>> validations
    []
    >>> item.set()
    >>> validations
    ['First']
    >>> validations.clear()
    >>> loaded = Observed.get(item.key)
    >>> validations
    []

Construction and reads do not call application validate(). Decoding still checks
representations and reconstructs types. Calling super retains built-in checks.
Adapter availability and index encoding checks cannot be bypassed by overrides:
missing adapters raise ValueError; invalid index output raises ValidationError.

## Order Enum Values

    >>> class Priority(Enum):
    ...     HIGH = 2
    ...     LOW = 1
    >>> class Ranked(Node):
    ...     namespace = scope
    ...     priority: Priority = Field()
    ...     indexes = (Ordering(priority),)
    >>> db.bind(Ranked)
    >>> Ranked(priority=Priority.HIGH).set()
    >>> Ranked(priority=Priority.LOW).set()
    >>> Ranked.first(order_by=Ranked.priority).priority is Priority.LOW
    True

Enum fields hold enum members and serialize underlying values. Homogeneous
supported underlying types are indexable, ordered by value rather than declaration.

## Reject an Invalid Constant Default

    >>> with raises(TypeError):
    ...     class BadDefault(Node):
    ...         count: int = 'one'

## Reject a Reserved Field Name

    >>> with raises(TypeError):
    ...     class BadName(Node):
    ...         set: str

## Reject an Index From Another Model

    >>> with raises(TypeError):
    ...     class BadIndex(Node):
    ...         indexes = (Ordering(Ranked.priority),)

Invalid declarations fail at class creation, including missing or foreign index
fields, incompatible constant defaults, and reserved API fields. No storage is
inspected or migrated. Factories run and their values are checked when produced,
not at class definition. Custom types may be declared before adapter registration;
availability depends on the database selected by an operation.

## Cleanup

    >>> for model in (Ranked, Observed, Task):
    ...     for record in model.iterate(batch_size=100):
    ...         _ = record.delete()
    >>> db.close()
