# dev-standards

A Claude Code plugin for Python and TypeScript development with opinionated coding standards.

## Installation

```bash
claude --plugin-dir /path/to/this/folder
```

Or add to your Claude Code settings.

## Skills

### python-style

Triggers when writing Python code. Provides standards for:

- **Functions over classes** - Service functions with explicit dependencies, not service classes
- **Pydantic for DTOs** - Use `BaseModel` with `frozen=True` for immutable data
- **Dependency injection** - Pass dependencies as function parameters, use `functools.partial`
- **Error handling** - Crash early, validate at boundaries, never swallow exceptions
- **File organization** - Functions under 50 lines, files under 800 lines, nesting ≤ 4

### rest-api

Triggers when designing REST APIs. Provides patterns for:

- **Layered architecture** - Transport DTOs → Service Functions → Database Models
- **Extensible responses** - Never return bare arrays, wrap in objects
- **Decoupled resources** - API resources separate from database models
- **Direct DB access** - Service functions access database directly (no repository pattern)

### code-review

Triggers on code review requests. Quick checklist covering readability, correctness, design, and Python/API specifics.

## Hooks

Auto-formatting hooks run after every file edit:

| File Type | Formatter |
|-----------|-----------|
| `.py` | `uv run ruff format` |
| `.ts`, `.tsx` | `yarn run prettier --write` |
| `.md` | `yarn run prettier --write` |

## Key Principles

```python
# Functions over classes
def create_user(
    request: CreateUserRequest,
    db_session: Session,
    notifier: Notifier,
) -> User:
    user = User(email=request.email)
    db_session.add(user)
    notifier.send_welcome(user)
    return user

# Use partial for DI
from functools import partial
create_user_fn = partial(create_user, db_session=session, notifier=notifier)
```
