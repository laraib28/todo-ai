# MCP Tools Specification - Phase III: AI Chat Tools

## Overview

This document defines the MCP (Model Context Protocol) tools that the OpenAI Agent uses to interact with the todo database. All tools enforce row-level security by filtering operations to the authenticated user's data only.

---

## Tool Architecture

```
┌─────────────────────────────────────────┐
│        OpenAI Agent (GPT-4)             │
│  - Interprets natural language          │
│  - Decides which tools to call          │
│  - Formats responses                    │
└──────────────┬──────────────────────────┘
               │
               │ Function Calls
               │
┌──────────────▼──────────────────────────┐
│         MCP Server (FastAPI)            │
│  - Validates user_id from JWT           │
│  - Enforces row-level security          │
│  - Executes database operations         │
└──────────────┬──────────────────────────┘
               │
               │ SQL Queries
               │
┌──────────────▼──────────────────────────┐
│      Neon PostgreSQL Database           │
│  - tasks table (user_id filtered)       │
│  - conversation_history table           │
└─────────────────────────────────────────┘
```

---

## Security Model

### User Context Propagation

All MCP tools receive the authenticated user's ID from the JWT token:

```python
# JWT decoded in FastAPI dependency
current_user: User = Depends(get_current_user)

# User ID passed to MCP server
mcp_server.set_user_context(user_id=current_user.id)

# All tool calls automatically filtered by user_id
result = await mcp_server.call_tool("create_task", {...})
```

### Row-Level Security

**Guarantee**: No tool can access or modify data belonging to other users.

**Implementation**:
- All database queries include `WHERE user_id = {current_user_id}`
- User ID validated on every tool call
- Unauthorized access returns error, never null data

---

## MCP Tools

### 1. create_task

**Description**: Create a new todo task for the authenticated user.

**Function Signature**:
```python
async def create_task(
    title: str,
    description: str = "",
    priority: str = "medium",
    user_id: int = None  # Injected from JWT
) -> dict
```

**JSON Schema** (MCP Format):
```json
{
  "name": "create_task",
  "description": "Create a new todo task with title, optional description, and priority",
  "inputSchema": {
    "type": "object",
    "required": ["title"],
    "properties": {
      "title": {
        "type": "string",
        "description": "Task title (1-200 characters)",
        "minLength": 1,
        "maxLength": 200
      },
      "description": {
        "type": "string",
        "description": "Task description (optional, max 2000 characters)",
        "maxLength": 2000,
        "default": ""
      },
      "priority": {
        "type": "string",
        "description": "Task priority level",
        "enum": ["low", "medium", "high"],
        "default": "medium"
      }
    }
  }
}
```

**Return Type**:
```json
{
  "success": true,
  "task": {
    "id": 42,
    "title": "Buy groceries",
    "description": "Milk, eggs, bread",
    "priority": "high",
    "is_complete": false,
    "created_at": "2026-01-02T10:30:00Z",
    "updated_at": "2026-01-02T10:30:00Z"
  }
}
```

**Error Response**:
```json
{
  "success": false,
  "error": "Title cannot be empty"
}
```

**Implementation Example**:
```python
from sqlmodel import Session, select
from app.models import Task

async def create_task(
    title: str,
    description: str = "",
    priority: str = "medium",
    user_id: int = None
) -> dict:
    """Create a new task for the user."""
    if not user_id:
        return {"success": False, "error": "Unauthorized"}

    # Validate priority
    if priority not in ["low", "medium", "high"]:
        return {"success": False, "error": "Invalid priority. Must be: low, medium, or high"}

    # Create task
    task = Task(
        user_id=user_id,
        title=title.strip(),
        description=description.strip(),
        priority=priority
    )

    session.add(task)
    session.commit()
    session.refresh(task)

    return {
        "success": True,
        "task": {
            "id": task.id,
            "title": task.title,
            "description": task.description,
            "priority": task.priority,
            "is_complete": task.is_complete,
            "created_at": task.created_at.isoformat(),
            "updated_at": task.updated_at.isoformat()
        }
    }
```

---

### 2. list_tasks

**Description**: List tasks for the authenticated user with optional filters.

**Function Signature**:
```python
async def list_tasks(
    is_complete: bool | None = None,
    priority: str | None = None,
    limit: int = 50,
    user_id: int = None  # Injected from JWT
) -> dict
```

