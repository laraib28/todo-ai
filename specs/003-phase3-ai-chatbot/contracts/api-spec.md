# API Specification - Phase III: AI Chat Endpoint

## Overview

This document defines the API contract for the AI-powered todo chatbot endpoint. The chat endpoint is the single entry point for all natural language interactions with the todo assistant.

---

## OpenAPI 3.0 Specification

```yaml
openapi: 3.0.3
info:
  title: Todo AI Chatbot API
  description: Natural language interface for todo task management
  version: 1.0.0
  contact:
    name: Todo API Support

servers:
  - url: http://localhost:8000/api
    description: Development server
  - url: https://api.todo.example.com/api
    description: Production server

paths:
  /chat:
    post:
      summary: Send chat message to AI assistant
      description: |
        Processes natural language messages and returns AI-generated responses.
        The assistant can create, update, complete, delete, and list todo tasks
        through natural language commands.
      operationId: sendChatMessage
      tags:
        - Chat
      security:
        - cookieAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/ChatRequest'
            examples:
              createTask:
                summary: Create a new task
                value:
                  message: "Create a task to buy groceries tomorrow"
              listTasks:
                summary: List tasks
                value:
                  message: "Show me all my high priority tasks"
              completeTask:
                summary: Complete a task
                value:
                  message: "Mark the groceries task as done"
              updateTask:
                summary: Update a task
                value:
                  message: "Change the priority of my meeting task to high"
      responses:
        '200':
          description: Successful chat response
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ChatResponse'
              examples:
                taskCreated:
                  summary: Task created successfully
                  value:
                    message: "I've created a new task: 'Buy groceries' with medium priority for tomorrow."
                    metadata:
                      action: "task_created"
                      task_id: 42
                taskListed:
                  summary: Tasks listed
                  value:
                    message: "You have 2 high priority tasks:\n1. Finish project proposal (due tomorrow)\n2. Call dentist (due today)"
                    metadata:
                      action: "tasks_listed"
                      count: 2
        '401':
          $ref: '#/components/responses/Unauthorized'
        '422':
          $ref: '#/components/responses/ValidationError'
        '429':
          $ref: '#/components/responses/RateLimitExceeded'
        '500':
          $ref: '#/components/responses/InternalServerError'

components:
  securitySchemes:
    cookieAuth:
      type: apiKey
      in: cookie
      name: access_token
      description: JWT token stored in HTTP-only cookie

  schemas:
    ChatRequest:
      type: object
      required:
        - message
      properties:
        message:
          type: string
          description: Natural language message from the user
          minLength: 1
          maxLength: 2000
          example: "Create a task to buy groceries tomorrow"
      additionalProperties: false

    ChatResponse:
      type: object
      required:
        - message
      properties:
        message:
          type: string
          description: AI-generated response to the user
          example: "I've created a new task: 'Buy groceries' with medium priority."
        metadata:
          type: object
          description: Optional metadata about the action performed
          properties:
            action:
              type: string
              description: Action type performed
              enum:
                - task_created
                - task_updated
                - task_deleted
                - task_completed
                - task_uncompleted
                - tasks_listed
                - no_action
              example: "task_created"
            task_id:
              type: integer
              description: ID of the task operated on (if applicable)
              example: 42
            count:
              type: integer
              description: Number of tasks returned (for list operations)
              example: 5
          additionalProperties: true

    ErrorResponse:
      type: object
      required:
        - detail
      properties:
        detail:
          type: string
          description: Human-readable error message
          example: "Invalid credentials"

    ValidationError:
      type: object
      required:
        - detail
      properties:
        detail:
          type: array
          items:
            type: object
            properties:
              loc:
                type: array
                items:
                  oneOf:
                    - type: string
                    - type: integer
                description: Location of the error
              msg:
                type: string
                description: Error message
              type:
                type: string
                description: Error type

  responses:
    Unauthorized:
      description: Authentication required or invalid token
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
          example:
            detail: "Not authenticated"

    ValidationError:
      description: Request validation failed
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ValidationError'
          example:
            detail:
              - loc: ["body", "message"]
                msg: "field required"
                type: "value_error.missing"

    RateLimitExceeded:
      description: Too many requests
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
          example:
            detail: "Rate limit exceeded. Please try again later."

    InternalServerError:
      description: Internal server error
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
          example:
            detail: "An unexpected error occurred. Please try again."
```

---

## Request Schema

### ChatRequest

```python
from pydantic import BaseModel, Field, validator

class ChatRequest(BaseModel):
    """Request schema for chat endpoint."""

    message: str = Field(
        ...,
        min_length=1,
        max_length=2000,
        description="Natural language message from the user"
    )

    @validator('message')
    def message_not_empty(cls, v):
        """Validate that message is not just whitespace."""
        if not v or not v.strip():
            raise ValueError('Message cannot be empty or whitespace only')
        return v.strip()

    class Config:
        schema_extra = {
            "example": {
                "message": "Create a task to buy groceries tomorrow"
            }
        }
```

