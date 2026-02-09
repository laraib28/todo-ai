# Feature Specification: Phase-III AI-Powered Todo Chatbot

**Feature Branch**: `003-phase3-ai-chatbot`
**Created**: 2026-01-02
**Status**: Draft
**Input**: User requirement for AI-powered natural language todo management

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Natural Language Task Creation (Priority: P1) 🎯 MVP

As a logged-in user, I want to create tasks using natural language (e.g., "Add buy milk to my tasks") so that I can quickly add tasks without filling forms.

**Why this priority**: Core chatbot functionality. Without natural language task creation, the chatbot has no value. This is the primary use case for the AI feature.

**Independent Test**: Login, open chat, send message "Add buy groceries with high priority", verify task created in database with correct attributes (title: "buy groceries", priority: "high", status: incomplete).

**Acceptance Scenarios**:

1. **Given** I am logged in and viewing the chat interface, **When** I send message "Add buy milk to my tasks", **Then** the AI creates a task with title "buy milk", default priority "medium", status incomplete, and responds with confirmation "Task created: buy milk"

2. **Given** I am chatting with the AI, **When** I send "Add call dentist with high priority and description schedule checkup", **Then** the AI creates task with title "call dentist", priority "high", description "schedule checkup", and responds with task details

3. **Given** I am using the chat, **When** I send "Create a task: buy groceries - milk, eggs, bread with low priority", **Then** the AI parses the request and creates task with title "buy groceries", description "milk, eggs, bread", priority "low"

4. **Given** I send an ambiguous message like "add it", **When** no prior context exists, **Then** the AI responds "I need more information. What task would you like to add?" without creating a task

5. **Given** I create a task via chat, **When** the task is successfully created, **Then** the task appears immediately in the main task list (Phase II dashboard) without page refresh

---

### User Story 2 - Natural Language Task Updates (Priority: P1) 🎯 MVP

As a logged-in user, I want to update tasks using natural language (e.g., "Change buy milk to buy almond milk") so that I can modify tasks conversationally.

**Why this priority**: Essential for task maintenance via chat. Users expect to modify tasks they discuss with the AI.

**Independent Test**: Create task "buy milk", send chat message "Change buy milk to buy almond milk", verify task title updated in database and response confirms change.

**Acceptance Scenarios**:

1. **Given** I have a task "buy milk", **When** I send "Change buy milk to buy almond milk", **Then** the AI updates the task title to "buy almond milk" and responds "Task updated: buy almond milk"

2. **Given** I have task with ID 5, **When** I send "Update task 5 priority to high", **Then** the AI updates task priority and responds with confirmation

3. **Given** I have a task "call dentist", **When** I send "Add description to call dentist task: schedule annual checkup", **Then** the AI updates the description field and confirms

4. **Given** I send "Change task 999 to completed" but task 999 doesn't exist, **Then** the AI responds "Task not found. Please check the task ID or title."

5. **Given** I send "Update task" without specifying which task or what to change, **Then** the AI responds with clarifying questions "Which task would you like to update? What changes should I make?"

---

### User Story 3 - Natural Language Task Completion Toggle (Priority: P1) 🎯 MVP

As a logged-in user, I want to mark tasks complete/incomplete using natural language (e.g., "Mark buy milk as done") so that I can update task status conversationally.

**Why this priority**: Primary workflow action. Users completing tasks via chat is a core use case.

**Independent Test**: Create incomplete task "buy milk", send "Mark buy milk as done", verify task is_complete=true in database and AI confirms.

**Acceptance Scenarios**:

1. **Given** I have incomplete task "buy milk", **When** I send "Mark buy milk as done", **Then** the AI toggles is_complete to true and responds "Task completed: buy milk"

2. **Given** I have complete task "buy groceries", **When** I send "Mark buy groceries as incomplete", **Then** the AI toggles is_complete to false and responds "Task reopened: buy groceries"

3. **Given** I have task with ID 3, **When** I send "Complete task 3", **Then** the AI marks it complete and confirms

4. **Given** I send "Mark it as done" with no context, **When** AI cannot determine which task, **Then** the AI asks "Which task would you like to mark as done?"

5. **Given** I have 2 tasks both named "buy milk", **When** I send "Complete buy milk", **Then** the AI asks "I found 2 tasks with that name. Which one? (IDs: 5, 7)"

---

### User Story 4 - Natural Language Task Deletion (Priority: P2)

