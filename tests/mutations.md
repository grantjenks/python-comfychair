# Mutations

Source: [Mutations](../docs/mutations.md).

## Setup

    >>> from pytest import raises
    >>> from comfychair import Database, Key, Node, Commit, ConflictError
    >>> db = Database()
    >>> scope = 'mutate_' + str(Key())
    >>> class Counter(Node):
    ...     namespace = scope
    ...     value: int = 0
    >>> db.bind(Counter)
    >>> def increment(counter):
    ...     counter.value += 1

## Mutate and Save in One Call

    >>> counter = Counter()
    >>> counter.set()
    >>> counter.mutate([increment]) is None
    True
    >>> counter.value
    1
    >>> Counter.get(counter.key).value
    1

Mutate works on nodes and edges. Each function edits its argument and returns
None. Functions run in order, then validation, serialization, and IF_NOT_MODIFIED
save. Confirmed success transfers fields and the new ETag to the original.

## Apply Callbacks in Order

    >>> def double(counter):
    ...     counter.value *= 2
    >>> counter = Counter(value=1)
    >>> counter.set()
    >>> counter.mutate([increment, double])
    >>> counter.value
    4

## Include Unsaved Edits on the First Attempt

    >>> counter = Counter(value=1)
    >>> counter.set()
    >>> counter.value = 10
    >>> counter.mutate([increment])
    >>> counter.value
    11

## Reload and Replay After a Conflict

    >>> competitor = Database()
    >>> counter = Counter(value=1)
    >>> counter.set()
    >>> counter.value = 10
    >>> seen = []
    >>> def concurrent_increment(working):
    ...     seen.append(working.value)
    ...     if len(seen) == 1:
    ...         current = Counter.get(working.key, using=competitor)
    ...         current.value = 20
    ...         current.set()
    ...     working.value += 1
    >>> counter.mutate([concurrent_increment], attempts=3)
    >>> seen
    [10, 20]
    >>> counter.value
    21

Unsaved edits participate only in attempt one. Later working copies start from
fresh storage; failed edits do not accumulate. Put repeatable changes in callbacks.

## Materialize the Iterable Once

    >>> yielded = []
    >>> calls = []
    >>> def retry_once(working):
    ...     calls.append('call')
    ...     if len(calls) == 1:
    ...         raise ConflictError('retry')
    ...     working.value += 1
    >>> def callbacks():
    ...     yielded.append('yield')
    ...     yield retry_once
    >>> counter.mutate(callbacks())
    >>> yielded
    ['yield']
    >>> calls
    ['call', 'call']

The finite iterable is materialized once and reused. Callables are invoked
without upfront signature inspection.

## Reject an Empty Iterable

    >>> with raises(ValueError):
    ...     counter.mutate([])

## Require None From a Callback

    >>> before = counter.value
    >>> with raises(TypeError):
    ...     counter.mutate([lambda working: 1])
    >>> counter.value == before
    True

## Require an Initial ETag

    >>> with raises(ValueError):
    ...     Counter().mutate([increment])

## Stop on Non-Conflict Failures

    >>> calls = []
    >>> def fail(working):
    ...     calls.append('call')
    ...     working.value = -1
    ...     raise RuntimeError('stop')
    >>> before = counter.value
    >>> with raises(RuntimeError):
    ...     counter.mutate([fail], attempts=3)
    >>> calls
    ['call']
    >>> counter.value == before
    True

## Exhaust the Attempt Limit

    >>> errors = []
    >>> def conflict(working):
    ...     error = ConflictError('conflict')
    ...     errors.append(error)
    ...     raise error
    >>> with raises(ConflictError) as caught:
    ...     counter.mutate([conflict], attempts=2)
    >>> len(errors)
    2
    >>> caught.value is errors[-1]
    True

Attempts counts total executions, must be positive, and defaults to three.
Retries have no backoff. Any ConflictError retries, including one raised by a
callback or invoked operation. Other failures propagate. StorageError can mean
an unknown storage outcome; it is not retried. The original object is changed
only after confirmed success; callbacks/construction are not repeated afterward.

## Fail if Retry Reload Finds No Record

    >>> counter = Counter(value=1)
    >>> counter.set()
    >>> def delete_during_attempt(working):
    ...     _ = Counter.delete(working.key, using=competitor)
    ...     working.value += 1
    >>> with raises(KeyError):
    ...     counter.mutate([delete_during_attempt])
    >>> counter.value
    1

## Execute Independently of a Commit

    >>> counter = Counter(value=1)
    >>> counter.set()
    >>> with raises(RuntimeError):
    ...     with Commit() as batch:
    ...         counter.mutate([increment])
    ...         raise RuntimeError('discard batch')
    >>> Counter.get(counter.key).value
    2

There is no batch.mutate(). Use Commit(callable).retry() for coordinated reads
and writes. External effects and edits to closed-over objects are not isolated
or undone; perform external effects after success.

Using selects both retry reads and conditional writes. Omission follows instance
association, binding, then default. ETags compare in the selected database without
restricting which Database object read them.

## Cleanup

    >>> for record in Counter.iterate(batch_size=100):
    ...     _ = record.delete()
    >>> db.close()
    >>> competitor.close()
