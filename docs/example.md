# Example

[Documentation](../README.md#documentation)

Uses Redis at `redis://localhost:6379/0`; no connection setup is required.
See [Database Configuration](database.md) to select another database.

## Models

```python
from datetime import datetime, timezone
from typing import Literal

from comfychair import Commit, Edge, Field, Key, Node, Ordering


def now():
    return datetime.now(timezone.utc)


class User(Node):
    key: Key
    name: str


class Task(Node):
    key: Key
    title: str
    created_at: datetime = Field(default_factory=now)

    indexes = (
        Ordering(created_at),
    )


class Assignment(Edge[User, Task]):
    status: Literal["open", "closed"] = Field(default="open")
    priority: int = Field(default=0)
    created_at: datetime = Field(default_factory=now)

    indexes = (
        Ordering(created_at),
        Ordering(priority),
        Ordering(status, created_at),
    )
```

## Writes

```python
user = User(name="Alice")
task = Task(title="Write the introduction")
assignment = Assignment(source=user, target=task, priority=3)

with Commit() as batch:
    batch.set(user)
    batch.set(task)
    batch.set(assignment)

same_user = User.get(user.key)
task.title = "Write the README introduction"
task.set()
```

## Queries

Open assignments in descending creation order:

```python
page = Assignment.page(
    user,
    Assignment.status == "open",
    order_by=Assignment.created_at,
    reverse=True,
    limit=20,
)

for assignment in page:
    print(assignment.target, assignment.priority)
```

Edges contain endpoint keys; reading an edge does not fetch nodes. Queries use
primary indexes or declared orderings.
