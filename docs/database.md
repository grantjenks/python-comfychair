# Database Configuration

[Documentation](../README.md#documentation) · [Example Models](example.md)

## Default Database

Without configuration, operations use `redis://localhost:6379/0`: Redis on
localhost, port 6379, database 0. Creating a `Database` or binding models is
optional. Redis must be running; Comfy Chair does not start a server.

The implicit connection is opened on first storage access. Importing the
package, declaring models, and constructing records perform no connection I/O.

## Database

`Database` identifies a storage connection. Pass database objects to `using`;
string aliases are not supported.

Explicit databases open lazily on first storage access. `Database(...)` and
`database.bind(...)` perform no connection I/O.

```python
from comfychair import Database

primary = Database("redis://localhost:6379/0")
archive = Database("redis://localhost:6379/1")

primary.bind(User, Task, Assignment)
```

## Connection Settings

`Database()` uses `redis://localhost:6379/0`. URL-based construction accepts
keyword-only `connect_timeout` and `timeout` arguments:

```python
database = Database(
    "redis://localhost:6379/0",
    connect_timeout=5.0,
    timeout=5.0,
)
```

| Setting | Default | Behavior |
| --- | --- | --- |
| URL | `redis://localhost:6379/0` | Host, port, database number, and optional credentials. |
| `connect_timeout` | `5.0` seconds | Socket connection timeout. |
| `timeout` | `5.0` seconds | Socket I/O timeout, not a deadline for the whole operation. |
| Authentication | None | Username and password may be supplied in the URL. |
| TLS | Determined by URL scheme | `redis://` uses TCP; `rediss://` uses TLS. |
| Command Retries | Disabled | Connection errors and timeouts propagate without replaying commands. |

Timeouts accept positive finite numbers of seconds or `None` to disable the
limit. Boolean, zero, negative, and non-finite values are invalid.

Credentials use the URL form `redis://username:password@host:port/database`;
reserved characters must be percent-encoded. TLS verifies the server
certificate and hostname using system trust roots. URL-based construction
does not expose an option to disable verification.

URL query options and arbitrary backend keyword arguments are not supported.
Use a configured client for custom certificates, credential providers, or
other backend-specific settings.

## Supplied Clients

`Database(client=redis_client)` uses an existing synchronous Redis client.
The client argument is mutually exclusive with an explicit URL or connection
options:

```python
database = Database(client=redis_client)
```

Comfy Chair does not change the client's connection settings. The supplied
client must return binary responses and have automatic command retries
disabled. Wrapping it performs no I/O; it may already have open connections.

Automatic command replay is disabled to avoid repeating writes with uncertain
outcomes. Explicit `mutate()` and `Commit(operation).retry()` retries remain
limited to `ConflictError`.

## Bind

`database.bind(*models)` supplies the default database for each node or edge
model. Binding does not restrict the model to that database. The same model
definition can store independent records in multiple databases.

An unbound model uses the implicit default unless `using` or an instance's
remembered database selects another connection. Constructing model instances
requires no binding.

## Using

`using` is keyword-only on all model and instance storage methods. It accepts a
`Database` object.

| Operations | Methods |
| --- | --- |
| Primary Lookup | `get()`, `exists()`, `get_or_create()` |
| Traversal | `page()`, `iterate()`, `first()`, `last()`, `count()` |
| Writes | `set()`, `add()`, `delete()` |
| Instance Updates | `refresh()`, `mutate()` |

```python
user = User.get(user_key)                  # Primary.
archived = User.get(user_key, using=archive)

archived.name = "Alice"
archived.set()                            # Archive.

User.delete(user_key, using=archive)
```

Database selection follows this order:

1. Explicit `using`.
2. The instance's remembered database, for instance operations.
3. The model's default binding.
4. The implicit default at `redis://localhost:6379/0`.

Connection errors propagate from the selected database. They do not trigger
fallback to another database.

## Instance State

A successful read or confirmed write associates the instance with its database.
Failed operations leave the association unchanged. `refresh()` and `mutate()`
use the same selection rules.

Each instance carries one database association and one ETag. Load separate
instances to work independently with records in different databases:

```python
current = User.get(user_key, using=primary)
archived = User.get(user_key, using=archive)
```

Writing an instance to another database preserves its key and applies the
selected write mode. It does not move or delete the original record. A
confirmed write updates the instance's database association and ETag.

`IF_NOT_MODIFIED` requires an ETag obtained from the selected database. An ETag
from another database cannot authorize a conditional set or delete. This
requirement also applies to `mutate()` and queued conditional operations.

## Edges

The selected database stores the edge record and its indexes. Endpoints remain
plain keys without database routing information. Passing endpoint model
instances extracts keys; it does not select the edge's database.

```python
edge = Assignment.get(user_key, task_key, using=archive)
task = Task.get(edge.target, using=archive)
```

Endpoint reads select their databases independently. No endpoint fetching or
existence checks are implied.

## Commit Routing

`Commit(using=database)` selects one database for every queued operation. Batch
methods do not accept per-operation `using` arguments.

```python
with Commit(using=archive) as batch:
    batch.set(user)
    batch.set(task)
    batch.set(assignment)
```

Without `using`, resolve each operation through its record's association,
model binding, then the implicit default. Model-and-key deletes use the model
binding or implicit default. All operations must resolve to the same database;
conflicting selections reject the batch before writing. Unconfigured models
share the implicit default, so `Commit()` needs no setup. An empty batch is a
no-op.

`Commit(operation, using=database).retry(attempts=3)` applies the selected database to each attempt's batch. Ordinary reads and writes inside the callable remain independent and require their own database selection. See [Retrying a Callable](commits.md#retrying-a-callable).