As a logged-in user, I want to delete tasks using natural language (e.g., "Delete buy milk task") so that I can remove tasks conversationally.

**Why this priority**: Useful for cleanup but not essential for MVP. Users can ignore irrelevant tasks or use Phase II UI for deletion.

**Independent Test**: Create task "test task", send "Delete test task", verify task removed from database and AI confirms deletion.

**Acceptance Scenarios**:

1. **Given** I have task "buy milk", **When** I send "Delete buy milk", **Then** the AI deletes the task permanently and responds "Task deleted: buy milk"

2. **Given** I send "Delete task 5", **When** task 5 exists and belongs to me, **Then** the AI deletes it and confirms

3. **Given** I send "Delete all tasks", **When** I have multiple tasks, **Then** the AI asks for confirmation "Are you sure you want to delete all your tasks? This cannot be undone." and waits for explicit "yes" before proceeding

4. **Given** I try to delete task that doesn't exist, **When** I send "Delete task 999", **Then** the AI responds "Task not found."

5. **Given** I send ambiguous deletion request "delete it", **When** no context exists, **Then** the AI asks "Which task would you like to delete?"

---

### User Story 5 - Natural Language Task Listing & Search (Priority: P1) 🎯 MVP

As a logged-in user, I want to query my tasks using natural language (e.g., "Show my high priority tasks" or "What tasks are incomplete?") so that I can find tasks conversationally.

**Why this priority**: Essential for users to understand their task state via chat. Without this, users can't know what tasks exist.

**Independent Test**: Create 3 tasks (2 high priority, 1 low priority), send "Show my high priority tasks", verify AI lists only the 2 high priority tasks.

**Acceptance Scenarios**:

1. **Given** I have 5 tasks total, **When** I send "Show all my tasks", **Then** the AI lists all 5 tasks with ID, title, priority, and status

2. **Given** I have 3 incomplete and 2 complete tasks, **When** I send "What tasks are incomplete?", **Then** the AI lists only the 3 incomplete tasks

3. **Given** I have tasks with various priorities, **When** I send "Show my high priority tasks", **Then** the AI lists only high priority tasks

4. **Given** I have task "buy milk", **When** I send "Do I have a task about milk?", **Then** the AI responds "Yes, you have: buy milk (ID: 5, Priority: medium, Status: incomplete)"

5. **Given** I have no tasks, **When** I send "Show my tasks", **Then** the AI responds "You don't have any tasks yet. Would you like to create one?"

---

### User Story 6 - Conversational Context & Multi-Turn Interactions (Priority: P2)

As a logged-in user, I want the AI to remember context within a conversation (e.g., "Add buy milk" then "Make it high priority") so that I can have natural multi-turn conversations.

**Why this priority**: Enhances UX but not required for MVP. Stateless architecture makes this challenging; each message is independent.

**Independent Test**: Send "Add buy milk", then immediately send "Make it high priority", verify AI understands "it" refers to just-created task.

**Acceptance Scenarios**:

1. **Given** I just sent "Add buy milk" and received task ID 5, **When** I send "Make it high priority" in the same conversation, **Then** the AI updates task 5's priority to high

2. **Given** I asked "Show my tasks" and AI listed tasks, **When** I send "Complete the first one", **Then** the AI marks the first listed task as complete

3. **Given** I sent "Add buy groceries", **When** I send "Add description: milk, eggs, bread" within 30 seconds, **Then** the AI infers "it" refers to the just-created task

4. **Given** conversation is idle for 5 minutes, **When** I send "Make it high priority", **Then** the AI asks "Which task would you like to update?" (context expired)

5. **Given** I switch between chat and Phase II UI, **When** I create task in UI then chat "Complete the grocery task", **Then** the AI finds the task regardless of creation method

**Note**: Context is stored in database (conversation_history table), NOT in-memory. Each API call retrieves recent context from DB.

---

### User Story 7 - Error Handling & Clarifications (Priority: P1) 🎯 MVP

As a logged-in user, I want the AI to handle errors gracefully and ask clarifying questions when my request is ambiguous so that I don't encounter cryptic errors.

**Why this priority**: Essential for good UX. Users will make ambiguous or invalid requests; AI must handle gracefully.

**Independent Test**: Send invalid request "asdfghjkl", verify AI responds with helpful message rather than error.

**Acceptance Scenarios**:

1. **Given** I send gibberish "asdfghjkl", **When** AI cannot parse intent, **Then** AI responds "I didn't understand that. You can ask me to add, update, complete, delete, or show tasks. For example: 'Add buy milk' or 'Show my tasks'"

2. **Given** I send "Update task 5" but task 5 belongs to another user, **When** AI attempts operation, **Then** AI responds "Task not found" (enforces user isolation, doesn't reveal other users' tasks)

3. **Given** database connection fails, **When** I send any request, **Then** AI responds "I'm having trouble connecting to the database. Please try again in a moment."

4. **Given** I send "Add" with no task details, **When** AI detects missing information, **Then** AI responds "What task would you like to add?"

5. **Given** I send request during OpenAI API outage, **When** AI service is unavailable, **Then** backend returns 503 Service Unavailable with message "AI service temporarily unavailable. Please try again later."

---

### User Story 8 - ChatKit Frontend Integration (Priority: P1) 🎯 MVP

As a logged-in user, I want a chat interface in the web app where I can type messages and see AI responses so that I can interact with the chatbot.

**Why this priority**: Frontend is required for users to access the chatbot. Without UI, feature is unusable.

**Independent Test**: Login to web app, navigate to /chat page, send message "Add buy milk", verify message appears in chat history and AI response is displayed.

**Acceptance Scenarios**:

1. **Given** I am logged in, **When** I navigate to /chat, **Then** I see ChatKit interface with input box, send button, and chat history area

2. **Given** I am on chat page, **When** I type "Add buy milk" and click Send, **Then** my message appears in chat history with timestamp, followed by AI response

3. **Given** I send a message, **When** AI is processing (network delay), **Then** I see loading indicator and input is disabled until response arrives

4. **Given** I have long conversation, **When** chat history exceeds viewport height, **Then** interface auto-scrolls to latest message

5. **Given** I am not authenticated, **When** I try to access /chat, **Then** middleware redirects to /login with message "Please log in to continue"

---

### Edge Cases

- **Empty User Task List**: User asks "Show my tasks" but has zero tasks → AI responds "You don't have any tasks yet. Would you like to create one?"
- **Duplicate Task Titles**: User has 3 tasks named "buy milk" and says "Complete buy milk" → AI lists all matching tasks with IDs and asks "Which one?"
- **Invalid Priority Values**: User says "Add buy milk with super ultra priority" → AI defaults to "medium" and responds "Task created with medium priority (valid options: high, medium, low)"
- **Very Long Task Titles**: User sends "Add" followed by 1000-character string → Backend validates max 200 chars, AI responds "Task title too long. Please keep it under 200 characters."
- **SQL Injection Attempts**: User sends "Add task'; DROP TABLE tasks; --" → SQLModel parameterized queries prevent injection; task created with literal title
- **OpenAI API Rate Limits**: Too many requests hit OpenAI rate limit → Backend returns 429 Too Many Requests with retry-after header
- **Concurrent Modifications**: User updates task via chat while simultaneously updating via Phase II UI → Last write wins (database-level)
- **Network Timeout**: Chat request takes >30 seconds → Backend times out, frontend shows "Request timed out. Please try again."
- **Malformed JSON from OpenAI**: OpenAI returns invalid JSON for tool calls → Agent SDK handles error, AI responds "I encountered an error processing your request. Please try again."
- **User Isolation Bypass Attempt**: User tries "Show all tasks" expecting to see other users' tasks → AI only queries WHERE user_id = current_user.id, returns only user's tasks
- **MCP Tool Execution Failure**: MCP tool raises exception (e.g., database error) → Agent SDK catches error, AI responds with user-friendly message

---

## Requirements *(mandatory)*

### Functional Requirements

#### AI Chatbot Core

- **FR-001**: System MUST expose single endpoint POST /api/chat/{user_id} that accepts natural language messages and returns AI responses
- **FR-002**: System MUST use OpenAI Agents SDK (Python) for AI agent logic and conversation orchestration
- **FR-003**: System MUST implement MCP server using official MCP SDK with tools for task operations (create, read, update, delete, list)
- **FR-004**: System MUST use stateless architecture - all conversation context stored in database, NO in-memory state
- **FR-005**: System MUST authenticate requests using existing Better Auth JWT tokens from Phase II
- **FR-006**: System MUST enforce user isolation - AI can ONLY access tasks belonging to authenticated user
- **FR-007**: AI agent MUST ONLY interact with tasks via MCP tools (no direct database access from agent code)
- **FR-008**: System MUST NOT use RAG, embeddings, vector databases, or Context7 (pure LLM reasoning + MCP tools only)
- **FR-009**: System MUST store conversation history in database for context (last 10 messages per user)
- **FR-010**: System MUST validate all user inputs before passing to AI (max message length: 1000 characters)

