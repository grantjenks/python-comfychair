# Example

[README](../README.md) · [Documentation](../README.md#documentation)

Examples assume a configured Redis connection.

Define nodes and a typed relationship between them:

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

Create records and store related writes together:

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

List a user's open assignments, newest first:

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

An edge stores its endpoint keys. Reading an edge does not fetch either node.
Every query uses an automatic primary index or a declared ordering.