**JSON Schema** (MCP Format):
```json
{
  "name": "list_tasks",
  "description": "List tasks with optional filters for completion status and priority",
  "inputSchema": {
    "type": "object",
    "properties": {
      "is_complete": {
        "type": ["boolean", "null"],
        "description": "Filter by completion status (true=completed, false=incomplete, null=all)",
        "default": null
      },
      "priority": {
        "type": ["string", "null"],
        "description": "Filter by priority level",
        "enum": ["low", "medium", "high", null],
        "default": null
      },
      "limit": {
        "type": "integer",
        "description": "Maximum number of tasks to return",
        "minimum": 1,
        "maximum": 100,
        "default": 50
      }
    }
  }
}
```

**Return Type**:
```json
{
  "success": true,
  "tasks": [
    {
      "id": 42,
      "title": "Buy groceries",
      "description": "Milk, eggs, bread",
      "priority": "high",
      "is_complete": false,
      "created_at": "2026-01-02T10:30:00Z",
      "updated_at": "2026-01-02T10:30:00Z"
    },
    {
      "id": 43,
      "title": "Finish project",
      "description": "",
      "priority": "medium",
      "is_complete": false,
      "created_at": "2026-01-01T15:00:00Z",
      "updated_at": "2026-01-01T15:00:00Z"
    }
  ],
  "count": 2
}
```

**Implementation Example**:
```python
async def list_tasks(
    is_complete: bool | None = None,
    priority: str | None = None,
    limit: int = 50,
    user_id: int = None
) -> dict:
    """List tasks for the user with optional filters."""
    if not user_id:
        return {"success": False, "error": "Unauthorized"}

    # Build query
    statement = select(Task).where(Task.user_id == user_id)

    # Apply filters
    if is_complete is not None:
        statement = statement.where(Task.is_complete == is_complete)

    if priority:
        statement = statement.where(Task.priority == priority)

    # Apply limit
    statement = statement.limit(limit).order_by(Task.created_at.desc())

    # Execute query
    tasks = session.exec(statement).all()

    return {
        "success": True,
        "tasks": [
            {
                "id": task.id,
                "title": task.title,
                "description": task.description,
                "priority": task.priority,
                "is_complete": task.is_complete,
                "created_at": task.created_at.isoformat(),
                "updated_at": task.updated_at.isoformat()
            }
            for task in tasks
        ],
        "count": len(tasks)
    }
```

---

### 3. update_task

**Description**: Update an existing task's title, description, or priority.

**Function Signature**:
```python
async def update_task(
    task_id: int,
    title: str | None = None,
    description: str | None = None,
    priority: str | None = None,
    user_id: int = None  # Injected from JWT
) -> dict
```

**JSON Schema** (MCP Format):
```json
{
  "name": "update_task",
  "description": "Update an existing task's title, description, or priority",
  "inputSchema": {
    "type": "object",
    "required": ["task_id"],
    "properties": {
      "task_id": {
        "type": "integer",
        "description": "ID of the task to update"
      },
      "title": {
        "type": ["string", "null"],
        "description": "New task title (1-200 characters)",
        "minLength": 1,
        "maxLength": 200
      },
      "description": {
        "type": ["string", "null"],
        "description": "New task description (max 2000 characters)",
        "maxLength": 2000
      },
      "priority": {
        "type": ["string", "null"],
        "description": "New task priority level",
        "enum": ["low", "medium", "high", null]
      }
    }
  }
}
```

**Return Type**:
```json
{
  "success": true,
  "task": {
    "id": 42,
    "title": "Buy groceries and snacks",
    "description": "Milk, eggs, bread, chips",
    "priority": "high",
    "is_complete": false,
    "created_at": "2026-01-02T10:30:00Z",
    "updated_at": "2026-01-02T11:00:00Z"
  }
}
```

**Error Response**:
```json
{
  "success": false,
  "error": "Task not found"
}
```

