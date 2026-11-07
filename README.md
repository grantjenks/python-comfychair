# Comfy Chair

> “Cardinal Fang! Fetch…THE COMFY CHAIR!”
>
> Monty Python, [The Spanish Inquisition](https://python.mzonline.com/sketches/spanishinquisition/)

Typed nodes, edges, and indexes for Python, backed by Redis.

Comfy Chair stores records by key and relationships as records with their own
metadata. Type annotations define fields. Primary indexes provide lookup and
ordered traversal; declared orderings support queries on other fields.
Namespaces keep independent collections of data separate.

Edges are records too: they store source and target keys along with fields such
as a role, status, or priority. Their indexes support queries over one source's
outgoing edges, including ordered traversal and pagination.

**Development status:** design stage. The API documentation specifies the intended
behavior; an implementation is not yet available. Examples assume a configured
Redis connection.

## Documentation

Start with the [example](docs/example.md), then use the reference pages for each
part of the API.

- [Example](docs/example.md): A complete node, edge, write, and query walkthrough.
- [Models and fields](docs/models.md): Node, Edge, Field, Key, endpoint keys, and namespaces.
- [Store and retrieve](docs/storage.md): CRUD, defaults, refresh, get_or_create, write modes, and ETags.
- [Ordered indexes](docs/indexes.md): Primary traversal, predicates, ranges, first, last, count, and indexed types.
- [Pages and cursors](docs/pagination.md): Page sequences, continuation cursors, and automatic iteration.
- [Commits](docs/commits.md): Queued writes, Result handles, batch conditions, callable retries, and limits.
- [Retrying mutations](docs/mutations.md): Callbacks, working copies, conflict retries, and object updates.
- [Relationships and exclusive ownership](docs/relationships.md): Email claims, concurrent creation, and swapping owners.
- [Choices and validation](docs/validation.md): Literal and enum choices, validate(), and declaration checks.
- [Serialization](docs/serialization.md): JSON, custom types, ETags, and separate index encodings.
