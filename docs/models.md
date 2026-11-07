# Models and fields

[README](../README.md) · [Documentation](../README.md#documentation)

Examples use the models defined in the [introductory example](example.md).

Declare stored fields with Python type annotations. Constructors accept keyword
arguments. Use `Field` for defaults, factories, and fields referenced in index
declarations.

The annotation supplies the value type; `Field` supplies configuration and
query behavior. `Field()` without a default declares a required value.

Plain annotations and ordinary defaults remain shorthand:

```python
class Note(Node):
    title: str
    priority: int = 0
```

Comfy Chair creates descriptors for these fields during class creation. Use an
explicit `Field()` when configuring a field or referencing it within the class
body, as in `Ordering(created_at)`. On an instance, field access returns the
stored Python value; on a class, it returns the descriptor:

```python
Task.created_at                  # Field descriptor
task.created_at                  # datetime value on a Task instance
Task.created_at >= start         # Query predicate
```

`Node` supplies a `key: Key` field. Omitting the key generates a fresh `Key`
when the object is constructed, without accessing storage. Inherit the field
or declare `key: Key` explicitly, as in the [example](example.md); both retain this
automatic default. An explicit factory is equivalent:

```python
from comfychair import Field, Key, Node


class Account(Node):
    key: Key = Field(default_factory=Key)
    name: str
```

Primary lookup and ordered traversal by key are automatic.

`Edge[User, Task]` supplies `source` and `target` fields whose value types
come from the endpoint models' primary keys. Both are `Key` values in this
example.
An edge's `key` is the pair `(source, target)`, scoped to its model and
namespace. There is one edge per endpoint pair for a given edge model.
Different edge models can connect the same pair independently.

Edges store endpoint keys and their own fields, not copies of endpoint
records. The generic parameters identify the endpoint models. Fetching an
edge does not fetch either node. Relationships are directed; outgoing indexes
do not imply an incoming index.

## Edge endpoints

Model relationships are stored as edges. Their `source` and `target` attributes
are ordinary primary-key values, used directly with model methods:

```python
assignment = Assignment(source=user.key, target=task.key, priority=3)
assignment.set()

loaded_user = User.get(assignment.source)
loaded_task = Task.get(assignment.target)
```

An edge constructor also accepts instances of the declared endpoint models
as shorthand and extracts their keys:

```python
assignment = Assignment(source=user, target=task)
assert assignment.source == user.key
assert assignment.target == task.key
```

Edge methods accept endpoint keys or the corresponding model instances;
source-scoped queries accept either form for the source. Passing a key never
fetches the endpoint or checks its existence. There are no reference wrappers,
embedded model fields, or implicit fetches.

## Generated keys

`Key` is Comfy Chair's public identifier type. It uses UUID7 when available in
Python's standard library (Python 3.14 and later), falling back to UUID4 on
earlier versions. Applications use `comfychair.Key` without importing or
choosing a UUID generator.

`Key` values are immutable, hashable, and comparable. Generate a key explicitly
with `Key()`, or reconstruct one from its standard UUID string representation:

```python
key = Key()
text = str(key)
restored = Key(text)

assert restored == key

account = Account(key=restored, name="Alice")
account.set()
same_account = Account.get(restored)
```

Parsing accepts both UUID4 and UUID7 regardless of the running Python version.
Reading stored keys preserves their values; existing keys remain valid when
upgrading Python. JSON serialization and reconstruction of `Key` fields are
automatic.

Keys compare in unsigned 128-bit numeric order. UUID7 keys approximately follow
generation time, but key order does not guarantee creation order across
generators or when UUID4 and UUID7 values are mixed. Declare an `Ordering` on
an explicit timestamp field when time is the intended query order.

## Primary keys

Nodes are identified by their `key` field; edges by their source and target
primary keys. Declare a node's `key` explicitly to use a supplied identifier
instead of a generated `Key`:

```python
class Document(Node):
    key: str
    title: str


document = Document(key="installation", title="Installation guide")
document.set()
found = Document.get("installation")
```

`get()` and `exists()` accept primary keys, not secondary field filters.
Use an `Ordering` query to find records by other fields. Primary keys cannot
be `None`.

Be careful not to change a record's key or an edge's endpoints. Identity is not
enforced as immutable. A subsequent write uses the new identity and can leave
the old record and edges behind; it does not rename or move the stored
record.

## Namespaces and storage identity

Set an optional `namespace` on each model to prefix its storage names:

```python
class Note(Node):
    namespace = "notes_plugin"
    title: str
```

Namespaces let separate packages define models with the same class name in one
Redis database without collisions. Omit the attribute to use no namespace
prefix. Model name and namespace together distinguish record types.

Edges may connect models in different namespaces. The edge declaration
identifies each endpoint model and its namespace. A namespace is a naming
prefix, not an authorization boundary or a Redis Cluster hash-slot assignment.

Model names and namespaces are part of storage identity. Renaming a model or
changing its namespace changes where it reads and writes; existing data is not
renamed or migrated automatically. Changing index declarations likewise does
not rebuild existing indexes. Take care when changing these declarations for
models that already have stored data.
