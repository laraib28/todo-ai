# Phase III Research: AI Chatbot Technical Stack

**Feature**: 003-phase3-ai-chatbot
**Date**: 2026-01-02
**Status**: Research Complete

---

## Executive Summary

This document consolidates research findings for Phase III AI-powered todo chatbot implementation using:
- **OpenAI Agents SDK** for AI orchestration
- **MCP SDK** for standardized tool interface
- **Custom React components** for chat UI (ChatKit not recommended)
- **Stateless architecture** with database-backed conversation context

**Key Decision**: Use OpenAI Agents SDK + MCP SDK + Custom Chat UI components (not ChatKit).

---

## 1. OpenAI Agents SDK Research

### Overview

- **Package**: `openai-agents` (PyPI)
- **Latest Version**: 0.6.4 (production-ready, March 2025)
- **Python**: 3.9+
- **Core Dependencies**: openai >= 2.0.0, pydantic >= 2.0

### Installation

```bash
pip install openai-agents
```

### Agent Setup with Function Calling

```python
from openai_agents import Agent, function_tool

@function_tool
async def create_task(title: str, description: str = "", priority: str = "medium") -> dict:
    """Create a new todo task."""
    # Implementation automatically converted to tool schema
    return {"id": 1, "title": title, "status": "created"}

agent = Agent(
    name="Todo Assistant",
    instructions="Help manage todo tasks",
    model="gpt-4o",
    tools=[create_task]
)
```

**Key Features**:
- Automatic schema generation from type hints and docstrings
- Async/sync support for all tools
- Tool behavior control via `ModelSettings(tool_choice="auto"|"required"|"none")`

### Conversation Context Management

**Session-Based Automatic History**:
```python
from openai_agents import SQLiteSession, Runner

session = SQLiteSession(f"user_{user_id}", db_path="./conversations.db")
result = await Runner.run(agent, "Create a task", session=session)
# Automatically includes full conversation history
```

**Session Options**:
- `SQLiteSession` - Lightweight, default (recommended for MVP)
- `OpenAIConversationsSession` - OpenAI-hosted
- `SQLAlchemySession` - Production, any database
- `EncryptedSession` - Sensitive conversations

Sessions automatically:
- Retrieve history before each run
- Store new messages after each run
- Enable multi-turn context preservation

**IMPORTANT**: For stateless architecture required by spec, use SQLAlchemySession with Neon PostgreSQL (not SQLite).

### Error Handling

**Native Error Types**:

| Error | HTTP Code | Recovery |
|-------|-----------|----------|
| `RateLimitError` | 429 | Exponential backoff retry |
| `APITimeoutError` | 504 | Retry with timeout increase |
| `APIConnectionError` | Network | Retry with exponential backoff |
| `APIError` | 502/500 | Retry, investigate if persistent |

**Recommended Pattern with Retry**:
```python
from tenacity import retry, wait_exponential, stop_after_attempt

@retry(
    wait=wait_exponential(multiplier=1, min=1, max=60),
    stop=stop_after_attempt(6),
    reraise=True
)
async def agent_run(agent, message, session):
    return await Runner.run(agent, message, session=session)
```

### Streaming vs Complete Responses

**SDK Capabilities**:
- **Default**: Complete responses (recommended for MVP)
- **Optional**: Streaming via `Runner.stream()` for real-time feedback

**MVP Recommendation**: Use complete responses (simpler, faster to implement).

```python
# Complete (default)
result = await Runner.run(agent, message, session=session)
return result.final_output  # String

# Streaming (optional, future enhancement)
async with Runner.stream(agent, message, session=session) as stream:
    async for event in stream:
        yield f"data: {json.dumps(event)}\n\n"  # SSE to frontend
```

### Integration Pattern for Todo Chatbot

```python
# backend/app/ai/agent.py
from openai_agents import Agent, function_tool, SQLAlchemySession, Runner
from app.database import engine
from app.models import Task

# Define tools as decorated functions
@function_tool
async def create_task(title: str, description: str = "", priority: str = "medium", user_id: int = None) -> dict:
    """Create a new todo task for the authenticated user."""
    # Implementation with database access
    task = Task(user_id=user_id, title=title, description=description, priority=priority)
    # Save and return
    return {"id": task.id, "title": task.title, "priority": task.priority}

# Create agent
agent = Agent(
    name="Todo Assistant",
    instructions="""You are a helpful assistant managing a user's todo list.
    Use the provided tools to help users create, update, complete, delete, and view their tasks.
    Always confirm operations and ask clarifying questions when needed.""",
    model="gpt-4o",
    tools=[create_task, get_tasks, update_task, toggle_task, delete_task]
)

# FastAPI endpoint
@router.post("/api/chat")
async def chat(request: ChatRequest, current_user: User = Depends(get_current_user)):
    # Create session for conversation history
    session = SQLAlchemySession(f"user_{current_user.id}", engine=engine)

    # Run agent with error handling
    try:
        result = await agent_run(agent, request.message, session)
        return {"response": result.final_output}
    except RateLimitError:
        return {"error": "AI service rate limit exceeded. Please try again."}
    except APIError as e:
        return {"error": "AI service temporarily unavailable"}
```