#### Natural Language Task Operations

- **FR-011**: AI MUST support task creation via natural language (extract title, description, priority from user message)
- **FR-012**: AI MUST support task updates via natural language (identify task by ID or title, update specified fields)
- **FR-013**: AI MUST support task completion toggle via natural language (mark complete/incomplete)
- **FR-014**: AI MUST support task deletion via natural language (with confirmation for bulk deletes)
- **FR-015**: AI MUST support task listing/search via natural language (filter by status, priority, title keyword)
- **FR-016**: AI MUST handle ambiguous requests by asking clarifying questions (e.g., "Which task?" when multiple matches)
- **FR-017**: AI MUST provide conversational confirmations for all operations ("Task created: buy milk")
- **FR-018**: AI MUST default to priority "medium" when not specified
- **FR-019**: AI MUST validate extracted parameters against schema (e.g., priority must be high/medium/low)
- **FR-020**: AI MUST gracefully handle parsing failures and provide helpful error messages

#### MCP Server & Tools

- **FR-021**: MCP server MUST expose tool: create_task(title: str, description: str = "", priority: str = "medium") → Task
- **FR-022**: MCP server MUST expose tool: get_tasks(user_id: int, status: str = "all", priority: str = "all") → List[Task]
- **FR-023**: MCP server MUST expose tool: update_task(task_id: int, user_id: int, title: str = None, description: str = None, priority: str = None) → Task
- **FR-024**: MCP server MUST expose tool: toggle_task_completion(task_id: int, user_id: int) → Task
- **FR-025**: MCP server MUST expose tool: delete_task(task_id: int, user_id: int) → bool
- **FR-026**: All MCP tools MUST enforce user_id parameter for user isolation (verify task.user_id == user_id)
- **FR-027**: All MCP tools MUST return 404-equivalent when task not found or user_id mismatch
- **FR-028**: MCP server MUST use existing Neon PostgreSQL database (same as Phase II)
- **FR-029**: MCP server MUST use existing Task model from Phase II (no schema changes)
- **FR-030**: MCP tools MUST include error handling and return structured error responses

#### ChatKit Frontend

- **FR-031**: Frontend MUST add /chat route with ChatKit interface
- **FR-032**: ChatKit MUST display conversation history (user messages + AI responses) in chronological order
- **FR-033**: ChatKit MUST provide text input field and Send button for user messages
- **FR-034**: ChatKit MUST show loading indicator while waiting for AI response
- **FR-035**: ChatKit MUST auto-scroll to latest message when new message arrives
- **FR-036**: ChatKit MUST display timestamps for each message
- **FR-037**: ChatKit MUST distinguish user messages from AI messages visually (different colors/alignment)
- **FR-038**: ChatKit MUST disable input during message processing to prevent double-sends
- **FR-039**: ChatKit MUST handle Enter key to send message (with Shift+Enter for newlines)
- **FR-040**: ChatKit MUST integrate with existing Phase II authentication (JWT cookie)
- **FR-041**: ChatKit MUST redirect to /login if user not authenticated
- **FR-042**: ChatKit MUST display error messages when chat API fails
- **FR-043**: Frontend MUST add navigation link to /chat in existing Navbar

#### API & Data Layer

- **FR-044**: Backend MUST add POST /api/chat endpoint to existing FastAPI app
- **FR-045**: Chat endpoint MUST accept JSON: `{"message": "user message", "user_id": int}`
- **FR-046**: Chat endpoint MUST return JSON: `{"response": "AI response", "timestamp": "ISO8601"}`
- **FR-047**: Chat endpoint MUST validate JWT token using existing get_current_user dependency
- **FR-048**: Chat endpoint MUST rate limit to 10 requests/minute per user (prevent abuse)
- **FR-049**: Backend MUST add conversation_history table: id, user_id, role (user/assistant), message, timestamp
- **FR-050**: Backend MUST store last 10 messages per user in conversation_history (auto-delete older messages)
- **FR-051**: Backend MUST retrieve conversation context from DB before each AI request (stateless)
- **FR-052**: Backend MUST use existing CORS configuration from Phase II (allow frontend origin)
- **FR-053**: Backend MUST add OpenAI API key to environment variables (OPENAI_API_KEY)
- **FR-054**: Backend MUST handle OpenAI API errors gracefully (rate limits, timeouts, invalid responses)

