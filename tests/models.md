# Models and Fields

Source: [Models and Fields](../docs/models.md).

## Setup

    >>> from pytest import raises
    >>> from comfychair import Database, Edge, Field, Key, Node, Ordering
    >>> db = Database()
    >>> scope = 'models_' + str(Key())

## Declare Fields

Constructors accept keyword fields. Plain annotations and defaults are shorthand
for fields. Construction and assignment perform no storage I/O.

    >>> class Note(Node):
    ...     namespace = scope
    ...     title: str
    ...     priority: int = 0
    >>> db.bind(Note)
    >>> note = Note(title='Introduction')
    >>> note.title
    'Introduction'
    >>> note.priority
    0
    >>> note.etag is None
    True

## Configure a Field

Use an explicit Field to configure a factory or reference a field in the class
body. Field() without a default requires a value.

    >>> class Item(Node):
    ...     namespace = scope
    ...     title: str = Field()
    ...     priority: int = Field(default=0)
    ...     indexes = (Ordering(priority),)
    >>> db.bind(Item)
    >>> item = Item(title='Introduction')
    >>> item.priority
    0
    >>> isinstance(Item.priority, Field)
    True
    >>> with raises(TypeError):
    ...     Item()

Class access returns a descriptor; instance access returns a value.
Descriptors form [query predicates](indexes.md).

## Apply a Factory Per Instance

    >>> calls = []
    >>> def default_title():
    ...     calls.append('factory')
    ...     return 'Untitled'
    >>> class Draft(Node):
    ...     namespace = scope
    ...     title: str = Field(default_factory=default_title)
    >>> db.bind(Draft)
    >>> calls
    []
    >>> first = Draft()
    >>> second = Draft()
    >>> calls
    ['factory', 'factory']

## Load Through the Constructor

Reads call the normal constructor with decoded keyword fields, then attach the
stored ETag and database association. There are no extra lifecycle hooks.

    >>> constructed = []
    >>> class Observed(Node):
    ...     namespace = scope
    ...     title: str
    ...     def __init__(self, **fields):
    ...         super().__init__(**fields)
    ...         constructed.append(self.title)
    >>> db.bind(Observed)
    >>> original = Observed(title='Saved')
    >>> original.set()
    >>> constructed.clear()
    >>> loaded = Observed.get(original.key)
    >>> constructed
    ['Saved']
    >>> loaded is original
    False
    >>> loaded.etag == original.etag
    True

Overrides must accept declared keyword fields. Construction also occurs on
reads; it does not mean the record is new. Reads do not call application
validate(), but decoding checks representations and required identity/fields.

## Do Not Repeat Present Defaults

    >>> first.set()
    >>> calls.clear()
    >>> restored = Draft.get(first.key)
    >>> restored.title
    'Untitled'
    >>> calls
    []

Missing optional fields use defaults or factories. Missing required fields
fail. Loading never generates replacement identity. See [Schema Changes](adapters.md).

## Declare Typed Endpoints

    >>> class User(Node):
    ...     namespace = scope
    ...     name: str
    >>> class Assignment(Edge[User, Note]):
    ...     namespace = scope
    ...     priority: int = 0
    >>> db.bind(User, Assignment)
    >>> user = User(name='Alice')
    >>> assignment = Assignment(source=user, target=note)
    >>> assignment.source == user.key
    True
    >>> assignment.target == note.key
    True
    >>> assignment.key == (user.key, note.key)
    True

## Accept Keys Without Fetching Nodes

    >>> assignment = Assignment(source=user.key, target=note.key)
    >>> assignment.set()
    >>> User.exists(user.key)
    False
    >>> Note.exists(note.key)
    False
    >>> Assignment.get(user, note).target == note.key
    True

Endpoint fields use their declared models' key types. Raw keys do not establish
model ownership. No endpoint fetch, existence check, namespace comparison,
reference wrapper, embedded model field, or recursive persistence is implied.

## Reject the Wrong Endpoint Model

The same endpoint checks apply to constructors, lookup, deletion, and
source-scoped queries. Subclasses of the declared endpoint model are accepted.

    >>> with raises(TypeError):
    ...     Assignment(source=note, target=user)
    >>> with raises(TypeError):
    ...     Assignment.get(note, note)
    >>> with raises(TypeError):
    ...     Assignment.page(note, limit=1)

## Accept an Endpoint Subclass

    >>> class Member(User):
    ...     namespace = scope + '_members'
    >>> member = Member(name='Member')
    >>> Assignment(source=member, target=note).source == member.key
    True

Different endpoint namespaces do not prevent an edge. Model types, rather than
the namespaces or existence of raw keys, determine instance endpoint checks.

## Cleanup

    >>> for record in Assignment.iterate(user.key, batch_size=100):
    ...     _ = record.delete()
    >>> for model in (User, Observed, Draft, Item, Note):
    ...     for record in model.iterate(batch_size=100):
    ...         _ = record.delete()
    >>> db.close()