---

## 2. MCP (Model Context Protocol) SDK Research

### Overview

- **Package**: `mcp` (pip install mcp)
- **Python**: 3.11+
- **Protocol**: JSON-RPC 2.0 over stdio
- **Architecture**: Two-tier client-server model

### Core MCP Server Implementation

```python
from mcp.server import Server
from mcp.types import Tool, TextContent

server = Server("todo-server")

@server.list_tools()
async def list_tools() -> list[Tool]:
    """Return available tools to clients"""
    return [
        Tool(
            name="create_task",
            description="Create a new task for the authenticated user",
            inputSchema={
                "type": "object",
                "properties": {
                    "title": {"type": "string", "description": "Task title", "maxLength": 200},
                    "priority": {"type": "string", "enum": ["high", "medium", "low"]},
                },
                "required": ["title"],
            },
        ),
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict, user_id: int = None) -> list[TextContent]:
    """Handle tool execution with user_id context"""
    if name == "create_task":
        # Implementation with user_id scoping
        return [TextContent(type="text", text="Task created")]
```

### Tool Definition Standards

**JSON Schema Requirements**:
```python
{
    "type": "object",
    "properties": {
        "parameter_name": {
            "type": "string",           # Types: string, number, integer, boolean, array, object
            "description": "...",       # REQUIRED: what the parameter does
            "enum": [...],             # Optional: allowed values
            "minLength": 1,            # Optional: for strings
            "maxLength": 200,          # Optional: for strings
            "minimum": 0,              # Optional: for numbers
            "maximum": 100,            # Optional: for numbers
        }
    },
    "required": ["parameter_name"]     # Which parameters are mandatory
}
```

### User Context Flow for Multi-Tenant Security

```
MCP Client (Claude)
    ↓ Authenticates with JWT token
MCP Server (stdio)
    ↓ Middleware validates JWT signature
Extract user_id from JWT payload:
    {
        "user_id": 123,
        "email": "user@example.com",
        "exp": 1704200000
    }
    ↓ Pass user_id to tool handler
Tool handler executes with user_id context
    ↓ All database queries filtered: WHERE user_id = context_user_id
Row-level security enforced
```

**Tool Handler Pattern with User Context**:
```python
async def call_tool(name: str, arguments: dict, user_id: int) -> list[TextContent]:
    """Tool handler receives user_id from JWT authentication."""

    if not user_id:
        return error_response("Unauthorized: No user context")

    # All database operations scoped to user_id
    if name == "create_task":
        task = Task(
            user_id=user_id,  # Automatically set from JWT
            title=arguments["title"],
            priority=arguments.get("priority", "medium")
        )
        # Save and return
```

### Multi-Tenant Security Checklist

- ✅ user_id required for all database operations
- ✅ All SELECT queries include `WHERE user_id = current_user`
- ✅ All INSERT operations set user_id from JWT
- ✅ All UPDATE/DELETE verify ownership (403 if mismatch)
- ✅ Database constraints enforce foreign keys
- ✅ No task IDs exposed across user boundaries

### Database Access Patterns

**Connection Management (Recommended)**:
```python
from sqlmodel import Session
from app.database import engine

async def call_tool(name: str, arguments: dict, user_id: int):
    with Session(engine) as session:
        # All queries use this session
        task = session.query(Task).get(task_id)
        # Auto cleanup on exit
```

**Safe Query Construction**:
```python
# ✅ SAFE - Parameterized (SQLModel handles)
tasks = session.query(Task).where(Task.user_id == user_id).all()

# ❌ UNSAFE - String interpolation (SQL injection!)
query = f"SELECT * FROM tasks WHERE user_id = {user_id}"
```

### MCP Tools and OpenAI Function Calling

**Direct Schema Compatibility**:
```python
# MCP Tool converts directly to OpenAI function
def mcp_to_openai_tools(mcp_tools: list[Tool]) -> list[dict]:
    """Convert MCP tools to OpenAI format"""
    return [{
        "type": "function",
        "function": {
            "name": tool.name,
            "description": tool.description,
            "parameters": tool.inputSchema
        }
    } for tool in mcp_tools]
```

---

## 3. ChatKit UI Library Research

### Status: NOT RECOMMENDED

