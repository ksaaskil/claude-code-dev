---
name: rest-api
description: This skill should be used when the user asks to "create an API", "design REST endpoints", "implement API routes", "add an endpoint", "create a REST service", or when working on HTTP API code. Provides API design patterns including layered architecture and service functions.
---

# REST API Design Standards

## Core Principles

### Document with OpenAPI

Every API should have an OpenAPI specification. Maintain documentation that includes:
- All endpoints with methods
- Request/response schemas
- Error responses
- Authentication requirements

Design the API contract before implementation to enable discussions and keep database models decoupled from REST resources.

### Return Extensible Objects

Never return bare arrays. Always wrap responses in objects to enable adding metadata without breaking clients:

```python
# Bad - not extensible
@app.get("/users")
async def list_users() -> list[UserDTO]:
    return users

# Good - extensible
class UserListResponse(BaseModel):
    data: list[UserDTO]
    total: int
    next_cursor: str | None = None

@app.get("/users")
async def list_users() -> UserListResponse:
    return UserListResponse(data=users, total=len(users))
```

### Decouple Resources from Data Models

API resources are NOT database models. Map between them to isolate clients from schema changes:

```python
# Database model (internal)
class UserModel:
    id: UUID
    email: str
    password_hash: str  # Never expose!
    created_at: datetime
    internal_flags: int

# API resource (external)
class UserResource(BaseModel):
    id: str
    email: str
    created_at: str  # ISO format string
    # No password_hash, no internal_flags
```

## Architecture Layers

```
┌─────────────────────────────┐
│   Transport Layer (DTOs)    │  <- Pydantic models for API
├─────────────────────────────┤
│   Service Functions         │  <- Business logic + DB access
├─────────────────────────────┤
│      Database Models        │  <- ORM/DB specific
└─────────────────────────────┘
```

### Transport Objects (DTOs)

Define separate models for API requests and responses:

```python
class CreateUserRequest(BaseModel):
    email: str
    password: str

class UserResponse(BaseModel):
    model_config = {"frozen": True}

    id: str
    email: str
    created_at: str

    @classmethod
    def from_model(cls, user: User) -> "UserResponse":
        return cls(
            id=str(user.id),
            email=user.email,
            created_at=user.created_at.isoformat(),
        )
```

### Service Functions

Keep business logic in functions with explicit dependencies. Access the database directly - no need for repository abstraction in most cases:

```python
def create_user(
    request: CreateUserRequest,
    db_session: Session,
) -> UserResponse:
    user = User(email=request.email)
    user.set_password(request.password)
    db_session.add(user)
    db_session.flush()
    return UserResponse.from_model(user)

def get_user(
    user_id: str,
    db_session: Session,
) -> UserResponse:
    user = db_session.query(User).get(user_id)
    if user is None:
        raise UserNotFoundError(user_id)
    return UserResponse.from_model(user)

def list_users(
    db_session: Session,
    limit: int = 20,
    cursor: str | None = None,
) -> UserListResponse:
    query = db_session.query(User).order_by(User.created_at)
    if cursor:
        query = query.filter(User.id > cursor)
    users = query.limit(limit + 1).all()

    has_more = len(users) > limit
    users = users[:limit]

    return UserListResponse(
        data=[UserResponse.from_model(u) for u in users],
        total=db_session.query(User).count(),
        next_cursor=str(users[-1].id) if has_more else None,
    )
```

### Views Integration

Views call service functions with dependencies injected:

```python
from functools import partial

# Create pre-configured service functions
def get_services(db_session: Session):
    return {
        "create_user": partial(create_user, db_session=db_session),
        "get_user": partial(get_user, db_session=db_session),
    }

@app.post("/users", status_code=201)
async def create_user_endpoint(
    request: CreateUserRequest,
    db_session: Session = Depends(get_db_session),
) -> UserResponse:
    return create_user(request, db_session)

@app.get("/users/{user_id}")
async def get_user_endpoint(
    user_id: str,
    db_session: Session = Depends(get_db_session),
) -> UserResponse:
    return get_user(user_id, db_session)
```

## Response Conventions

### Success Responses

- `200 OK` - GET, PUT, PATCH with body
- `201 Created` - POST creating resource (include Location header)
- `204 No Content` - DELETE, PUT/PATCH without body

### Error Responses

Use consistent error format:

```python
class ErrorResponse(BaseModel):
    error: str
    code: str
    details: dict[str, Any] | None = None

# Example response
{
    "error": "Validation failed",
    "code": "VALIDATION_ERROR",
    "details": {"email": "Invalid format"}
}
```

### HTTP Status Codes

- `400 Bad Request` - Invalid input
- `401 Unauthorized` - Missing/invalid authentication
- `403 Forbidden` - Authenticated but not allowed
- `404 Not Found` - Resource doesn't exist
- `409 Conflict` - Resource state conflict
- `422 Unprocessable Entity` - Validation errors
- `500 Internal Server Error` - Unexpected errors

## Pre-Completion Checklist

Before completing API code:

- [ ] OpenAPI documentation defined
- [ ] Responses wrapped in objects (not bare arrays)
- [ ] DTOs separate from database models
- [ ] Service functions (not classes) for business logic
- [ ] Dependencies passed as function parameters
- [ ] Consistent error response format
- [ ] Appropriate HTTP status codes
