# Database Configuration

[Documentation](../README.md#documentation) · [Example Models](example.md)

## Database

`Database` identifies a storage connection. Pass database objects to `using`;
string aliases are not supported.

```python
from comfychair import Database

primary = Database("redis://localhost:6379/0")
archive = Database("redis://localhost:6379/1")

primary.bind(User, Task, Assignment)
```

## Bind

`database.bind(*models)` supplies the default database for each node or edge
model. Binding does not restrict the model to that database. The same model
definition can store independent records in multiple databases.

An unbound model requires an explicit `using` argument or, for instance
operations, a remembered database. Otherwise, storage operations raise a
configuration error. Constructing model instances requires no binding.

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
4. A configuration error.

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

Without `using`, infer the database from queued records' associations and model
defaults. Model-and-key deletes use the model default. All operations must
resolve to the same database; missing or conflicting selections reject the
batch before writing. An empty batch is a no-op.

`Commit(operation, using=database).retry(attempts=3)` applies the selected database to each attempt's batch. Ordinary reads and writes inside the callable remain independent and require their own database selection. See [Retrying a Callable](commits.md#retrying-a-callable).
