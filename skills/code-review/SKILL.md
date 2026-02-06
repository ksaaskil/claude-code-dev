---
name: code-review
description: This skill should be used when the user asks to "review code", "check my code", "review this file", "check for issues", or before completing a feature. Provides a quick checklist for code quality.
---

# Code Review Checklist

## Quick Review

### Readability

- [ ] Code reads like a story with clear intent
- [ ] Names are self-documenting
- [ ] No unnecessary comments (code explains itself)
- [ ] Functions under 50 lines
- [ ] Nesting depth ≤ 4 levels
- [ ] Files under 800 lines

### Correctness

- [ ] Errors are raised, not swallowed
- [ ] Input validated at boundaries
- [ ] No mutation of input parameters
- [ ] Edge cases handled

### Design

- [ ] Functions preferred over classes for business logic
- [ ] Dependencies passed as function parameters
- [ ] No monkey-patching in tests
- [ ] Single responsibility per function
- [ ] Interfaces use Protocol when needed
- [ ] Immutable data structures where possible

### Python Specifics

- [ ] Pydantic for DTOs (not dataclasses)
- [ ] Type hints on all public functions
- [ ] Context managers for resources
- [ ] Generators for large data iteration
- [ ] No mutable default arguments

### API Specifics (if applicable)

- [ ] Responses wrapped in objects
- [ ] DTOs separate from database models
- [ ] Consistent error format
- [ ] Appropriate HTTP status codes

## Red Flags

Stop and request changes if you see:

| Issue | Why It's Bad |
|-------|--------------|
| Service classes | Use service functions instead |
| Deep nesting (> 4 levels) | Hard to read and test |
| Functions over 50 lines | Too much responsibility |
| Files over 800 lines | Should be split |
| Mutable default args | Shared state bug |
| Global state mutation | Unpredictable behavior |
| Monkey-patching | Brittle tests |
| Dataclasses for API DTOs | Use Pydantic instead |
| Bare arrays in API responses | Not extensible |
| Business logic in views | Belongs in service functions |

## Review Process

1. **Understand the intent**: Read the PR description and understand what problem is being solved
2. **Check structure**: Verify file organization and separation of concerns
3. **Review logic**: Check correctness, error handling, and edge cases
4. **Verify tests**: Ensure adequate test coverage with proper mocking
5. **Check style**: Confirm adherence to coding standards
