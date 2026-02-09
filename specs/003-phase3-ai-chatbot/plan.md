# Implementation Plan: Phase-III AI-Powered Todo Chatbot

**Branch**: `003-phase3-ai-chatbot` | **Date**: 2026-01-02 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/003-phase3-ai-chatbot/spec.md`

## Summary

Add AI-powered natural language chatbot to existing Phase II todo web application. Users can create, update, complete, delete, and query tasks via conversational chat interface instead of traditional forms. Backend uses OpenAI Agents SDK for AI orchestration and official MCP SDK to expose task operations as tools. Frontend integrates ChatKit for chat UI. All state stored in Neon PostgreSQL (stateless architecture, no in-memory state). Agent ONLY interacts with database via MCP tools. No RAG, no embeddings, no vector databases, no Context7.

**Core Technical Approach**:
- Extend existing Phase II FastAPI backend with single chat endpoint: POST /api/chat
- Integrate OpenAI Agents SDK (Python) for natural language processing and tool orchestration
- Build MCP server with 5 tools (create, read, update, toggle, delete tasks)
- Store conversation history in new `conversation_history` table for context (last 10 messages per user)
- Frontend adds /chat route with ChatKit component
- Enforce user isolation at MCP tool level (all tools validate user_id)
- Maintain existing Better Auth JWT authentication

## Technical Context

**Language/Version**: Python 3.11+ (backend), TypeScript 5.3+ / Node.js 20+ (frontend)

**Primary Dependencies**:
- Backend: FastAPI 0.104+ (existing), SQLModel 0.0.14+ (existing), openai (NEW - OpenAI Agents SDK), mcp-sdk (NEW - official MCP SDK), python-jose (existing), passlib (existing)
- Frontend: Next.js 16+ (existing), React 19+ (existing), ChatKit (NEW), Tailwind CSS (existing)

**Storage**: Neon Serverless PostgreSQL (existing from Phase II)
- New table: conversation_history (id, user_id, role, message, timestamp)
- Existing tables: users, tasks (NO changes)

**Testing**: Manual testing via ChatKit UI and FastAPI /docs for backend endpoints

**Target Platform**:
- Backend: Linux server (FastAPI on uvicorn)
- Frontend: Web browsers (Chrome, Firefox, Safari, Edge)
- Database: Neon cloud-hosted PostgreSQL

**Project Type**: Web application (extends Phase II)

**Performance Goals**:
- Chat endpoint: < 3 seconds p95 latency (OpenAI API + tool execution)
- MCP tool execution: < 500ms per operation
- Conversation history load: < 100ms (last 10 messages)
- Frontend ChatKit: < 1 second initial load

**Constraints**:
- MUST NOT modify Phase I CLI code (src/ directory)
- MUST NOT modify Phase II core functionality (additive changes only)
- NO RAG (Retrieval-Augmented Generation)
- NO embeddings or vector databases
- NO Context7
- NO in-memory state (all state in database)
- Agent MUST ONLY use MCP tools (no direct database access from agent code)
- Single chat endpoint: POST /api/chat (no streaming, complete responses only)
- Rate limit: 10 requests/minute per user
- Stateless architecture: backend can restart mid-conversation without losing context

**Scale/Scope**:
- Expected users: 100-1000 concurrent users
- Chat messages: 10-100 per user per day
- Conversation history: 10 messages per user retained
- OpenAI API: gpt-4o model (configurable)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### Alignment with Constitution Principles

**Note**: The original constitution applies to Phase I CLI application. Phase II extended to web with its own constraints. Phase III further extends Phase II with AI chatbot. Evaluating against project-wide principles:

✅ **Spec-Driven Development**: Following Agentic Dev Stack (spec → plan → tasks → implement)

✅ **Test-First Development**: Acceptance scenarios defined in spec before implementation

✅ **Clean Code & Simplicity**:
- Modular architecture: FastAPI router → Agent → MCP tools → Database
- Type hints required (Python backend, TypeScript frontend)
- PEP 8 compliance (backend)
- Clear separation of concerns

✅ **Security First** (Phase II principle):
- JWT authentication (existing Better Auth)
- User isolation enforced at MCP tool level (user_id validation)
- Input validation (max message length, parameter validation)
- Rate limiting (10 req/min per user)
- No injection vulnerabilities (SQLModel parameterized queries)

✅ **Data Persistence** (Phase II principle):
- Neon PostgreSQL for all state (extends Phase II database)
- Conversation history in DB (not in-memory)
- Stateless architecture

✅ **No Phase-I/II Modification**:
- Phase I: src/ directory untouched
- Phase II: Only additive changes (new routes, new tables, new frontend pages)
- No breaking changes to existing functionality

### New Constraints Introduced

🆕 **AI-Specific Constraints**:
- OpenAI Agents SDK for AI logic (new dependency)
- MCP SDK for tool orchestration (new dependency)
- ChatKit for frontend chat UI (new dependency)
- NO RAG, NO embeddings, NO Context7 (explicit simplicity)
- Agent ONLY uses MCP tools (architectural constraint)

### Constitutional Violations: NONE

No violations detected. Phase III extends Phase II with AI capabilities while maintaining:
- Modular architecture
- Database persistence
- User isolation
- Clean code practices
- Additive changes only

## Project Structure

### Documentation (this feature)

```text
specs/003-phase3-ai-chatbot/
├── spec.md              # Feature specification (COMPLETE)
├── plan.md              # This file (/sp.plan command output)
├── research.md          # Phase 0 output (to be generated)
├── data-model.md        # Phase 1 output (to be generated)
├── quickstart.md        # Phase 1 output (to be generated)
├── contracts/           # Phase 1 output (to be generated)
│   └── api-spec.md      # OpenAPI spec for POST /api/chat
└── tasks.md             # Phase 2 output (/sp.tasks command - NOT created by /sp.plan)
```

### Source Code (repository root)

```text
to-do/
├── backend/                         # Phase-II FastAPI backend (EXTEND)
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py                  # MODIFY: Add chat router import
│   │   ├── database.py              # MODIFY: Add create_db_and_tables for new table
│   │   ├── models.py                # MODIFY: Add ConversationHistory SQLModel
│   │   ├── schemas.py               # MODIFY: Add ChatRequest, ChatResponse
│   │   ├── auth.py                  # NO CHANGE
│   │   ├── dependencies.py          # NO CHANGE (reuse get_current_user)
│   │   ├── routers/
│   │   │   ├── __init__.py
│   │   │   ├── auth.py              # NO CHANGE
│   │   │   ├── tasks.py             # NO CHANGE
│   │   │   └── chat.py              # NEW: POST /api/chat endpoint
│   │   ├── ai/                      # NEW: OpenAI Agent logic
│   │   │   ├── __init__.py
│   │   │   ├── agent.py             # Agent setup, tool registration, orchestration
│   │   │   └── prompts.py           # System prompt for todo assistant
│   │   └── mcp/                     # NEW: MCP server and tools
│   │       ├── __init__.py
│   │       ├── server.py            # MCP server initialization
│   │       └── tools.py             # 5 MCP tools (create, get, update, toggle, delete)
│   ├── tests/                       # Manual testing only (no automated tests for MVP)
│   ├── requirements.txt             # MODIFY: Add openai, mcp-sdk
│   ├── .env.example                 # MODIFY: Add OPENAI_API_KEY, model config
│   └── README.md                    # MODIFY: Phase III setup instructions
│
├── frontend/                        # Phase-II Next.js frontend (EXTEND)
│   ├── app/
│   │   ├── layout.tsx               # NO CHANGE
│   │   ├── page.tsx                 # NO CHANGE
│   │   ├── register/                # NO CHANGE
│   │   ├── login/                   # NO CHANGE
│   │   ├── dashboard/               # NO CHANGE
│   │   └── chat/                    # NEW: Chat page
│   │       └── page.tsx             # ChatKit integration, message handling
│   ├── components/
│   │   ├── TaskList.tsx             # NO CHANGE
│   │   ├── TaskItem.tsx             # NO CHANGE
│   │   ├── TaskForm.tsx             # NO CHANGE
│   │   ├── Navbar.tsx               # MODIFY: Add "Chat" navigation link
│   │   └── ChatInterface.tsx        # NEW: ChatKit wrapper component
│   ├── lib/
│   │   ├── api.ts                   # MODIFY: Add sendChatMessage(message) function
│   │   ├── auth.ts                  # NO CHANGE
│   │   └── types.ts                 # MODIFY: Add ChatMessage, ChatResponse types
│   ├── package.json                 # MODIFY: Add chatkit dependency
│   ├── .env.example                 # NO CHANGE (uses existing NEXT_PUBLIC_API_URL)
│   └── README.md                    # MODIFY: Phase III usage instructions
│
├── src/                             # Phase-I CLI code (DO NOT MODIFY)
│   └── [existing CLI code]
│
├── specs/
│   ├── 001-cli-todo-app/            # Phase-I (DO NOT MODIFY)
│   ├── 002-phase2-web/              # Phase-II (DO NOT MODIFY)
│   └── 003-phase3-ai-chatbot/       # Phase-III (this feature)
│
└── README.md                        # MODIFY: Add Phase III section
```

**Structure Decision**:

Web application structure (Option 2 from template) with backend and frontend separation. Extends existing Phase II structure with:
- Backend: New `/app/ai/` module for OpenAI Agent logic
- Backend: New `/app/mcp/` module for MCP server and tools
- Backend: New `/app/routers/chat.py` for chat endpoint
- Backend: Modified `/app/models.py` to add ConversationHistory model
- Frontend: New `/app/chat/` route for chat UI
- Frontend: New `/components/ChatInterface.tsx` for ChatKit integration
- Frontend: Modified Navbar to add chat navigation

This maintains clean separation between Phase I (CLI), Phase II (web CRUD), and Phase III (AI chat) while reusing existing authentication, database, and task models.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

**Status**: No constitutional violations detected. No complexity tracking required.

All new dependencies (OpenAI Agents SDK, MCP SDK, ChatKit) are necessary for the AI chatbot feature and do not violate existing principles. The architecture remains modular and extends Phase II without modifications to Phase I or core Phase II functionality.

---

## Phase 0: Research & Technology Validation

**Goal**: Resolve all technical unknowns and validate feasibility of OpenAI Agents SDK + MCP SDK integration.

### Research Tasks

1. **OpenAI Agents SDK Integration**
   - Research: Official Python SDK for OpenAI Agents (function calling, tool orchestration)
   - Deliverable: Validate SDK supports:
     - Function calling with custom tools (MCP tools)
     - Conversation context management (pass message history)
     - Error handling for API failures
     - Streaming vs complete responses (we need complete only)
   - Output: Document SDK version, key APIs, example integration pattern

2. **MCP SDK (Model Context Protocol)**
   - Research: Official MCP SDK for Python
   - Deliverable: Validate SDK supports:
     - Tool definition (function signatures, parameters, descriptions)
     - Tool registration with AI agent
     - User context passing (user_id for isolation)
     - Database access patterns from tools
   - Output: Document MCP server setup, tool definition pattern, integration with OpenAI

3. **ChatKit Frontend Library**
   - Research: ChatKit for React/Next.js
   - Deliverable: Validate ChatKit provides:
     - Message display (user/assistant differentiation)
     - Input field with send button
     - Loading states
     - Timestamp display
     - Auto-scroll to latest message
   - Output: Document ChatKit installation, basic usage, styling integration with Tailwind

4. **Stateless Conversation Context**
   - Research: Best practices for storing conversation history in DB for stateless agents
   - Deliverable: Design pattern for:
     - Storing last N messages per user in PostgreSQL
     - Retrieving context on each request (query optimization)
     - Auto-pruning old messages (keep only last 10)
     - Handling concurrent requests (last-write-wins acceptable)
   - Output: Database schema for conversation_history, query patterns

5. **OpenAI Rate Limiting & Error Handling**
   - Research: OpenAI API rate limits, error codes, retry strategies
   - Deliverable: Document:
     - Rate limit tiers for gpt-4o
     - Error codes (429 rate limit, 503 service unavailable, 401 auth, 400 bad request)
     - Recommended retry strategies (exponential backoff)
     - Timeout handling (30 second request timeout)
   - Output: Error handling patterns, user-facing error messages

6. **User Isolation in MCP Tools**
   - Research: Security patterns for multi-tenant MCP tools
   - Deliverable: Design pattern for:
     - Passing user_id to every MCP tool call
     - Validating task ownership (task.user_id == user_id)
     - Preventing unauthorized access (return 404 instead of 403 to avoid user enumeration)
   - Output: MCP tool function signatures with user_id parameter, validation logic

### Research Output: research.md

See `specs/003-phase3-ai-chatbot/research.md` (generated by this plan)

---

## Phase 1: Data Model & API Contracts

**Goal**: Define database schema additions and API contract for chat endpoint.

### 1.1 Data Model

**New Entity: ConversationHistory**

```python
# SQLModel definition
class ConversationHistory(SQLModel, table=True):
    __tablename__ = "conversation_history"

    id: Optional[int] = Field(default=None, primary_key=True)
    user_id: int = Field(foreign_key="users.id", nullable=False, index=True)
    role: str = Field(nullable=False)  # "user" or "assistant"
    message: str = Field(nullable=False, max_length=2000)
    timestamp: datetime = Field(default_factory=datetime.utcnow, nullable=False)

    # Index for efficient querying of recent messages
    __table_args__ = (
        Index('idx_user_timestamp', 'user_id', 'timestamp'),
    )