**Critical Finding**: ChatKit is **not actively maintained** and **lacks documentation** for production use.

**Issues**:
- Unmaintained packages
- Poor Next.js 16+ compatibility
- No TypeScript support
- Requires Pusher service (expensive)
- Rigid styling, poor customization

### Recommended Alternative: Custom Components

**Why custom is better**:
- Full control over UI/UX
- Lightweight (2-5KB vs 30KB+)
- Native Tailwind CSS integration
- Perfect TypeScript support
- Zero external dependencies

### Custom Chat Component Implementation

**Message Display**:
```typescript
// frontend/components/ChatMessage.tsx
interface ChatMessage {
  id: string;
  role: 'user' | 'assistant';
  content: string;
  timestamp: Date;
  isLoading?: boolean;
}

export function ChatMessage({ message }: { message: ChatMessage }) {
  const isUser = message.role === 'user';

  return (
    <div className={isUser ? 'flex justify-end' : 'flex justify-start'}>
      <div className={isUser ? 'bg-blue-500 text-white rounded-br-none' : 'bg-gray-200 text-gray-900 rounded-bl-none'}>
        <p>{message.content}</p>
        <span>{message.timestamp.toLocaleTimeString()}</span>
      </div>
    </div>
  );
}
```

**Input Field**:
```typescript
// frontend/components/ChatInput.tsx
export function ChatInput({ onSend, isLoading }: ChatInputProps) {
  const [input, setInput] = useState('');

  const handleKeyPress = (e) => {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault();
      onSend(input);
      setInput('');
    }
  };

  return (
    <div className="flex gap-2 p-4 border-t">
      <textarea
        value={input}
        onChange={(e) => setInput(e.target.value)}
        onKeyPress={handleKeyPress}
        placeholder="Type a task or command..."
        disabled={isLoading}
        className="flex-1 px-4 py-2 border rounded-lg"
      />
      <button
        onClick={() => { onSend(input); setInput(''); }}
        disabled={isLoading || !input.trim()}
        className="bg-blue-500 text-white rounded-lg px-4 py-2"
      >
        Send
      </button>
    </div>
  );
}
```

**Loading States**:
1. Optimistic update (show user message immediately)
2. Placeholder "thinking" indicator
3. Streaming update (progressive response)
4. Bounce animation

**Auto-Scroll**:
```typescript
// Use Intersection Observer for smart scrolling
const messagesEndRef = useRef<HTMLDivElement>(null);

useEffect(() => {
  messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
}, [messages]);
```

---

## 4. Stateless Architecture Design

### Requirement Analysis

**Spec Requirement**: "Stateless architecture - all state in database, NO in-memory state"

**Solution**: Use SQLAlchemySession with Neon PostgreSQL

```python
# backend/app/database.py
from sqlmodel import create_engine

engine = create_engine(
    NEON_DATABASE_URL,
    pool_size=5,
    max_overflow=10,
    pool_pre_ping=True,
)

# backend/app/models.py
class ConversationHistory(SQLModel, table=True):
    __tablename__ = "conversation_history"

    id: Optional[int] = Field(default=None, primary_key=True)
    user_id: int = Field(foreign_key="users.id", nullable=False, index=True)
    role: str = Field(nullable=False)  # "user" or "assistant"
    message: str = Field(nullable=False, max_length=2000)
    timestamp: datetime = Field(default_factory=datetime.utcnow)
```

**Conversation Context Retrieval**:
```python
def get_conversation_context(user_id: int, limit: int = 10) -> list[dict]:
    """Retrieve last N messages for user from database"""
    with Session(engine) as session:
        messages = session.query(ConversationHistory)\
            .where(ConversationHistory.user_id == user_id)\
            .order_by(ConversationHistory.timestamp.desc())\
            .limit(limit)\
            .all()

        return [{"role": m.role, "content": m.message} for m in reversed(messages)]
```

**Agent Execution with Database Context**:
```python
@router.post("/api/chat")
async def chat(request: ChatRequest, current_user: User = Depends(get_current_user)):
    # Retrieve conversation context from database (stateless)
    context_messages = get_conversation_context(current_user.id, limit=10)

    # Run agent with context
    session = SQLAlchemySession(f"user_{current_user.id}", engine=engine)
    result = await Runner.run(agent, request.message, session=session)

    # Store user message and AI response in database
    store_conversation_message(current_user.id, "user", request.message)
    store_conversation_message(current_user.id, "assistant", result.final_output)

    return {"response": result.final_output}
```

**Benefits**:
- Backend can restart mid-conversation without losing context
- Horizontal scaling supported (no session affinity required)
- Conversation history persists across sessions
- Simple to implement and maintain

---

## 5. Integration Architecture

### Complete System Flow

