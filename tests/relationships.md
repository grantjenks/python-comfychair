# Relationships and Ownership

Source: [Relationships and Exclusive Ownership](../docs/relationships.md).

## Setup

    >>> from pytest import raises
    >>> from comfychair import (Database, Edge, Key, Node, Commit,
    ...     ConflictError, IF_MISSING, IF_NOT_MODIFIED)
    >>> db = Database()
    >>> scope = 'owners_' + str(Key())
    >>> class User(Node):
    ...     namespace = scope
    ...     name: str
    >>> class Email(Node):
    ...     namespace = scope
    ...     key: str
    >>> class EmailOwner(Edge[Email, User]):
    ...     namespace = scope
    >>> db.bind(User, Email, EmailOwner)

## Claim an Address

Email claims a key. EmailOwner relates it to a user. Normalize addresses in
application code before claiming them.

    >>> def create_user(name, address, using=db):
    ...     user = User(name=name)
    ...     email = Email(key=address)
    ...     ownership = EmailOwner(source=email, target=user)
    ...     with Commit(using=using) as batch:
    ...         _ = batch.set(email, mode=IF_MISSING)
    ...         _ = batch.set(user, mode=IF_MISSING)
    ...         _ = batch.set(ownership, mode=IF_MISSING)
    ...     return user
    >>> alice = create_user('Alice', 'alice@example.com')
    >>> Email.exists('alice@example.com')
    True

## Resolve an Address

    >>> ownership = EmailOwner.first('alice@example.com')
    >>> user = User.get(ownership.target)
    >>> user.name
    'Alice'
    >>> ownership.source
    'alice@example.com'
    >>> isinstance(ownership.target, Key)
    True

No declared Ordering is required. Edges store keys, not embedded users. A missing
first edge raises KeyError unless default is supplied. There is no automatic
reverse edge or incoming traversal from a user to their email.

## Reject a Duplicate Claim

    >>> before = User.count()
    >>> with raises(ConflictError):
    ...     create_user('Another Alice', 'alice@example.com')
    >>> User.count() == before
    True
    >>> EmailOwner.count('alice@example.com')
    1

IF_MISSING gates all writes. An unsuccessful add would only return False and
would not cancel user/edge creation.

## Compete From Two Connections

Use a barrier, not sleeps, to let two requests contend on the same email key.
Both clients execute real storage operations.

    >>> from concurrent.futures import ThreadPoolExecutor
    >>> from threading import Barrier
    >>> clients = [Database(), Database()]
    >>> barrier = Barrier(2)
    >>> def request(index):
    ...     barrier.wait(timeout=10)
    ...     try:
    ...         _ = create_user('Contender', 'shared@example.com', using=clients[index])
    ...         return 'created'
    ...     except ConflictError:
    ...         return 'conflict'
    >>> before = User.count()
    >>> with ThreadPoolExecutor(max_workers=2) as pool:
    ...     outcomes = list(pool.map(request, (0, 1)))
    >>> sorted(outcomes)
    ['conflict', 'created']
    >>> User.count() - before
    1
    >>> EmailOwner.count('shared@example.com')
    1

## Swap Two Owners

Start with distinct addresses/users and consistent ownership. Only four edge
writes are needed; user and email nodes remain unchanged.

    >>> bob = create_user('Bob', 'bob@example.com')
    >>> alice_etag, bob_etag = alice.etag, bob.etag
    >>> email_etag = Email.get('alice@example.com').etag
    >>> def swap(email_a, user_a, email_b, user_b):
    ...     old_a = EmailOwner.get(email_a, user_a)
    ...     old_b = EmailOwner.get(email_b, user_b)
    ...     new_a = EmailOwner(source=email_a, target=user_b)
    ...     new_b = EmailOwner(source=email_b, target=user_a)
    ...     with Commit() as batch:
    ...         _ = batch.delete(old_a, mode=IF_NOT_MODIFIED)
    ...         _ = batch.delete(old_b, mode=IF_NOT_MODIFIED)
    ...         _ = batch.set(new_a, mode=IF_MISSING)
    ...         _ = batch.set(new_b, mode=IF_MISSING)
    >>> swap('alice@example.com', alice.key, 'bob@example.com', bob.key)
    >>> User.get(EmailOwner.first('alice@example.com').target).name
    'Bob'
    >>> User.get(EmailOwner.first('bob@example.com').target).name
    'Alice'

## Preserve Node Versions During a Swap

    >>> User.get(alice.key).etag == alice_etag
    True
    >>> User.get(bob.key).etag == bob_etag
    True
    >>> Email.get('alice@example.com').etag == email_etag
    True

Both old ETags must match and new edges must be absent. All four checks precede
writes; conflict changes no relationships through the batch. The workflow
requires one Redis primary or Cluster slot.

## Recognize That Ownership Is an Application Rule

    >>> extra = EmailOwner(source='alice@example.com', target=alice)
    >>> extra.set()
    >>> EmailOwner.count('alice@example.com')
    2

Edges identify pairs and permit multiple targets per source. No unique Lookup
index or automatic ownership constraint exists. All writers must preserve the
application's one-owner invariant.

## Keep the Claim After Deleting an Edge

    >>> extra.delete()
    True
    >>> Email.exists('alice@example.com')
    True

Applications manage both lifecycles explicitly; deletion never cascades.

## Cleanup

    >>> for email in Email.iterate(batch_size=100):
    ...     for ownership in EmailOwner.iterate(email.key, batch_size=100):
    ...         _ = ownership.delete()
    ...     _ = email.delete()
    >>> for user in User.iterate(batch_size=100):
    ...     _ = user.delete()
    >>> for client in clients:
    ...     client.close()
    >>> db.close()