### Key Entities

- **ConversationMessage**: Represents a single message in chat history
  - Attributes: id (int, PK), user_id (int, FK to User), role (str, enum: user/assistant), message (str, max 2000 chars), timestamp (datetime)
  - Relationships: Many-to-one with User

- **Task**: Existing entity from Phase II (NO changes)
  - Attributes: id, user_id, title, description, priority, is_complete, created_at, updated_at
  - Relationships: Many-to-one with User

- **User**: Existing entity from Phase II (NO changes)

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: User can complete full chatbot workflow (login → open chat → "Add buy milk" → task created → "Show my tasks" → task listed → "Complete buy milk" → task marked complete) without errors
- **SC-002**: AI correctly extracts task parameters from 90%+ of common natural language requests (tested with 50 sample phrases)
- **SC-003**: User isolation is 100% enforced - AI NEVER accesses other users' tasks (verified by creating 2 users and attempting cross-access via chat)
- **SC-004**: All task operations via chat are reflected in Phase II dashboard immediately (verified by refreshing dashboard after chat operations)
- **SC-005**: Conversation context persists across multiple messages within same session (tested with multi-turn conversation: "Add task" → "Make it high priority")
- **SC-006**: System handles ambiguous requests gracefully - asks clarifying questions instead of guessing (tested with "Update it" with no context)
- **SC-007**: ChatKit frontend integrates seamlessly with existing Phase II UI (same auth, navbar, styling)
- **SC-008**: MCP server tools enforce user_id validation on every operation (verified by inspecting tool code and testing unauthorized access)
- **SC-009**: No in-memory state - backend can restart mid-conversation and continue correctly (verified by restarting server during active chat)
- **SC-010**: OpenAI API errors are handled gracefully with user-friendly messages (tested by simulating API failures)

## Non-Functional Requirements

### Performance

- **NFR-001**: Chat endpoint responds in < 3 seconds for p95 latency (OpenAI API latency + tool execution)
- **NFR-002**: MCP tool execution completes in < 500ms (database query performance)
- **NFR-003**: Conversation history queries load in < 100ms (last 10 messages per user)
- **NFR-004**: Frontend ChatKit loads initial chat history in < 1 second

### Security

- **NFR-005**: All chat requests MUST be authenticated with valid JWT token
- **NFR-006**: User isolation MUST be enforced at MCP tool level (verify user_id on every operation)
- **NFR-007**: OpenAI API key MUST be stored in environment variable, never committed to git
- **NFR-008**: User messages MUST be sanitized before storage to prevent XSS (escape HTML)
- **NFR-009**: Rate limiting MUST prevent abuse (10 requests/minute per user)
- **NFR-010**: MCP server MUST NOT expose internal database connection details in error messages
- **NFR-011**: Conversation history MUST be user-isolated (WHERE user_id = current_user.id)

### Reliability

- **NFR-012**: System MUST handle OpenAI API timeouts gracefully (30 second timeout with retry)
- **NFR-013**: System MUST handle database connection failures with appropriate error messages
- **NFR-014**: MCP tools MUST use database transactions for atomic operations
- **NFR-015**: Chat endpoint MUST return 503 Service Unavailable when OpenAI API is down
- **NFR-016**: Frontend MUST implement retry logic for failed chat requests (3 retries with exponential backoff)

### Usability

- **NFR-017**: AI responses MUST be conversational and user-friendly (not technical/robotic)
- **NFR-018**: AI MUST ask clarifying questions when request is ambiguous (not guess incorrectly)
- **NFR-019**: ChatKit MUST show loading indicator during AI processing (provide feedback)
- **NFR-020**: Error messages MUST be actionable and non-technical ("Task not found" not "404 HTTP error")
- **NFR-021**: ChatKit MUST be responsive and work on mobile devices

### Code Quality

- **NFR-022**: Backend code MUST follow PEP 8 style guidelines
- **NFR-023**: Backend code MUST use type hints for all function signatures
- **NFR-024**: Frontend code MUST use TypeScript with strict mode
- **NFR-025**: MCP server MUST follow official MCP SDK patterns and conventions
- **NFR-026**: OpenAI Agents SDK integration MUST follow official SDK best practices
- **NFR-027**: All environment variables MUST be documented in .env.example

