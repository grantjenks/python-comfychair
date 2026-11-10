# Errors and Arguments

Source: [Errors and Arguments](../docs/errors.md).

## Setup

    >>> from pytest import raises
    >>> from comfychair import (Database, Key, Node, Error, ValidationError,
    ...     SerializationError, UnsupportedQueryError, InvalidCursorError,
    ...     ConflictError, StorageError, CommitError)
    >>> db = Database()
    >>> scope = 'errors_' + str(Key())
    >>> class Task(Node):
    ...     namespace = scope
    ...     title: str
    >>> db.bind(Task)
    >>> Task(title='One').set()

## Catch Library Errors Together

    >>> all(issubclass(kind, Error) for kind in (
    ...     ValidationError, SerializationError, UnsupportedQueryError,
    ...     InvalidCursorError, ConflictError, StorageError, CommitError))
    True
    >>> issubclass(Error, Exception)
    True

Built-in TypeError, ValueError, and KeyError retain their ordinary types.

## Require Positive Limits

    >>> with raises(ValueError):
    ...     Task.page(limit=0)
    >>> with raises(TypeError):
    ...     Task.page(limit=2.0)
    >>> with raises(TypeError):
    ...     Task.page(limit=None)

## Use the Integer Protocol

    >>> class One:
    ...     def __index__(self):
    ...         return 1
    >>> len(Task.page(limit=One()))
    1

## Accept Python Booleans Naturally

    >>> len(Task.page(limit=True))
    1
    >>> with raises(ValueError):
    ...     Task.page(limit=False)

Limit, batch_size, and attempts use __index__ and must be positive, without
special boolean exclusions or arbitrary maximums. None is not an unlimited
page request; use iterate. Reverse uses ordinary truth testing.

## Check Iteration Arguments at the Call

    >>> with raises(ValueError):
    ...     Task.iterate(batch_size=0)

Integer conversion/positivity and query structure are checked before storage
access. Other argument errors may occur when used. Database objects, descriptors,
and callables are used without upfront type or signature inspection; semantic
constraints such as index support still apply.

## Let Defaults Handle Only Absence

    >>> Task.get(Key(), default=None) is None
    True
    >>> with raises(UnsupportedQueryError):
    ...     Task.first(Task.title == 'One', default=None)

## Preserve Callback Exceptions

    >>> task = Task(title='Callback')
    >>> task.set()
    >>> failure = RuntimeError('application failure')
    >>> def fail(working):
    ...     raise failure
    >>> with raises(RuntimeError) as caught:
    ...     task.mutate([fail])
    >>> caught.value is failure
    True

## Exception Boundaries

| Exception | Meaning |
| --- | --- |
| TypeError | Invalid argument usage or model definition. |
| ValueError | Invalid values/combinations, missing adapters, unsupported modes. |
| KeyError | Missing record or empty first/last, unless default supplied. |
| ValidationError | Invalid model values or index adapter output. |
| SerializationError | Record encoding or decoding failure. |
| UnsupportedQueryError | No supported index query. |
| InvalidCursorError | Malformed or incompatible cursor. |
| ConflictError | Failed write condition. |
| StorageError | Connection, timeout, or other backend failure. |
| CommitError | Invalid commit usage or unconfirmed result access. |

Wrapped errors preserve causes and include model/field details where applicable.
Application callback exceptions propagate unchanged. Configuration uses ordinary
Python errors; there is no separate configuration or closed-database error.
Closed databases reopen lazily. Queued modes are checked when queued; identities,
fields, and ETags are captured/validated at context exit.

Backend failure can leave writes unconfirmed. StorageError does not imply no
writes. Result.value raises CommitError until confirmed. Only ConflictError
triggers explicit mutation/commit retries; storage failures never replay
automatically. See [Guarantees](guarantees.md).

## Cleanup

    >>> for record in Task.iterate(batch_size=100):
    ...     _ = record.delete()
    >>> db.close()