**Implementation Example**:
```python
async def update_task(
    task_id: int,
    title: str | None = None,
    description: str | None = None,
    priority: str | None = None,
    user_id: int = None
) -> dict:
    """Update an existing task."""
    if not user_id:
        return {"success": False, "error": "Unauthorized"}

    # Find task (with user_id filter for security)
    statement = select(Task).where(Task.id == task_id, Task.user_id == user_id)
    task = session.exec(statement).first()

    if not task:
        return {"success": False, "error": "Task not found"}

    # Update fields
    if title is not None:
        task.title = title.strip()

    if description is not None:
        task.description = description.strip()

    if priority is not None:
        if priority not in ["low", "medium", "high"]:
            return {"success": False, "error": "Invalid priority"}
        task.priority = priority

    # Update timestamp
    task.updated_at = datetime.utcnow()

    session.add(task)
    session.commit()
    session.refresh(task)

    return {
        "success": True,
        "task": {
            "id": task.id,
            "title": task.title,
            "description": task.description,
            "priority": task.priority,
            "is_complete": task.is_complete,
            "created_at": task.created_at.isoformat(),
            "updated_at": task.updated_at.isoformat()
        }
    }
```

---

### 4. toggle_task_completion

**Description**: Mark a task as complete or incomplete.

**Function Signature**:
```python
async def toggle_task_completion(
    task_id: int,
    is_complete: bool,
    user_id: int = None  # Injected from JWT
) -> dict
```

**JSON Schema** (MCP Format):
```json
{
  "name": "toggle_task_completion",
  "description": "Mark a task as complete or incomplete",
  "inputSchema": {
    "type": "object",
    "required": ["task_id", "is_complete"],
    "properties": {
      "task_id": {
        "type": "integer",
        "description": "ID of the task to toggle"
      },
      "is_complete": {
        "type": "boolean",
        "description": "New completion status (true=completed, false=incomplete)"
      }
    }
  }
}
```

**Return Type**:
```json
{
  "success": true,
  "task": {
    "id": 42,
    "title": "Buy groceries",
    "description": "Milk, eggs, bread",
    "priority": "high",
    "is_complete": true,
    "created_at": "2026-01-02T10:30:00Z",
    "updated_at": "2026-01-02T12:00:00Z"
  }
}
```

**Implementation Example**:
```python
async def toggle_task_completion(
    task_id: int,
    is_complete: bool,
    user_id: int = None
) -> dict:
    """Toggle task completion status."""
    if not user_id:
        return {"success": False, "error": "Unauthorized"}

    # Find task
    statement = select(Task).where(Task.id == task_id, Task.user_id == user_id)
    task = session.exec(statement).first()

    if not task:
        return {"success": False, "error": "Task not found"}

    # Update completion status
    task.is_complete = is_complete
    task.updated_at = datetime.utcnow()

    session.add(task)
    session.commit()
    session.refresh(task)

    return {
        "success": True,
        "task": {
            "id": task.id,
            "title": task.title,
            "description": task.description,
            "priority": task.priority,
            "is_complete": task.is_complete,
            "created_at": task.created_at.isoformat(),
            "updated_at": task.updated_at.isoformat()
        }
    }
```

---

### 5. delete_task

**Description**: Permanently delete a task.

**Function Signature**:
```python
async def delete_task(
    task_id: int,
    user_id: int = None  # Injected from JWT
) -> dict
```

**JSON Schema** (MCP Format):
```json
{
  "name": "delete_task",
  "description": "Permanently delete a task",
  "inputSchema": {
    "type": "object",
    "required": ["task_id"],
    "properties": {
      "task_id": {
        "type": "integer",
        "description": "ID of the task to delete"
      }
    }
  }
}
```

**Return Type**:
```json
{
  "success": true,
  "message": "Task deleted successfully"
}
```

**Error Response**:
```json
{
  "success": false,
  "error": "Task not found"
}
```

**Implementation Example**:
```python
async def delete_task(
    task_id: int,
    user_id: int = None
) -> dict:
    """Delete a task permanently."""
    if not user_id:
        return {"success": False, "error": "Unauthorized"}

    # Find task
    statement = select(Task).where(Task.id == task_id, Task.user_id == user_id)
    task = session.exec(statement).first()

    if not task:
        return {"success": False, "error": "Task not found"}

    # Delete task
    session.delete(task)
    session.commit()

    return {
        "success": True,
        "message": "Task deleted successfully"
    }
```

---

### 6. get_task

**Description**: Get a single task by ID.

**Function Signature**:
```python
async def get_task(
    task_id: int,
    user_id: int = None  # Injected from JWT
) -> dict
```