### Maintainability

- **NFR-028**: MCP tools MUST be modular and reusable (separate from agent logic)
- **NFR-029**: Agent prompt/instructions MUST be configurable (not hardcoded)
- **NFR-030**: Conversation history retention policy MUST be configurable (default: 10 messages)
- **NFR-031**: OpenAI model selection MUST be configurable via environment variable (default: gpt-4o)

### Constraints

- **NFR-032**: Backend MUST continue using FastAPI framework (extend Phase II backend)
- **NFR-033**: Frontend MUST continue using Next.js 16+ with App Router (extend Phase II frontend)
- **NFR-034**: Database MUST be Neon Serverless PostgreSQL (same as Phase II, NO vector database)
- **NFR-035**: AI SDK MUST be OpenAI Agents SDK (official Python SDK)
- **NFR-036**: MCP SDK MUST be official MCP SDK (mcp package)
- **NFR-037**: ChatKit MUST be used for chat UI (no custom chat implementation)
- **NFR-038**: Phase I CLI and Phase II web app MUST NOT be modified (additive changes only)
- **NFR-039**: NO RAG, NO embeddings, NO vector databases, NO Context7
- **NFR-040**: Agent MUST ONLY use MCP tools for task operations (no direct DB access from agent)
- **NFR-041**: Architecture MUST be stateless (all state in database, no in-memory state)

## Out of Scope (Explicitly Forbidden for MVP)

- Voice input/output for chat
- Multi-language support (English only)
- Conversation export/download
- Chat history search
- Task recommendations or suggestions from AI
- Scheduled tasks or reminders via AI
- Task sharing or collaboration via chat
- Integration with external services (calendar, email, etc.)
- Custom AI personalities or personas
- Fine-tuning or custom models
- Streaming responses (complete response only)
- File uploads via chat
- Image generation or analysis
- Task analytics or insights from AI
- Natural language queries about statistics ("How many tasks did I complete this week?")
- Conversation branching or threading
- Message editing or deletion
- User feedback on AI responses (thumbs up/down)
- AI memory across sessions (context limited to last 10 messages)
- Proactive AI suggestions ("Would you like to create a task?")
- Integration with Phase I CLI

## Technical Context

**Architecture**: AI agent layer added on top of existing Phase II full-stack app

**Backend Stack (Extended from Phase II)**:
- Framework: FastAPI (existing)
- AI Agent: OpenAI Agents SDK (Python 3.11+) - NEW
- MCP Server: Official MCP SDK (mcp package) - NEW
- ORM: SQLModel (existing)
- Database: Neon Serverless PostgreSQL (existing)
- Authentication: Better Auth with JWT (existing)
- New Dependencies: openai, mcp-sdk, anthropic-sdk

**Frontend Stack (Extended from Phase II)**:
- Framework: Next.js 16+ (existing)
- Chat UI: ChatKit - NEW
- Language: TypeScript (existing)
- Styling: Tailwind CSS (existing)
- HTTP Client: fetch API (existing)

**AI & MCP Architecture**:

```
User → ChatKit UI → POST /api/chat → FastAPI → OpenAI Agent → MCP Tools → Database
                                         ↓
                                    Conversation History
                                         (Database)
```

**Flow**:
1. User sends message via ChatKit
2. Frontend POSTs to /api/chat with JWT authentication
3. Backend validates JWT, extracts user_id
4. Backend loads last 10 conversation messages from DB (context)
5. Backend calls OpenAI Agent with user message + context
6. Agent analyzes message, determines intent, calls appropriate MCP tool
7. MCP tool executes database operation (with user_id validation)
8. MCP tool returns result to agent
9. Agent generates conversational response
10. Backend stores user message + AI response in conversation_history
11. Backend returns AI response to frontend
12. Frontend displays response in ChatKit

**MCP Tools**:
- `create_task(title, description?, priority?) → Task`
- `get_tasks(user_id, status?, priority?) → List[Task]`
- `update_task(task_id, user_id, title?, description?, priority?) → Task`
- `toggle_task_completion(task_id, user_id) → Task`
- `delete_task(task_id, user_id) → bool`

