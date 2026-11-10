# Database Selection

Source: [Database Configuration](../docs/database.md).

## Setup

These examples need independent Redis databases 0 and 1 on one local primary.

    >>> from comfychair import Database, Edge, Key, Node, Commit, IF_NOT_MODIFIED
    >>> original_default = Database.default()
    >>> primary = Database('redis://localhost:6379/0')
    >>> archive = Database('redis://localhost:6379/1')
    >>> scope = 'database_' + str(Key())
    >>> class Note(Node):
    ...     namespace = scope
    ...     title: str

## Select the Process Default

Database() creates a distinct object; it does not replace the shared default.
Constructing or selecting a database opens no connection.

    >>> Database.default() is original_default
    True
    >>> primary is original_default
    False
    >>> primary.set_default() is None
    True
    >>> Database.default() is primary
    True

The implicit default is redis://localhost:6379/0. Redis must be running; the
package does not start it. The first storage operation opens the connection.

## Use an Unbound Model

    >>> note = Note(title='Primary')
    >>> note.set()
    >>> Note.get(note.key, using=primary).title
    'Primary'

## Bind a Model

    >>> archive.bind(Note)
    >>> archived = Note(key=note.key, title='Archive')
    >>> archived.set()
    >>> Note.get(note.key).title
    'Archive'

Bind supplies a default, not exclusive ownership. Rebinding replaces the
binding without I/O, migration, or changing existing instance associations.
Rebinding to the same selection is a no-op.

## Prefer an Instance Association

    >>> note.title = 'Primary edit'
    >>> note.set()
    >>> Note.get(note.key, using=primary).title
    'Primary edit'
    >>> Note.get(note.key, using=archive).title
    'Archive'

## Prefer Explicit Using

Writing to another database preserves the key and original stored record.

    >>> note.title = 'Copied'
    >>> note.set(using=archive)
    >>> Note.get(note.key, using=primary).title
    'Primary edit'
    >>> Note.get(note.key, using=archive).title
    'Copied'

## Remember the Successful Destination

    >>> note.title = 'Remembered'
    >>> note.set()
    >>> Note.get(note.key, using=archive).title
    'Remembered'

One instance carries one association and one ETag. Use separate loaded objects
to work independently with the same identity in different databases. Successful
reads and confirmed writes associate their selected database. Failed or
unconfirmed operations leave the association unchanged.

## Leave Unsaved Instances Unassociated

    >>> class Unbound(Node):
    ...     namespace = scope
    >>> pending = Unbound()
    >>> archive.set_default()
    >>> pending.set()
    >>> Unbound.exists(pending.key, using=archive)
    True
    >>> Unbound.exists(pending.key, using=primary)
    False

Changing the default does not override bindings or remembered associations.
It neither connects nor closes the previous default. Configuration is
process-wide, not ambient, thread-local, or context-local; configure at startup.

## Compare Tokens Across Connection Objects

    >>> second_connection = Database('redis://localhost:6379/0')
    >>> current = Note.get(note.key, using=primary)
    >>> current.title = 'Conditional'
    >>> current.set(mode=IF_NOT_MODIFIED, using=second_connection)
    >>> Note.get(note.key, using=primary).title
    'Conditional'

ETags compare against the selected store without reader provenance. A missing
record or mismatched token conflicts. Refresh and mutate use the same routing.

## Decode Without a Database Association

From-json selects adapters without associating the decoded object with that
database. A subsequent write still uses its model binding.

    >>> detached = Note.from_json(current.to_json(), using=primary)
    >>> detached.title = 'Decoded'
    >>> detached.set()
    >>> Note.get(detached.key, using=archive).title
    'Decoded'
    >>> Note.get(detached.key, using=primary).title
    'Conditional'

## Remember an Absent Delete's Destination

    >>> absent = Note(title='Absent')
    >>> absent.delete(using=primary)
    False
    >>> absent.set()
    >>> Note.exists(absent.key, using=primary)
    True
    >>> Note.exists(absent.key, using=archive)
    False

## Route Edges Independently

    >>> class Link(Edge[Note, Note]):
    ...     namespace = scope
    >>> archive.bind(Link)
    >>> link = Link(source=absent, target=current)
    >>> link.set()
    >>> Link.exists(absent.key, current.key, using=archive)
    True
    >>> Link.exists(absent.key, current.key, using=primary)
    False

Endpoints are plain keys. Passing model instances extracts keys but does not
select a database, fetch endpoints, or check existence. Endpoint reads route
independently.

## Route a Whole Commit

    >>> with Commit(using=primary) as batch:
    ...     _ = batch.set(link)
    >>> Link.exists(absent.key, current.key, using=primary)
    True

Without explicit using, all queued selections must be the same Database object,
even when URLs match. Conflicting selections reject before writes. Model-and-key
deletes use the model binding/default. Batch methods accept no per-operation
using. Empty batches do nothing. Unbound, unassociated records share the default.
Explicit commit routing does not route ordinary calls, including retry reads.

## Routing Rules

All model and instance storage methods accept keyword-only using: get, exists,
get_or_create, page, iterate, first, last, count, set, add, delete, refresh,
and mutate. Selection is explicit using, instance association, model binding,
then current default. Pass database objects, not string aliases. Failure raises
StorageError from that selection; there is no fallback database.

## Cleanup

    >>> for database in (primary, archive):
    ...     for record in Link.iterate(absent.key, using=database, batch_size=100):
    ...         _ = record.delete(using=database)
    ...     for model in (Note, Unbound):
    ...         for record in model.iterate(using=database, batch_size=100):
    ...             _ = record.delete(using=database)
    >>> original_default.set_default()
    >>> primary.close()
    >>> archive.close()
    >>> second_connection.close()