---

## Response Schema

### ChatResponse

```python
from pydantic import BaseModel, Field
from typing import Optional, Literal

class ChatMetadata(BaseModel):
    """Metadata about the action performed."""

    action: Literal[
        "task_created",
        "task_updated",
        "task_deleted",
        "task_completed",
        "task_uncompleted",
        "tasks_listed",
        "no_action"
    ] = Field(description="Action type performed")
    task_id: Optional[int] = Field(default=None, description="ID of the task operated on")
    count: Optional[int] = Field(default=None, description="Number of tasks returned")

    class Config:
        schema_extra = {
            "example": {
                "action": "task_created",
                "task_id": 42
            }
        }


class ChatResponse(BaseModel):
    """Response schema for chat endpoint."""

    message: str = Field(description="AI-generated response to the user")
    metadata: Optional[ChatMetadata] = Field(default=None, description="Optional action metadata")

    class Config:
        schema_extra = {
            "example": {
                "message": "I've created a new task: 'Buy groceries' with medium priority.",
                "metadata": {
                    "action": "task_created",
                    "task_id": 42
                }
            }
        }
```

---

## Authentication

### Cookie-Based JWT Authentication

**Cookie Name**: `access_token`

**Cookie Properties**:
- `httponly`: true (prevents XSS access)
- `samesite`: "lax" (CSRF protection)
- `max_age`: 86400 (24 hours)
- `secure`: true (HTTPS only in production)

**JWT Claims**:
```json
{
  "sub": "user_id",
  "exp": 1704153600
}
```

**Authentication Flow**:
1. User logs in via `/api/auth/login`
2. Server sets JWT in `access_token` cookie
3. Browser automatically sends cookie with `/api/chat` requests
4. Server validates JWT and extracts user_id
5. All MCP tools receive user_id for row-level security

**Error Response** (401 Unauthorized):
```json
{
  "detail": "Not authenticated"
}
```

---

## Error Codes

| Status Code | Error Type | Description | Example |
|-------------|------------|-------------|---------|
| 200 | Success | Request processed successfully | Chat message processed |
| 401 | Unauthorized | Missing or invalid JWT token | User not logged in |
| 422 | Validation Error | Invalid request body | Empty message field |
| 429 | Rate Limit Exceeded | Too many requests | > 60 messages/minute |
| 500 | Internal Server Error | Unexpected error | OpenAI API failure |

---

## Rate Limiting

**Limits**:
- **Per User**: 60 requests per minute
- **Per IP**: 100 requests per minute (across all users)

**Rate Limit Headers**:
```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 59
X-RateLimit-Reset: 1704153600
```

**Rate Limit Response** (429):
```json
{
  "detail": "Rate limit exceeded. Please try again in 30 seconds."
}
```

---

## Examples

### Example 1: Create Task

**Request**:
```bash
curl -X POST http://localhost:8000/api/chat \
  -H "Content-Type: application/json" \
  -H "Cookie: access_token=eyJ..." \
  -d '{
    "message": "Create a task to buy groceries tomorrow with high priority"
  }'
```

**Response** (200 OK):
```json
{
  "message": "I've created a new task: 'Buy groceries' with high priority. It's due tomorrow.",
  "metadata": {
    "action": "task_created",
    "task_id": 42
  }
}
```

---

### Example 2: List Tasks

**Request**:
```bash
curl -X POST http://localhost:8000/api/chat \
  -H "Content-Type: application/json" \
  -H "Cookie: access_token=eyJ..." \
  -d '{
    "message": "Show me all my incomplete tasks"
  }'
```

**Response** (200 OK):
```json
{
  "message": "You have 3 incomplete tasks:\n1. Buy groceries (high priority, due tomorrow)\n2. Finish project proposal (medium priority, due Friday)\n3. Call dentist (low priority, no due date)",
  "metadata": {
    "action": "tasks_listed",
    "count": 3
  }
}
```

---

### Example 3: Complete Task

**Request**:
```bash
curl -X POST http://localhost:8000/api/chat \
  -H "Content-Type: application/json" \
  -H "Cookie: access_token=eyJ..." \
  -d '{
    "message": "Mark the groceries task as done"
  }'
```

**Response** (200 OK):
```json
{
  "message": "Great job! I've marked 'Buy groceries' as completed.",
  "metadata": {
    "action": "task_completed",
    "task_id": 42
  }
}
```

---

### Example 4: Update Task Priority

**Request**:
```bash
curl -X POST http://localhost:8000/api/chat \
  -H "Content-Type: application/json" \
  -H "Cookie: access_token=eyJ..." \
  -d '{
    "message": "Change the meeting task priority to high"
  }'
```