```

**Relationships**:
- Many-to-one with User (user_id foreign key)
- CASCADE delete when user deleted

**No changes to existing entities**: User, Task remain unchanged from Phase II

### 1.2 API Contract

**Endpoint: POST /api/chat**

Request:
```json
{
  "message": "Add buy milk with high priority"
}
```

Headers:
```
Cookie: access_token=<jwt_token>
Content-Type: application/json
```

Success Response (200):
```json
{
  "response": "Task created: buy milk with high priority (ID: 5)",
  "timestamp": "2026-01-02T10:30:00Z"
}
```

Error Responses:
```json
// 401 Unauthorized (invalid/missing JWT)
{
  "detail": "Not authenticated"
}

// 400 Bad Request (message too long or missing)
{
  "detail": "Message is required and must be under 1000 characters"
}

// 429 Too Many Requests (rate limit exceeded)
{
  "detail": "Rate limit exceeded. Please wait before sending more messages."
}

// 503 Service Unavailable (OpenAI API down)
{
  "detail": "AI service temporarily unavailable. Please try again later."
}

// 500 Internal Server Error (database or unexpected error)
{
  "detail": "An error occurred processing your request."
}
```

**Rate Limiting**: 10 requests per minute per user (based on user_id from JWT)

**Authentication**: Existing Better Auth JWT in httpOnly cookie (reuse get_current_user dependency)

### 1.3 MCP Tools Contract

**Tool 1: create_task**
```python
def create_task(
    user_id: int,
    title: str,
    description: str = "",
    priority: str = "medium"
) -> dict:
    """Create a new task for the user.

    Args:
        user_id: ID of the user creating the task
        title: Task title (required, max 200 chars)
        description: Task description (optional, max 2000 chars)
        priority: Task priority (high/medium/low, default: medium)

    Returns:
        dict: Created task with id, title, description, priority, is_complete, created_at

    Raises:
        ValueError: If title empty or priority invalid
    """