**OpenAI Agent Configuration**:
- Model: gpt-4o (configurable via env)
- Temperature: 0.7 (balanced between creative and deterministic)
- Max tokens: 500 (concise responses)
- Tools: MCP tools registered with OpenAI function calling
- System prompt: "You are a helpful assistant managing a user's todo list. Use the provided tools to help users create, update, complete, delete, and view their tasks. Always confirm operations and ask clarifying questions when needed."

**Database Schema (Additions)**:

```sql
-- New table for conversation history
CREATE TABLE conversation_history (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(20) NOT NULL CHECK (role IN ('user', 'assistant')),
    message TEXT NOT NULL,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user_timestamp (user_id, timestamp DESC)
);

-- No changes to existing users and tasks tables
```

**API Design**:

```
POST /api/chat
Request:
{
    "message": "Add buy milk with high priority"
}
Headers:
    Cookie: access_token=<jwt_token>

Response:
{
    "response": "Task created: buy milk with high priority (ID: 5)",
    "timestamp": "2026-01-02T10:30:00Z"
}
```

**Environment Variables (Additions)**:

```bash
# OpenAI API
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4o
OPENAI_MAX_TOKENS=500
OPENAI_TEMPERATURE=0.7

# MCP Server
MCP_SERVER_NAME=todo-mcp-server

# Conversation Settings
CONVERSATION_HISTORY_LIMIT=10
CHAT_RATE_LIMIT=10  # requests per minute per user
```

**Performance Goals**:
- Chat response time: < 3 seconds p95
- MCP tool execution: < 500ms
- Support: 100+ concurrent users

**Scale/Scope**: Add AI chatbot to existing multi-user app, expected 100-1000 users, 10-100 chat messages per user per day

## Project Structure

```
to-do/
├── backend/                         # Phase-II FastAPI backend (EXTEND)
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py                  # MODIFY: Add chat router
│   │   ├── database.py              # MODIFY: Add ConversationHistory model
│   │   ├── models.py                # MODIFY: Add ConversationHistory SQLModel
│   │   ├── schemas.py               # MODIFY: Add ChatRequest, ChatResponse schemas
│   │   ├── auth.py                  # NO CHANGE
│   │   ├── dependencies.py          # NO CHANGE
│   │   ├── routers/
│   │   │   ├── __init__.py
│   │   │   ├── auth.py              # NO CHANGE
│   │   │   ├── tasks.py             # NO CHANGE
│   │   │   └── chat.py              # NEW: Chat endpoint
│   │   ├── ai/                      # NEW: AI agent logic
│   │   │   ├── __init__.py
│   │   │   ├── agent.py             # OpenAI Agent setup and orchestration
│   │   │   └── prompts.py           # System prompts and instructions
│   │   └── mcp/                     # NEW: MCP server and tools
│   │       ├── __init__.py
│   │       ├── server.py            # MCP server setup
│   │       └── tools.py             # MCP tools (create, read, update, delete, list)
│   ├── requirements.txt             # MODIFY: Add openai, mcp-sdk
│   ├── .env.example                 # MODIFY: Add OpenAI and MCP env vars
│   └── README.md                    # MODIFY: Add Phase III setup instructions
├── frontend/                        # Phase-II Next.js frontend (EXTEND)
│   ├── app/
│   │   ├── layout.tsx               # NO CHANGE
│   │   ├── page.tsx                 # NO CHANGE
│   │   ├── register/                # NO CHANGE
│   │   ├── login/                   # NO CHANGE
│   │   ├── dashboard/               # NO CHANGE
│   │   └── chat/                    # NEW: Chat page
│   │       └── page.tsx             # ChatKit integration
│   ├── components/
│   │   ├── TaskList.tsx             # NO CHANGE
│   │   ├── TaskItem.tsx             # NO CHANGE
│   │   ├── TaskForm.tsx             # NO CHANGE
│   │   ├── Navbar.tsx               # MODIFY: Add /chat link
│   │   └── ChatInterface.tsx        # NEW: ChatKit wrapper
│   ├── lib/
│   │   ├── api.ts                   # MODIFY: Add chat API function
│   │   ├── auth.ts                  # NO CHANGE
│   │   └── types.ts                 # MODIFY: Add ChatMessage type
│   ├── package.json                 # MODIFY: Add chatkit dependency
│   └── README.md                    # MODIFY: Add Phase III usage instructions
├── src/                             # Phase-I CLI code (DO NOT MODIFY)
├── specs/
│   ├── 001-cli-todo-app/            # Phase-I spec (DO NOT MODIFY)
│   ├── 002-phase2-web/              # Phase-II spec (DO NOT MODIFY)
│   └── 003-phase3-ai-chatbot/       # Phase-III spec (THIS FILE)
│       ├── spec.md                  # This file
│       ├── plan.md                  # To be generated by /sp.plan
│       └── tasks.md                 # To be generated by /sp.tasks
└── README.md                        # MODIFY: Add Phase III overview
```