```
Frontend (Next.js 16+)
    ↓ User sends message
POST /api/chat (FastAPI)
    ↓ Validate JWT, extract user_id
Retrieve conversation history from DB (last 10 messages)
    ↓ Load context
OpenAI Agent (with MCP tools)
    ↓ Analyze message, determine intent
Call MCP Tool (with user_id context)
    ↓ Execute database operation
Database (Neon PostgreSQL)
    ↓ Return result
Agent generates conversational response
    ↓ Store messages
Save user message + AI response to conversation_history table
    ↓ Return response
Frontend displays response
```

### Technology Stack Summary

| Component | Technology | Rationale |
|-----------|------------|-----------|
| AI Orchestration | OpenAI Agents SDK | Official SDK, auto schema generation, session management |
| Tool Interface | MCP SDK | Standardized protocol, OpenAI compatible, type-safe |
| Chat UI | Custom React Components | Lightweight, customizable, Tailwind native |
| Conversation Storage | Neon PostgreSQL | Stateless, scalable, existing database |
| Session Management | SQLAlchemySession | Database-backed, stateless architecture |
| Frontend Framework | Next.js 16+ App Router | Existing stack, server components |
| Backend Framework | FastAPI | Existing stack, async support |

---

## 6. Performance & Security Considerations

### Performance

**OpenAI API Latency**:
- Expected: 1-3 seconds for gpt-4o responses
- Target: < 3 seconds p95 (spec requirement)
- Mitigation: Use complete responses (not streaming) for MVP

**Database Queries**:
- Conversation history: < 100ms (10 messages, indexed by user_id + timestamp)
- MCP tool operations: < 500ms (spec requirement)
- Optimization: Composite index on (user_id, timestamp DESC)

### Security

**Authentication**:
- ✅ JWT validation on every /api/chat request
- ✅ User ID extracted from JWT payload
- ✅ MCP tools receive user_id context
- ✅ All queries filtered by user_id

**Rate Limiting**:
- ✅ 10 requests/minute per user (spec requirement)
- ✅ Prevent OpenAI API abuse
- ✅ Protect against brute force

**Input Validation**:
- ✅ Max message length: 1000 characters
- ✅ Sanitize inputs before AI processing
- ✅ Escape HTML in responses
- ✅ Parameterized database queries only

---

## 7. Implementation Recommendations

### Priority 1: Core Setup

1. **Install Dependencies**:
   ```bash
   # Backend
   pip install openai-agents mcp-sdk

   # Frontend (no ChatKit!)
   npm install clsx date-fns
   ```

2. **Create Database Schema**:
   - Add `conversation_history` table
   - Composite index: (user_id, timestamp DESC)

3. **Implement MCP Tools**:
   - `create_task` - with user_id scoping
   - `get_tasks` - filtered by user_id
   - `update_task` - with ownership verification
   - `toggle_task` - atomic operation
   - `delete_task` - with confirmation

4. **Setup OpenAI Agent**:
   - Initialize agent with tools
   - Configure SQLAlchemySession
   - Implement error handling with retry

5. **Create Chat Endpoint**:
   - POST /api/chat with JWT auth
   - Load conversation context from DB
   - Run agent and store messages
   - Return response

### Priority 2: Frontend

6. **Custom Chat Components**:
   - ChatMessage.tsx (message display)
   - ChatInput.tsx (input field)
   - ChatContainer.tsx (main UI with state)

7. **Chat Page**:
   - /app/chat/page.tsx (protected route)
   - useChat hook for state management
   - Integration with existing Navbar

### Priority 3: Testing

8. **Manual Testing**:
   - End-to-end chat flow
   - User isolation (2 users, verify separation)
   - Error handling (OpenAI API failures)
   - Multi-turn conversations

---

## Conclusion

**Recommended Approach**:

1. ✅ Use **OpenAI Agents SDK** for AI orchestration (proven, well-documented)
2. ✅ Use **MCP SDK** for standardized tool interface (OpenAI compatible)
3. ✅ Build **custom React components** for chat UI (lightweight, customizable)
4. ✅ Store **all state in Neon PostgreSQL** (stateless architecture)
5. ✅ Enforce **user isolation** at MCP tool level (security first)

**Avoid**:
- ❌ ChatKit library (unmaintained, poor docs)
- ❌ In-memory conversation storage (violates stateless requirement)
- ❌ SQLiteSession (use SQLAlchemySession with Neon instead)

**Next Steps**:
1. Generate data-model.md (database schema for conversation_history)
2. Generate contracts/ (API spec, MCP tool schemas)
3. Generate quickstart.md (setup instructions)
4. Execute implementation via `/sp.implement`

---

**Research Complete** | Total Research Time: ~2 hours | Last Updated: 2026-01-02
