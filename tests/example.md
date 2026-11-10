# Example

Source: [Example](../docs/example.md).

## Declare a User

No connection or binding setup is required with localhost Redis database 0.
Namespaces below isolate the example's records from other runs.

    >>> from datetime import datetime, timezone
    >>> from typing import Literal
    >>> from comfychair import Commit, Database, Edge, Field, Key, Node, Ordering
    >>> scope = 'example_' + str(Key())
    >>> def now():
    ...     return datetime.now(timezone.utc)
    >>> class User(Node):
    ...     namespace = scope
    ...     key: Key
    ...     name: str

## Declare a Task

    >>> class Task(Node):
    ...     namespace = scope
    ...     key: Key
    ...     title: str
    ...     created_at: datetime = Field(default_factory=now)
    ...     indexes = (Ordering(created_at),)

## Declare a Relationship

    >>> class Assignment(Edge[User, Task]):
    ...     namespace = scope
    ...     status: Literal['open', 'closed'] = Field(default='open')
    ...     priority: int = Field(default=0)
    ...     created_at: datetime = Field(default_factory=now)
    ...     indexes = (
    ...         Ordering(created_at), Ordering(priority), Ordering(status, created_at),
    ...     )

## Construct Records

    >>> user = User(name='Alice')
    >>> task = Task(title='Write the introduction')
    >>> assignment = Assignment(source=user, target=task, priority=3)
    >>> assignment.target == task.key
    True

## Save Together

    >>> with Commit() as batch:
    ...     _ = batch.set(user)
    ...     _ = batch.set(task)
    ...     _ = batch.set(assignment)
    >>> User.get(user.key).name
    'Alice'

## Edit a Task

    >>> task.title = 'Write the README introduction'
    >>> task.set()
    >>> Task.get(task.key).title
    'Write the README introduction'

## List Open Assignments Newest First

    >>> page = Assignment.page(
    ...     user, Assignment.status == 'open',
    ...     order_by=Assignment.created_at, reverse=True, limit=20,
    ... )
    >>> len(page)
    1
    >>> page[0].priority
    3
    >>> page[0].target == task.key
    True

Queries use primary indexes or declared orderings. Loading an edge does not
fetch endpoint nodes. The next lookup is explicit.

## Follow an Endpoint

    >>> Task.get(page[0].target).title
    'Write the README introduction'

## Cleanup

    >>> assignment.delete()
    True
    >>> task.delete()
    True
    >>> user.delete()
    True
    >>> Database.default().close()