**Response** (200 OK):
```json
{
  "message": "I've updated the priority of 'Team meeting' to high.",
  "metadata": {
    "action": "task_updated",
    "task_id": 15
  }
}
```

---

### Example 5: Validation Error

**Request**:
```bash
curl -X POST http://localhost:8000/api/chat \
  -H "Content-Type: application/json" \
  -H "Cookie: access_token=eyJ..." \
  -d '{
    "message": ""
  }'
```

**Response** (422 Unprocessable Entity):
```json
{
  "detail": [
    {
      "loc": ["body", "message"],
      "msg": "Message cannot be empty or whitespace only",
      "type": "value_error"
    }
  ]
}
```

---

### Example 6: Authentication Error

**Request**:
```bash
curl -X POST http://localhost:8000/api/chat \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Create a task"
  }'
```

**Response** (401 Unauthorized):
```json
{
  "detail": "Not authenticated"
}
```

---

## FastAPI Implementation

### Router Setup

```python
from fastapi import APIRouter, Depends, HTTPException, status, Request
from sqlmodel import Session
from app.database import get_session
from app.auth import get_current_user
from app.models import User
from app.schemas import ChatRequest, ChatResponse
from app.services.chat_service import ChatService

router = APIRouter(prefix="/chat", tags=["chat"])


@router.post("", response_model=ChatResponse)
async def send_chat_message(
    request: ChatRequest,
    current_user: User = Depends(get_current_user),
    session: Session = Depends(get_session)
):
    """Process natural language chat message."""
    chat_service = ChatService(session, current_user.id)

    try:
        response = await chat_service.process_message(request.message)
        return response
    except Exception as e:
        # Log error
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail="An unexpected error occurred. Please try again."
        )
```

---

## Security Considerations

### Input Validation
- Maximum message length: 2000 characters
- Trim whitespace before processing
- Sanitize HTML entities to prevent XSS
- Validate JWT signature and expiration

### Row-Level Security
- All MCP tools receive user_id from JWT
- All database queries filtered by user_id
- No cross-user data access allowed

### Rate Limiting
- Per-user limit: 60 requests/minute
- Per-IP limit: 100 requests/minute
- Sliding window algorithm
- 429 response with Retry-After header

### Error Handling
- Never expose internal errors to client
- Log all errors with request ID
- Generic error messages for 500 errors
- Specific validation errors for 422 errors

---

## Testing Requirements

### Unit Tests
- [ ] Test ChatRequest validation (empty message, max length)
- [ ] Test ChatResponse serialization
- [ ] Test authentication dependency (valid/invalid JWT)
- [ ] Test rate limiting decorator

### Integration Tests
- [ ] Test POST /api/chat with valid JWT (200)
- [ ] Test POST /api/chat without JWT (401)
- [ ] Test POST /api/chat with invalid message (422)
- [ ] Test POST /api/chat rate limiting (429)
- [ ] Test conversation history saved to database

### End-to-End Tests
- [ ] Test full flow: login → chat → create task → verify task created
- [ ] Test multi-turn conversation context
- [ ] Test error handling (OpenAI API failure)

---

## Performance Requirements

- **Response Time**: p95 < 2 seconds
- **Throughput**: 100 requests/second
- **Availability**: 99.9% uptime
- **Error Rate**: < 0.1% (excluding user errors)

---

## Monitoring

### Metrics
- Request count by status code
- Response time (p50, p95, p99)
- OpenAI API latency
- Database query latency
- Rate limit rejections

### Alerts
- Error rate > 1% for 5 minutes
- p95 latency > 5 seconds for 5 minutes
- OpenAI API failure rate > 10%

---

## Acceptance Criteria

- [ ] OpenAPI 3.0 spec validates without errors
- [ ] POST /api/chat endpoint implemented
- [ ] JWT authentication enforced
- [ ] Rate limiting configured (60/min per user)
- [ ] Request/response validation working
- [ ] Error handling returns proper status codes
- [ ] All unit tests passing
- [ ] All integration tests passing
- [ ] API documentation generated from OpenAPI spec

---

## Dependencies

- FastAPI 0.104+
- Pydantic 2.0+
- PyJWT 2.8+
- slowapi (rate limiting)

---

## References

- [OpenAPI 3.0 Specification](https://swagger.io/specification/)
- [FastAPI Security](https://fastapi.tiangolo.com/tutorial/security/)
- [Pydantic Validation](https://docs.pydantic.dev/latest/)
- Phase III Spec: `/specs/003-phase3-ai-chatbot/spec.md`
- Phase III Plan: `/specs/003-phase3-ai-chatbot/plan.md`
- Data Model: `/specs/003-phase3-ai-chatbot/data-model.md`
