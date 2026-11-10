# Connection Lifetime

Source: [Database Configuration](../docs/database.md).

## Setup

    >>> from pytest import raises
    >>> from comfychair import Database, Key, Node
    >>> scope = 'connections_' + str(Key())
    >>> class Note(Node):
    ...     namespace = scope
    ...     title: str

## Construct Lazily

No connection is opened by imports, model declaration, record construction,
Database construction, bind, set_default, or context entry.

    >>> db = Database()
    >>> db.bind(Note)
    >>> note = Note(title='First')

## Configure Timeouts

    >>> configured = Database(
    ...     'redis://localhost:6379/0', connect_timeout=5.0, timeout=5.0,
    ... )
    >>> unlimited = Database(connect_timeout=None, timeout=None)
    >>> with raises(ValueError):
    ...     Database(timeout=0)
    >>> with raises(ValueError):
    ...     Database(connect_timeout=-1)

Defaults are five seconds for socket connection and socket I/O, not an overall
operation deadline. Accept positive finite seconds or None. Reject zero,
negative, and non-finite values. Automatic command retries are disabled.

## Close Before Opening

    >>> db.close() is None
    True
    >>> db.close() is None
    True

## Reopen Lazily

    >>> note.set()
    >>> db.close()
    >>> Note.get(note.key).title
    'First'

Close releases owned connections and preserves bindings, adapters, default
selection, and instance associations. Finish outstanding work before closing;
there is no drain or cancellation.

## Use a Lifetime Context

    >>> with db as entered:
    ...     print(entered is db)
    ...     loaded = Note.get(note.key)
    True
    >>> loaded.title
    'First'
    >>> loaded.refresh()

Context exit closes, including on exceptions. It creates neither an ambient
database nor a transaction, and does not undo bindings. Nested contexts each
close on exit; later operations may reconnect.

## Keep the Current Default Selected

    >>> previous = Database.default()
    >>> db.set_default()
    >>> db.close()
    >>> Database.default() is db
    True
    >>> previous.set_default()

## Supply a Client

A supplied synchronous Redis client must return bytes and disable automatic
command replay. Comfy Chair neither reconfigures nor closes it. This usage is
shown as a contract rather than constructing a version-specific Redis client.

`Database(client=redis_client)` borrows the client. An explicit URL or connection
options cannot also be supplied. Wrapping performs no I/O; the borrowed client
may already have live connections. json_dumps/json_loads still apply.

## Configure Transport

- URL default: redis://localhost:6379/0.
- Credentials: redis://username:password@host:port/database, percent-encoding
  reserved characters. Default authentication: none.
- redis:// uses TCP; rediss:// uses TLS with system trust roots, certificate
  validation, and hostname verification. No URL option disables verification.
- URL query options and arbitrary backend keyword arguments are unsupported.
- Use a supplied client for custom certificates, credential providers, or other
  backend-specific settings. The application owns its lifetime.

## Share Connections Carefully

Database objects may be shared between threads; owned clients use a pool.
Instances provide no synchronization for concurrent field mutation. The API is
synchronous; async applications arrange execution off the event loop.

Create explicit databases and clients inside each worker process. Do not share
live connections across processes. Defaults and registrations are process-local.
Supplied clients must support the intended concurrency. Concurrent default
changes or rebinding are unsupported; configure at startup.

## Cleanup

    >>> for record in Note.iterate(batch_size=100):
    ...     _ = record.delete()
    >>> db.close()
    >>> configured.close()
    >>> unlimited.close()
