# Relationships and exclusive ownership

[README](../README.md) · [Documentation](../README.md#documentation)

Examples use the models defined in the [introductory example](example.md).

Edges identify pairs, so several targets can share one source. A rule such as
one owner per email is maintained by application workflows, not enforced by
the edge type or an ordered index.

Use an email node as the claim on an address and an edge for its owner.
The `User` model from the [example](example.md) stores a name, with no duplicated email
field or embedded relationship:

```python
class Email(Node):
    key: str


class EmailOwner(Edge[Email, User]):
    pass
```

An `EmailOwner` has a string `source` and a `Key` value for `target`. Find the owner using
only the email address through automatic source traversal:

```python
ownership = EmailOwner.first("alice@example.com")
user = User.get(ownership.target)
```

No `Ordering` declaration is needed. `first()` raises `KeyError` if no edge
exists, unless an explicit default is supplied. It selects one edge; the
application's ownership workflow ensures that there is only one.

## Concurrent creation

Create the email claim, user, and edge together. Addresses in this example
have already been normalized according to the application's rules.

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

Concurrent requests for the same address contend on the same `Email` key.
One commit succeeds; another finds the email claim present and raises
`ConflictError` before any of its writes. The losing request creates no user
or edge. Use `set(mode=IF_MISSING)` for this requirement: a failed `add()`
would produce `False` without cancelling the other writes.

This workflow reserves the address. Creating additional edges directly would
bypass the one-owner rule. Deleting an ownership edge alone does not release
the email claim; applications manage that lifecycle explicitly.

## Swapping owners

Given two distinct addresses and users with consistent existing ownership,
replace the two edges. Neither email nor user node changes:

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

The old edges' ETags must still match, and the new edges must be absent.
All four conditions are checked before any writes; a conflict leaves the
stored relationships unchanged by this batch. Ownership-changing workflows
must preserve the one-owner rule. These examples have the same Redis primary
or Cluster hash-slot requirement as other commits.