## Example User Journey (Happy Path)

### First-Time Chat Interaction

1. User logs in to web app (existing Phase II auth)
2. User clicks "Chat" link in navbar
3. Browser navigates to /chat page
4. ChatKit interface loads with empty conversation history
5. User types: "Add buy groceries with high priority"
6. User clicks Send button
7. Frontend sends POST /api/chat with JWT cookie
8. Backend validates JWT, extracts user_id
9. Backend loads conversation history (empty for first message)
10. Backend calls OpenAI Agent with message + context
11. Agent analyzes: intent=create_task, title="buy groceries", priority="high"
12. Agent calls MCP tool: `create_task("buy groceries", "", "high")`
13. MCP tool inserts task into database with user_id
14. MCP tool returns Task object
15. Agent generates response: "Task created: buy groceries with high priority (ID: 7)"
16. Backend stores both messages in conversation_history
17. Backend returns AI response to frontend
18. Frontend displays AI response in chat
19. User sees confirmation message

### Multi-Turn Conversation

20. User types: "Show my tasks"
21. Agent calls MCP tool: `get_tasks(user_id, "all", "all")`
22. MCP tool returns list of tasks
23. Agent formats response: "You have 1 task: buy groceries (ID: 7, Priority: high, Status: incomplete)"
24. User types: "Complete it"
25. Backend loads conversation history (last 10 messages including context)
26. Agent infers "it" refers to task 7 from context
27. Agent calls MCP tool: `toggle_task_completion(7, user_id)`
28. MCP tool updates task.is_complete = true
29. Agent responds: "Task completed: buy groceries"
30. User refreshes Phase II dashboard
31. Dashboard shows task with strikethrough (completed)

## Quality Gates Before Next Phase

- ✅ Spec is complete, unambiguous, and covers all user stories with acceptance criteria
- ✅ OpenAI Agents SDK integration is fully specified
- ✅ MCP server architecture with tools is defined
- ✅ Stateless architecture with database-backed context is specified
- ✅ User isolation at MCP tool level is explicitly required
- ✅ ChatKit frontend integration is detailed
- ✅ All constraints are explicitly stated (no RAG, no embeddings, no Context7)
- ✅ Security requirements are comprehensive (JWT auth, user isolation, rate limiting)
- ✅ Technology stack is specified (OpenAI Agents SDK, MCP SDK, ChatKit)
- ✅ Database schema additions are defined (conversation_history table)
- ✅ API design is specified (single endpoint: POST /api/chat)
- ✅ Edge cases and error handling are identified
- ✅ Success criteria are measurable
- ✅ Out of scope items prevent scope creep
- ✅ Phase I and Phase II integration constraints are clear (no modifications, additive only)

## Alignment with Constitution

This specification adheres to the project constitution:

- **Spec-Driven Development**: This is the specification phase of Phase III following Agentic Dev Stack Workflow
- **Test-First Development**: Acceptance scenarios define expected behavior before implementation
- **Security First**: JWT authentication, user isolation, input validation, rate limiting
- **Clean Architecture**: Separation of concerns (FastAPI router → Agent → MCP tools → Database)
- **Multi-User Support**: User isolation enforced at MCP tool level (user_id validation)
- **Data Persistence**: All state in Neon PostgreSQL (conversation history, tasks)
- **No Phase-I/II Modification**: Explicit constraint to only extend existing code, not modify
- **Stateless Architecture**: All context in database, no in-memory state

**Next Steps**:
1. Review and approve this specification
2. Generate implementation plan (plan.md) using `/sp.plan` command
3. Generate task breakdown (tasks.md) using `/sp.tasks` command
4. Execute implementation using `/sp.implement` command
5. Commit and create PR using `/sp.git.commit_pr` command

---

**Note to Implementer**: This spec defines WHAT to build, not HOW to build it. The implementation plan (plan.md) will define architectural decisions, and tasks.md will break down specific implementation steps. Follow the Spec-Kit Plus workflow strictly.
