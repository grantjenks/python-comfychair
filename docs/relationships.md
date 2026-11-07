# Relationships and Exclusive Ownership

[Documentation](../README.md#documentation) · [Example models](example.md)

Edges identify pairs. Multiple targets may share a source. Exclusive ownership
requires an application workflow; edge types and orderings do not enforce it.

## Email Ownership

`Email` claims an address. `EmailOwner` relates it to a user. The
[User model](example.md) stores a name, without a duplicated email field or
embedded relationship.

```python
class Email(Node):
    key: str


class EmailOwner(Edge[Email, User]):
    pass
```

`EmailOwner.source` is `str`; `EmailOwner.target` is `Key`. Resolve an address
through primary traversal:

```python
ownership = EmailOwner.first("alice@example.com")
user = User.get(ownership.target)
```

No `Ordering` is required. `first()` selects one edge, raising `KeyError` if
absent unless `default` is supplied. The application maintains the one-owner
invariant.

## Concurrent Creation

Create the email claim, user, and ownership edge in one commit. Addresses must
already be normalized according to application rules.

```python
def create_user(name, address):
    user = User(name=name)
    email = Email(key=address)
    ownership = EmailOwner(source=email, target=user)

    with Commit() as batch:
        batch.set(email, mode=IF_MISSING)
        batch.set(user, mode=IF_MISSING)
        batch.set(ownership, mode=IF_MISSING)

    return user
```

Concurrent requests for one address contend on the same `Email` key. One
succeeds; another raises `ConflictError` before writing a user or edge.
`IF_MISSING` gates the batch. An unsuccessful `add()` would return `False`
without cancelling other writes.

Directly adding ownership edges can violate the invariant. Deleting an edge
does not release the email claim. Applications manage both lifecycles.

## Swapping Owners

Requires distinct addresses and users with consistent existing ownership.
Replace two edges; leave email and user nodes unchanged.

```python
def swap_emails(email_a_key, user_a_key, email_b_key, user_b_key):
    old_a = EmailOwner.get(email_a_key, user_a_key)
    old_b = EmailOwner.get(email_b_key, user_b_key)

    new_a = EmailOwner(source=email_a_key, target=user_b_key)
    new_b = EmailOwner(source=email_b_key, target=user_a_key)

    with Commit() as batch:
        batch.delete(old_a, mode=IF_NOT_MODIFIED)
        batch.delete(old_b, mode=IF_NOT_MODIFIED)
        batch.set(new_a, mode=IF_MISSING)
        batch.set(new_b, mode=IF_MISSING)
```

Both old ETags must match and both new edges must be absent. All four checks
precede writes. Conflict leaves relationships unchanged by the batch.

All ownership changes must preserve the one-owner invariant. These examples
require one Redis primary or Cluster hash slot, as specified under
[commit limits](commits.md#limits).