```

**Tool 2: get_tasks**
```python
def get_tasks(
    user_id: int,
    status: str = "all",
    priority: str = "all"
) -> list[dict]:
    """Get user's tasks with optional filtering.

    Args:
        user_id: ID of the user
        status: Filter by status (all/complete/incomplete, default: all)
        priority: Filter by priority (all/high/medium/low, default: all)

    Returns:
        list[dict]: List of tasks matching filters
    """
```

**Tool 3: update_task**
```python
def update_task(
    task_id: int,
    user_id: int,
    title: str = None,
    description: str = None,
    priority: str = None
) -> dict:
    """Update task fields.

    Args:
        task_id: ID of task to update
        user_id: ID of the user (for ownership verification)
        title: New title (optional)
        description: New description (optional)
        priority: New priority (optional)

    Returns:
        dict: Updated task

    Raises:
        ValueError: If task not found or user_id mismatch
    """
```

**Tool 4: toggle_task_completion**
```python
def toggle_task_completion(
    task_id: int,
    user_id: int
) -> dict:
    """Toggle task completion status.

    Args:
        task_id: ID of task to toggle
        user_id: ID of the user (for ownership verification)

    Returns:
        dict: Updated task with new is_complete status

    Raises:
        ValueError: If task not found or user_id mismatch
    """
