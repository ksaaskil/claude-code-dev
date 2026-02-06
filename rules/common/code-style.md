# Code Style

## Structure
- Functions over classes for business logic
- Pass dependencies as function parameters
- Single responsibility per function

## Size Limits
- Functions: <50 lines
- Files: <800 lines (target 200-400)
- Nesting: max 4 levels

## Error Handling
- Raise errors, don't swallow them
- Validate at system boundaries
- Log before re-raising

## Immutability
- Create new objects instead of mutating
- Never mutate input parameters

## Readability
- Code reads like a story
- Self-documenting names
- Avoid unnecessary comments

See `/code-review` for review checklist.
