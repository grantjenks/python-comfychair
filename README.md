# Comfy Chair

> “Cardinal Fang! Fetch…THE COMFY CHAIR!”
>
> Monty Python, [The Spanish Inquisition](https://python.mzonline.com/sketches/spanishinquisition/)

Typed nodes, edges, and indexes for Python, backed by Redis.

Nodes store records by key. Edges store directed relationships by source and
target keys, with fields for relationship metadata. Type annotations define
fields. Primary indexes support lookup and ordered traversal; declared
orderings support queries on other fields. Edge queries are scoped to a source.
Namespaces distinguish model storage.

**Status:** API design. No implementation is available. Examples assume a
configured Redis connection.

## Documentation

- [Example](docs/example.md)
- [Models and fields](docs/models.md)
- [Store and retrieve](docs/storage.md)
- [Ordered indexes](docs/indexes.md)
- [Pages and cursors](docs/pagination.md)
- [Commits](docs/commits.md)
- [Mutations](docs/mutations.md)
- [Relationships and exclusive ownership](docs/relationships.md)
- [Choices and validation](docs/validation.md)
- [Serialization](docs/serialization.md)