**JSON Schema** (MCP Format):
```json
{
  "name": "get_task",
  "description": "Get a single task by ID",
  "inputSchema": {
    "type": "object",
    "required": ["task_id"],
    "properties": {
      "task_id": {
        "type": "integer",
        "description": "ID of the task to retrieve"
      }
    }
  }
}
```

**Return Type**:
```json
{
  "success": true,
  "task": {
    "id": 42,
    "title": "Buy groceries",
    "description": "Milk, eggs, bread",
    "priority": "high",
    "is_complete": false,
    "created_at": "2026-01-02T10:30:00Z",
    "updated_at": "2026-01-02T10:30:00Z"
  }
}
```

**Implementation Example**:
```python
async def get_task(
    task_id: int,
    user_id: int = None
) -> dict:
    """Get a single task by ID."""
    if not user_id:
        return {"success": False, "error": "Unauthorized"}

    # Find task
    statement = select(Task).where(Task.id == task_id, Task.user_id == user_id)
    task = session.exec(statement).first()

    if not task:
        return {"success": False, "error": "Task not found"}

    return {
        "success": True,
        "task": {
            "id": task.id,
            "title": task.title,
            "description": task.description,
            "priority": task.priority,
            "is_complete": task.is_complete,
            "created_at": task.created_at.isoformat(),
            "updated_at": task.updated_at.isoformat()
        }
    }
```

---

## MCP Server Implementation

### Server Setup

```python
from mcp.server import Server, NotificationOptions
from mcp.server.models import InitializationOptions
from mcp.types import Tool, TextContent

# Create MCP server
server = Server("todo-mcp-server")

# Store user context (set by FastAPI on each request)
_user_context: dict[str, int] = {}


def set_user_context(request_id: str, user_id: int):
    """Set user context for the current request."""
    _user_context[request_id] = user_id


def get_user_context(request_id: str) -> int | None:
    """Get user context for the current request."""
    return _user_context.get(request_id)


# List available tools
@server.list_tools()
async def list_tools() -> list[Tool]:
    """List all available MCP tools."""
    return [
        Tool(
            name="create_task",
            description="Create a new todo task with title, optional description, and priority",
            inputSchema={
                "type": "object",
                "required": ["title"],
                "properties": {
                    "title": {"type": "string", "minLength": 1, "maxLength": 200},
                    "description": {"type": "string", "maxLength": 2000, "default": ""},
                    "priority": {"type": "string", "enum": ["low", "medium", "high"], "default": "medium"}
                }
            }
        ),
        Tool(
            name="list_tasks",
            description="List tasks with optional filters for completion status and priority",
            inputSchema={
                "type": "object",
                "properties": {
                    "is_complete": {"type": ["boolean", "null"], "default": None},
                    "priority": {"type": ["string", "null"], "enum": ["low", "medium", "high", None], "default": None},
                    "limit": {"type": "integer", "minimum": 1, "maximum": 100, "default": 50}
                }
            }
        ),
        Tool(
            name="update_task",
            description="Update an existing task's title, description, or priority",
            inputSchema={
                "type": "object",
                "required": ["task_id"],
                "properties": {
                    "task_id": {"type": "integer"},
                    "title": {"type": ["string", "null"], "minLength": 1, "maxLength": 200},
                    "description": {"type": ["string", "null"], "maxLength": 2000},
                    "priority": {"type": ["string", "null"], "enum": ["low", "medium", "high", None]}
                }
            }
        ),
        Tool(
            name="toggle_task_completion",
            description="Mark a task as complete or incomplete",
            inputSchema={
                "type": "object",
                "required": ["task_id", "is_complete"],
                "properties": {
                    "task_id": {"type": "integer"},
                    "is_complete": {"type": "boolean"}
                }
            }
        ),
        Tool(
            name="delete_task",
            description="Permanently delete a task",
            inputSchema={
                "type": "object",
                "required": ["task_id"],
                "properties": {
                    "task_id": {"type": "integer"}
                }
            }
        ),
        Tool(
            name="get_task",
            description="Get a single task by ID",
            inputSchema={
                "type": "object",
                "required": ["task_id"],
                "properties": {
                    "task_id": {"type": "integer"}
                }
            }
        )
    ]


# Handle tool calls
@server.call_tool()
async def call_tool(name: str, arguments: dict, request_id: str = None) -> list[TextContent]:
    """Execute MCP tool calls."""
    # Get user context
    user_id = get_user_context(request_id) if request_id else None

    if not user_id:
        return [TextContent(type="text", text=json.dumps({"success": False, "error": "Unauthorized"}))]

    # Route to appropriate tool
    if name == "create_task":
        result = await create_task(**arguments, user_id=user_id)
    elif name == "list_tasks":
        result = await list_tasks(**arguments, user_id=user_id)
    elif name == "update_task":
        result = await update_task(**arguments, user_id=user_id)
    elif name == "toggle_task_completion":
        result = await toggle_task_completion(**arguments, user_id=user_id)
    elif name == "delete_task":
        result = await delete_task(**arguments, user_id=user_id)
    elif name == "get_task":
        result = await get_task(**arguments, user_id=user_id)
    else:
        result = {"success": False, "error": f"Unknown tool: {name}"}

    return [TextContent(type="text", text=json.dumps(result))]
```

