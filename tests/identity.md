# Keys and Namespaces

Source: [Models and Fields](../docs/models.md).

## Setup

    >>> import uuid
    >>> from pytest import raises
    >>> from comfychair import Database, Field, Key, Node
    >>> db = Database()
    >>> scope = 'identity_' + str(Key())

## Generate a Key

Key is immutable and hashable. Generation uses standard-library UUID7 when
available, otherwise UUID4. It performs no storage access.

    >>> key = Key()
    >>> uuid.UUID(str(key)).version == (7 if hasattr(uuid, 'uuid7') else 4)
    True
    >>> Key(str(key)) == key
    True
    >>> {key: 'saved'}[Key(str(key))]
    'saved'

## Read Either UUID Version

Parsing does not depend on the generator available in the running Python.

    >>> v4 = '00000000-0000-4000-8000-000000000001'
    >>> v7 = '00000000-0000-7000-8000-000000000001'
    >>> str(Key(v4)) == v4
    True
    >>> str(Key(v7)) == v7
    True
    >>> Key(v4) < Key(v7)
    True

Order is unsigned 128-bit numeric order. UUID7 is approximately chronological;
mixed versions and separate generators do not promise creation order. Use a
timestamp Ordering when chronological traversal is required.

## Inherit the Key Default

    >>> class Inherited(Node):
    ...     namespace = scope
    >>> class Declared(Node):
    ...     namespace = scope
    ...     key: Key
    >>> class Explicit(Node):
    ...     namespace = scope
    ...     key: Key = Field(default_factory=Key)
    >>> all(isinstance(model().key, Key) for model in (Inherited, Declared, Explicit))
    True

## Preserve a Supplied Key

    >>> db.bind(Declared)
    >>> record = Declared(key=Key(v4))
    >>> record.set()
    >>> str(Declared.get(record.key).key)
    '00000000-0000-4000-8000-000000000001'

Reads preserve stored identity across Python upgrades. Key fields have built-in
JSON encoding and decoding.

## Supply a String Key

    >>> class Document(Node):
    ...     namespace = scope
    ...     key: str
    ...     title: str
    >>> db.bind(Document)
    >>> document = Document(key='installation', title='Install')
    >>> document.set()
    >>> Document.get('installation').title
    'Install'
    >>> with raises(TypeError):
    ...     Document(title='Missing identity')

Primary keys cannot be None. Node identity is its key; edge identity is its
source-target pair, within the model and namespace.

## Changing a Key Does Not Rename Storage

    >>> document.key = 'setup'
    >>> document.set()
    >>> Document.exists('installation')
    True
    >>> Document.exists('setup')
    True

Do not change saved keys or edge endpoints. Identity is mutable; writing the
new identity can strand the old record and relationships.

## Separate Equal Model Names

    >>> class Note(Node):
    ...     namespace = scope + '_one'
    ...     key: str
    ...     title: str
    >>> FirstNote = Note
    >>> class Note(Node):
    ...     namespace = scope + '_two'
    ...     key: str
    ...     title: str
    >>> SecondNote = Note
    >>> db.bind(FirstNote, SecondNote)
    >>> FirstNote(key='intro', title='One').set()
    >>> SecondNote(key='intro', title='Two').set()
    >>> FirstNote.get('intro').title
    'One'
    >>> SecondNote.get('intro').title
    'Two'

Namespace is an optional storage prefix. Omission uses no prefix. It is neither
an authorization boundary nor a Redis Cluster slot assignment. Cross-namespace
edges are allowed. Model renames and namespace changes do not migrate data;
index declaration changes do not rebuild indexes.

## Cleanup

    >>> for model in (Declared, Document, FirstNote, SecondNote):
    ...     for record in model.iterate(batch_size=100):
    ...         _ = record.delete()
    >>> db.close()
