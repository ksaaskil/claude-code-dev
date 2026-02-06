# Python Code Style

## Data Models
- Pydantic `BaseModel` for DTOs, not dataclasses
- Use `frozen=True` for immutable models
- Separate request/response models

## Interfaces
- `Protocol` for duck typing (structural subtyping)
- No abstract base classes unless necessary

## Type Safety
- Type hints on all public functions
- No `Any` unless truly dynamic

## Patterns
- Context managers for resource cleanup
- Generators for large data iteration
- `functools.partial` for dependency injection

## Avoid
- Mutable default arguments (`def foo(items=[])`)
- Service classes (use functions)
- Global state mutation
- Monkey-patching in tests

See `/python-style` for examples and details.