---

## Integration with OpenAI Agents SDK

### Convert MCP Tools to OpenAI Function Format

```python
from openai_agents import Agent

def mcp_to_openai_tools(mcp_tools: list[Tool]) -> list[dict]:
    """Convert MCP tool schemas to OpenAI function calling format."""
    return [
        {
            "type": "function",
            "function": {
                "name": tool.name,
                "description": tool.description,
                "parameters": tool.inputSchema
            }
        }
        for tool in mcp_tools
    ]

# Get MCP tools
mcp_tools = await server.list_tools()

# Convert to OpenAI format
openai_tools = mcp_to_openai_tools(mcp_tools)

# Create agent with tools
agent = Agent(
    name="Todo Assistant",
    instructions="You are a helpful todo assistant. Use the available tools to help users manage their tasks.",
    model="gpt-4o",
    tools=openai_tools
)
```

---

## Error Handling

### Common Errors

| Error | Cause | Response |
|-------|-------|----------|
| Unauthorized | Missing or invalid user_id | `{"success": false, "error": "Unauthorized"}` |
| Task not found | Task doesn't exist or belongs to another user | `{"success": false, "error": "Task not found"}` |
| Invalid priority | Priority not in [low, medium, high] | `{"success": false, "error": "Invalid priority"}` |
| Validation error | Missing required field or invalid type | `{"success": false, "error": "Title is required"}` |

### Error Response Format

All tools return errors in a consistent format:

```json
{
  "success": false,
  "error": "Human-readable error message"
}
```

---

## Testing Requirements

### Unit Tests
- [ ] Test each tool with valid inputs
- [ ] Test each tool with invalid inputs (validation errors)
- [ ] Test each tool with missing user_id (unauthorized)
- [ ] Test each tool with non-existent task_id (not found)

### Security Tests
- [ ] Test user A cannot access user B's tasks (all tools)
- [ ] Test user_id filter on all database queries
- [ ] Test tool calls without user_id return unauthorized error

### Integration Tests
- [ ] Test full flow: create → list → update → toggle → delete
- [ ] Test MCP server integration with OpenAI Agents SDK
- [ ] Test tool schema validation (JSON Schema)

---

## Performance Requirements

- **Tool Execution Time**: p95 < 100ms
- **Database Query Time**: p95 < 50ms
- **Concurrent Tool Calls**: Support 100 concurrent requests

---

## Acceptance Criteria

- [ ] All 6 MCP tools implemented
- [ ] All tools enforce user_id filtering
- [ ] All tools return consistent error format
- [ ] JSON schemas validate correctly
- [ ] MCP server registered and running
- [ ] OpenAI Agents SDK integration working
- [ ] All unit tests passing
- [ ] All security tests passing (no cross-user access)
- [ ] All integration tests passing

---

## Dependencies

- MCP SDK 1.0+
- SQLModel 0.0.8+
- OpenAI Agents SDK 0.6.4+
- FastAPI 0.104+

---

## References

- [MCP SDK Documentation](https://github.com/modelcontextprotocol/python-sdk)
- [OpenAI Function Calling](https://platform.openai.com/docs/guides/function-calling)
- [JSON Schema Specification](https://json-schema.org/)
- Phase III Spec: `/specs/003-phase3-ai-chatbot/spec.md`
- Phase III Plan: `/specs/003-phase3-ai-chatbot/plan.md`
- Data Model: `/specs/003-phase3-ai-chatbot/data-model.md`
- API Spec: `/specs/003-phase3-ai-chatbot/contracts/api-spec.md`