```

**Tool 5: delete_task**
```python
def delete_task(
    task_id: int,
    user_id: int
) -> dict:
    """Delete a task permanently.

    Args:
        task_id: ID of task to delete
        user_id: ID of the user (for ownership verification)

    Returns:
        dict: {"success": true, "message": "Task deleted"}

    Raises:
        ValueError: If task not found or user_id mismatch
    """
```

### 1.4 Deliverables

- `data-model.md`: Full schema for ConversationHistory entity
- `contracts/api-spec.md`: OpenAPI specification for POST /api/chat
- `contracts/mcp-tools-spec.md`: MCP tools function signatures and schemas
- `quickstart.md`: Setup instructions for Phase III (install dependencies, configure OpenAI API key, run backend/frontend)

---

## Phase 2: Task Breakdown

**Note**: Phase 2 (task breakdown) is NOT executed by `/sp.plan`. Use `/sp.tasks` command after plan is approved.

The `/sp.tasks` command will generate `tasks.md` with:
- Setup tasks (install dependencies, configure environment)
- Backend tasks (MCP tools, agent integration, chat endpoint)
- Frontend tasks (ChatKit integration, chat page, API client)
- Testing tasks (manual testing via UI and /docs)
- Documentation tasks (update README, quickstart)

---

## Risks & Mitigation

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| OpenAI API rate limits hit during development/testing | High | Medium | Use low-traffic times for testing; implement exponential backoff; document rate limits in quickstart |
| MCP SDK documentation insufficient or SDK immature | High | Low | Research SDK early (Phase 0); fallback to direct function calls if MCP abstraction proves problematic |
| ChatKit library conflicts with Next.js 16 or Tailwind | Medium | Low | Validate ChatKit compatibility in Phase 0 research; use alternative chat UI library if needed |
| OpenAI API costs exceed budget | Medium | Low | Use gpt-4o-mini for development/testing; document cost per message in quickstart; implement rate limiting |
| Conversation context grows too large (>10 messages causes latency) | Medium | Low | Limit to 10 messages; truncate old messages; measure latency in testing |
| User isolation bypass in MCP tools | Critical | Low | Mandatory code review of all MCP tools; test with 2 users attempting cross-access; security checklist |
| Stateless architecture fails (context lost on backend restart) | High | Low | Store ALL context in DB; test by restarting backend mid-conversation; validate context retrieval query |

---

## Implementation Order

1. **Phase 0 Research** (1-2 hours)
   - OpenAI Agents SDK validation
   - MCP SDK validation
   - ChatKit validation
   - Generate research.md

2. **Phase 1 Design** (1-2 hours)
   - Generate data-model.md
   - Generate contracts/api-spec.md
   - Generate contracts/mcp-tools-spec.md
   - Generate quickstart.md
   - Update agent context

3. **Phase 2 Tasks** (via `/sp.tasks` command)
   - Generate tasks.md with dependency-ordered tasks

4. **Phase 3 Implementation** (via `/sp.implement` command)
   - Execute tasks from tasks.md
   - Backend: Models, MCP tools, agent setup, chat endpoint
   - Frontend: ChatKit integration, chat page
   - Testing: Manual validation

5. **Phase 4 Commit & PR** (via `/sp.git.commit_pr` command)
   - Commit changes
   - Create pull request

---

## Success Metrics

- [ ] All Phase 0 research tasks completed with documented findings
- [ ] All Phase 1 design artifacts generated (data-model.md, contracts/, quickstart.md)
- [ ] Agent context updated with new technologies
- [ ] Plan passes constitution check (no violations)
- [ ] All technical unknowns resolved (no NEEDS CLARIFICATION remaining)
- [ ] Phase 2 tasks.md ready for `/sp.tasks` command

---

**Next Steps**:
1. Execute Phase 0 research (generate research.md)
2. Execute Phase 1 design (generate data-model.md, contracts/, quickstart.md)
3. Update agent context
4. Run `/sp.tasks` to generate tasks.md
5. Review and approve plan before implementation
