# Models and Fields

[Documentation](../README.md#documentation) · [Example models](example.md)

## Field

Declare stored fields with type annotations. Constructors accept keyword
arguments. `Field` configures defaults, factories, and query descriptors.
`Field()` without a default requires a value.

Plain annotations and ordinary defaults are shorthand:

```python
class Note(Node):
    title: str
    priority: int = 0
```

Class creation supplies field descriptors. An explicit `Field()` is required to
configure a field or reference it in the class body, as in
`Ordering(created_at)`. Class access returns the descriptor; instance access
returns the value.

```python
Task.created_at                  # Field descriptor
task.created_at                  # datetime value on a Task instance
Task.created_at >= start         # Query predicate
```

## Construction and Loading

Construction applies defaults and default factories without storage access.
It does not automatically call `validate()`.

Loading decodes stored fields, checks required fields and identity, and calls
the model's normal constructor with decoded keyword arguments. The stored ETag
and database association are attached after construction. Missing optional
fields use their defaults or factories; present fields do not invoke factories.
Loading never generates replacement identity.

Constructor overrides observe both application construction and loading:

```python
constructed = 0


class Task(Node):
    title: str

    def __init__(self, **fields):
        super().__init__(**fields)
        global constructed
        constructed += 1
```

`get()` constructs the returned model. `exists()` and `count()` do not
construct model instances. Reading an edge constructs the edge without
constructing its endpoint nodes.

Constructor overrides must accept the declared keyword fields. Avoid side
effects that assume the record is new. There are no additional lifecycle hooks.

Reads reconstruct declared types but do not automatically call `validate()`.
Application validation runs explicitly or before writing. See
[Choices and Validation](validation.md).

## Node

`Node` supplies `key: Key`. Omitting the key generates a new `Key` at
construction, without storage access. Inheriting the field or declaring
`key: Key` retains this default. An explicit factory is equivalent:

```python
from comfychair import Field, Key, Node


class Account(Node):
    key: Key = Field(default_factory=Key)
    name: str
```

Primary lookup and ordered traversal by key are automatic.

## Edge Endpoints

`Edge[User, Task]` supplies `source` and `target` fields with the endpoint
models' primary-key types. Both are `Key` values in the [example](example.md).
An edge's `key` is `(source, target)`, scoped to its model and namespace. Each
edge model stores at most one record per endpoint pair.

Edges store endpoint keys and relationship fields. Relationships are directed.
Outgoing indexes do not provide incoming traversal.

Endpoint keys can be passed directly to model methods:

```python
assignment = Assignment(source=user.key, target=task.key, priority=3)
assignment.set()

loaded_user = User.get(assignment.source)
loaded_task = Task.get(assignment.target)
```

Constructors also accept endpoint model instances and extract their keys:

```python
assignment = Assignment(source=user, target=task)
assert assignment.source == user.key
assert assignment.target == task.key
```

Model instances must match the declared endpoint model or a subclass.
An instance of another model raises `TypeError`:

```python
Assignment(source=task, target=user)  # TypeError: wrong endpoint models.
```

Edge methods accept endpoint keys or corresponding model instances.
Source-scoped queries accept either form for the source. The same endpoint
checks apply to constructors, `get()`, `delete()`, and source-scoped queries.

Raw keys follow the declared key type without establishing model ownership.
Reading edges or passing endpoint keys does not fetch nodes, check existence,
or compare endpoint namespaces. Reference wrappers, embedded model fields, and
implicit fetches are not supported.

## Generated Keys

`Key` is an immutable, hashable, comparable identifier. `Key()` generates UUID7
when available in the standard library (Python 3.14 and later), otherwise
UUID4. Applications need not select or import a UUID generator.

`Key(text)` parses a standard UUID string:

```python
key = Key()
text = str(key)
restored = Key(text)

assert restored == key

account = Account(key=restored, name="Alice")
account.set()
same_account = Account.get(restored)
```

Parsing accepts UUID4 and UUID7 on every supported Python version. Reads
preserve stored keys, including after Python upgrades. JSON encoding and
reconstruction of `Key` fields are automatic.

Keys compare as unsigned 128-bit integers. UUID7 approximately follows
generation time; ordering across generators or mixed UUID versions does not
guarantee creation order. Use an `Ordering` on a timestamp field for time
ordering.

## Primary Keys

Nodes use `key`; edges use `(source, target)`. Declare a different node key
type to require a supplied identifier:

```python
class Document(Node):
    key: str
    title: str


document = Document(key="installation", title="Installation guide")
document.set()
found = Document.get("installation")
```

`get()` and `exists()` accept primary keys. Queries on other fields require an
`Ordering`. Primary keys cannot be `None`.

Do not change a stored record's key or an edge's endpoints. Identity is
mutable: a subsequent write uses the new identity and may leave the old record
and edges behind. It does not rename or move data.

## Namespaces and Storage Identity

`namespace` is an optional model storage prefix:

```python
class Note(Node):
    namespace = "notes_plugin"
    title: str
```

Model name and namespace identify a record type. Separate packages can use the
same model name with different namespaces in one Redis database. Omission uses
no prefix.

Edges may connect models in different namespaces; each endpoint model
identifies its own namespace. Namespaces provide neither authorization
boundaries nor Redis Cluster hash-slot assignments.

Renaming a model or changing its namespace changes its storage location without
migrating existing data. Changing index declarations does not rebuild stored
indexes.
