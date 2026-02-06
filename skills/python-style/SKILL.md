---
name: python-style
description: This skill should be used when the user asks to "write Python code", "create a Python file", "implement a Python class", "add a Python function", or when working on any Python (.py) files. Provides coding standards including immutability, Pydantic DTOs, Protocol interfaces, dependency injection, and error handling.
---

# Python Development Standards

## Core Principles

### Functions Over Classes

Prefer plain functions over classes. Classes are appropriate for data containers (Pydantic models) and when you need Protocol interfaces, but business logic should live in functions.

```python
# Bad - unnecessary class
class UserService:
    def __init__(self, db_session: Session) -> None:
        self._db_session = db_session

    def create_user(self, request: CreateUserRequest) -> User:
        ...

# Good - plain function with explicit dependencies
def create_user(
    request: CreateUserRequest,
    db_session: Session,
    notifier: Notifier,
) -> User:
    user = User(email=request.email)
    db_session.add(user)
    notifier.send_welcome(user)
    return user
```

Use `functools.partial` to create pre-configured versions:

```python
from functools import partial

# Create a partially applied version
create_user_fn = partial(create_user, db_session=session, notifier=notifier)

# Call with just the request
user = create_user_fn(request)
```

### Immutability First

Create new objects instead of mutating existing ones. This prevents side effects, aids debugging, and supports concurrent operations.

```python
# Bad - mutating
def process_items(items: list[Item]) -> list[Item]:
    for item in items:
        item.status = "processed"  # Mutation!
    return items

# Good - creating new
def process_items(items: list[Item]) -> list[Item]:
    return [item.model_copy(update={"status": "processed"}) for item in items]
```

### File Organization

- Target: 200-400 lines per file
- Maximum: 800 lines (split if exceeded)
- Functions: Under 50 lines each
- Nesting depth: Maximum 4 levels
- Structure by feature/domain rather than type

### Code Should Read Like a Story

Write code with clear intent and logical flow. Decompose problems into manageable pieces. Abstract complexity in functions while maintaining readability at the high level.

## Patterns

### Interfaces: Use Protocol (Duck Typing)

Define interfaces using structural subtyping when you need to abstract behavior:

```python
from typing import Protocol

class Notifier(Protocol):
    def send(self, recipient: str, message: str) -> None: ...

class EmailSender(Protocol):
    def send_email(self, to: str, subject: str, body: str) -> None: ...
```

### Data Transfer Objects: Use Pydantic

Use `BaseModel` with `frozen=True` for immutable DTOs:

```python
from pydantic import BaseModel, Field

class UserDTO(BaseModel):
    model_config = {"frozen": True}

    id: str
    email: str = Field(..., description="User email address")
    name: str
    created_at: datetime
```

For request/response models:

```python
class CreateUserRequest(BaseModel):
    email: str
    password: str

class UserResponse(BaseModel):
    model_config = {"frozen": True}

    id: str
    email: str
    name: str
```

### Resource Management: Context Managers

Use context managers for any resource that needs cleanup:

```python
from contextlib import contextmanager

@contextmanager
def database_transaction(conn: Connection):
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
```

### Lazy Evaluation: Generators

Use generators for memory-efficient iteration over large datasets:

```python
def process_large_dataset(path: Path) -> Iterator[ProcessedItem]:
    with open(path) as f:
        for line in f:
            yield process_line(line)
```

## Dependency Injection

Pass dependencies as function parameters. Use `functools.partial` to create pre-configured versions.

```python
from functools import partial

# Service function with explicit dependencies
def get_user(
    user_id: str,
    db_session: Session,
) -> User:
    user = db_session.query(User).get(user_id)
    if user is None:
        raise UserNotFoundError(user_id)
    return user

# Create pre-configured version for use in routes
get_user_fn = partial(get_user, db_session=session)
```

Benefits:
- Self-documenting about requirements
- Simple to unit test - just pass mock dependencies
- No monkey-patching needed

## Error Handling

### Crash Early - Raise, Don't Swallow

Never silently catch exceptions. Let failures surface through monitoring.

```python
# Bad - swallowing
def get_user(user_id: str, db_session: Session) -> User | None:
    try:
        return db_session.query(User).get(user_id)
    except Exception:
        return None  # Lost information!

# Good - explicit
def get_user(user_id: str, db_session: Session) -> User:
    user = db_session.query(User).get(user_id)
    if user is None:
        raise UserNotFoundError(f"User {user_id} not found")
    return user
```

### Validate at Boundaries

Validate all external input at system boundaries using Pydantic:

```python
from pydantic import BaseModel, field_validator

class CreateUserRequest(BaseModel):
    email: str
    password: str

    @field_validator("email")
    @classmethod
    def validate_email(cls, v: str) -> str:
        if "@" not in v:
            raise ValueError("Invalid email format")
        return v.lower()

    @field_validator("password")
    @classmethod
    def validate_password(cls, v: str) -> str:
        if len(v) < 8:
            raise ValueError("Password must be at least 8 characters")
        return v
```

### Log Before Re-raising

When catching exceptions to add context, log before re-raising:

```python
import logging

logger = logging.getLogger(__name__)

def process_order(order_id: str, db_session: Session) -> Order:
    try:
        return _do_process(order_id, db_session)
    except Exception:
        logger.exception(f"Failed to process order {order_id}")
        raise
```

## Pre-Completion Checklist

Before completing any Python code, verify:

- [ ] Functions preferred over classes for business logic
- [ ] Dependencies passed as function parameters
- [ ] Code is readable with clear naming
- [ ] All functions under 50 lines
- [ ] Files under 800 lines
- [ ] Nesting depth ≤ 4 levels
- [ ] Errors are raised, not swallowed
- [ ] No mutation of input parameters
- [ ] Pydantic models for DTOs (not dataclasses)
- [ ] Type hints on all public functions
- [ ] No hardcoded values (use constants/config)

## Red Flags

Stop and refactor if you see:
- Service classes (use functions instead)
- Deep nesting (> 4 levels)
- Functions over 50 lines
- Files over 800 lines
- Mutable default arguments (`def foo(items=[])`)
- Global state modification
- Monkey-patching in tests
