# Tasks: Phase III - AI-Powered Todo Chatbot

**Input**: Design documents from `/specs/003-phase3-ai-chatbot/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/api-spec.md, contracts/mcp-tools-spec.md

**Tests**: Manual testing only (no automated test tasks for MVP)

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

---

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and dependency installation

- [x] T001 Install OpenAI Agents SDK dependency in backend/requirements.txt (add openai-agents==0.6.4)
- [x] T002 [P] Install MCP SDK dependency in backend/requirements.txt (add mcp-sdk==1.0.0)
- [x] T003 [P] Add OPENAI_API_KEY to backend/.env.example with placeholder
- [x] T004 [P] Add OPENAI_MODEL to backend/.env.example (default: gpt-4o)
- [x] T005 [P] Add CONVERSATION_HISTORY_LIMIT to backend/.env.example (default: 10)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

### Database Schema

- [x] T006 Add ConversationHistory SQLModel to backend/app/models.py with fields (id, user_id, role, content, created_at)
- [x] T007 Update User model in backend/app/models.py to add conversation_history relationship
- [x] T008 Create Alembic migration in backend/alembic/versions/003_add_conversation_history.py per data-model.md
- [x] T009 Add database session dependency helper for conversation queries in backend/app/database.py

### API Schemas

- [x] T010 [P] Add ChatRequest schema to backend/app/schemas.py with message field validation
- [x] T011 [P] Add ChatResponse schema to backend/app/schemas.py with message and metadata fields
- [x] T012 [P] Add ChatMetadata schema to backend/app/schemas.py with action, task_id, count fields

### MCP Tools Infrastructure

- [x] T013 Create backend/app/mcp/ directory structure (__init__.py, server.py, tools.py)
- [x] T014 Implement MCP server setup in backend/app/mcp/server.py with user context management
- [x] T015 [P] Implement create_task MCP tool in backend/app/mcp/tools.py with user_id validation
- [x] T016 [P] Implement list_tasks MCP tool in backend/app/mcp/tools.py with filtering by user_id
- [x] T017 [P] Implement update_task MCP tool in backend/app/mcp/tools.py with ownership check
- [x] T018 [P] Implement toggle_task_completion MCP tool in backend/app/mcp/tools.py with user_id filter
- [x] T019 [P] Implement delete_task MCP tool in backend/app/mcp/tools.py with user_id validation
- [x] T020 [P] Implement get_task MCP tool in backend/app/mcp/tools.py with user_id filter

### AI Agent Setup

- [x] T021 Create backend/app/ai/ directory structure (__init__.py, agent.py, prompts.py)
- [x] T022 Define system prompt for todo assistant in backend/app/ai/prompts.py
- [x] T023 Implement OpenAI Agent initialization in backend/app/ai/agent.py with MCP tools registration
- [x] T024 Implement conversation context retrieval helper in backend/app/ai/agent.py (load last 10 messages from DB)
- [x] T025 Implement conversation message storage helper in backend/app/ai/agent.py (save user + assistant messages)

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: User Story 1 - Natural Language Task Creation (Priority: P1) 🎯 MVP

**Goal**: Users can create tasks via natural language (e.g., "Add buy milk with high priority")

**Independent Test**: Login, open chat, send "Add buy groceries with high priority", verify task created in database with correct attributes (title: "buy groceries", priority: "high", status: incomplete)

### Backend for User Story 1

- [x] T026 [US1] Create chat router in backend/app/routers/chat.py with POST /chat endpoint
- [x] T027 [US1] Add JWT authentication dependency to chat endpoint using get_current_user
- [x] T028 [US1] Implement chat message processing logic in chat router (load context → run agent → store messages)
- [x] T029 [US1] Add error handling for OpenAI API failures in chat endpoint (429, 503, 500 errors)
- [x] T030 [US1] Register chat router in backend/app/main.py with /api prefix

### Frontend for User Story 1

- [x] T031 [P] [US1] Create frontend/components/ChatMessage.tsx component with user/assistant styling
- [x] T032 [P] [US1] Create frontend/components/ChatInput.tsx component with textarea and send button
- [x] T033 [P] [US1] Create frontend/components/ChatContainer.tsx component with message list and auto-scroll
- [x] T034 [US1] Implement sendChatMessage API function in frontend/lib/api.ts with JWT cookie
- [x] T035 [US1] Create frontend/app/chat/page.tsx with ChatContainer integration and auth check
- [x] T036 [US1] Add "Chat" navigation link to frontend/components/Navbar.tsx

### Testing User Story 1

Manual testing checklist:
1. Login as user A
2. Navigate to /chat
3. Send "Add buy milk"
4. Verify AI creates task with default priority "medium"
5. Send "Add call dentist with high priority and description schedule checkup"
6. Verify AI creates task with priority "high" and description
7. Navigate to dashboard, verify tasks appear
8. Login as user B, verify cannot see user A's tasks
9. Test ambiguous message "add it" → verify AI asks clarification
10. Test invalid message (empty string) → verify 422 validation error

**Checkpoint**: At this point, User Story 1 should be fully functional and testable independently

---

## Phase 4: User Story 2 - Natural Language Task Updates (Priority: P1) 🎯 MVP

**Goal**: Users can update tasks via natural language (e.g., "Change buy milk to buy almond milk")

**Independent Test**: Create task "buy milk", send chat "Change buy milk to buy almond milk", verify task title updated in database

**Note**: This story builds on User Story 1 infrastructure (MCP update_task tool already implemented in foundational phase)

### Backend for User Story 2

- [x] T037 [US2] Enhance agent prompt in backend/app/ai/prompts.py to handle task update intents
- [x] T038 [US2] Add fuzzy task search logic to MCP tools (find task by title substring) in backend/app/mcp/tools.py

### Testing User Story 2

Manual testing checklist:
1. Create task "buy milk" via chat
2. Send "Change buy milk to buy almond milk"
3. Verify task title updated
4. Send "Update task 5 priority to high"
5. Verify task priority updated
6. Send "Add description to call dentist task: schedule annual checkup"
7. Verify description added
8. Test update non-existent task → verify AI responds "Task not found"
9. Test ambiguous update "Update task" → verify AI asks clarification

**Checkpoint**: At this point, User Stories 1 AND 2 should both work independently

---

## Phase 5: User Story 3 - Natural Language Task Completion Toggle (Priority: P1) 🎯 MVP

**Goal**: Users can mark tasks complete/incomplete via natural language (e.g., "Mark buy milk as done")

**Independent Test**: Create incomplete task "buy milk", send "Mark buy milk as done", verify task is_complete=true in database

**Note**: This story builds on User Story 1 infrastructure (MCP toggle_task_completion tool already implemented in foundational phase)

### Backend for User Story 3

- [x] T039 [US3] Enhance agent prompt in backend/app/ai/prompts.py to handle completion toggle intents
- [x] T040 [US3] Add conversational confirmations to MCP toggle response in backend/app/mcp/tools.py

### Testing User Story 3

Manual testing checklist:
1. Create task "buy milk" via chat
2. Send "Mark buy milk as done"
3. Verify task is_complete=true
4. Send "Mark buy groceries as incomplete"
5. Verify task is_complete=false
6. Send "Complete task 3" (by ID)
7. Verify task 3 marked complete
8. Test ambiguous "Mark it as done" → verify AI asks clarification
9. Test duplicate task names → verify AI asks which task

**Checkpoint**: All core task operations (create, update, complete) now work via chat

---

## Phase 6: User Story 5 - Natural Language Task Listing & Search (Priority: P1) 🎯 MVP

**Goal**: Users can query tasks via natural language (e.g., "Show my high priority tasks")

**Independent Test**: Create 3 tasks (2 high priority, 1 low priority), send "Show my high priority tasks", verify AI lists only the 2 high priority tasks

**Note**: This story uses MCP list_tasks tool already implemented in foundational phase

### Backend for User Story 5

- [x] T041 [US5] Enhance agent prompt in backend/app/ai/prompts.py to handle task listing/search intents
- [x] T042 [US5] Implement formatted task list response in MCP tools (human-readable format) in backend/app/mcp/tools.py

### Frontend for User Story 5

- [x] T043 [P] [US5] Add markdown rendering support to ChatMessage component for formatted lists in frontend/components/ChatMessage.tsx

### Testing User Story 5

Manual testing checklist:
1. Create 5 tasks with various priorities
2. Send "Show all my tasks" → verify lists all 5
3. Send "What tasks are incomplete?" → verify lists only incomplete tasks
4. Send "Show my high priority tasks" → verify filters by priority
5. Send "Do I have a task about milk?" → verify search by keyword
6. Test empty task list → verify AI responds "You don't have any tasks yet"
7. Verify task count in response matches actual tasks

**Checkpoint**: All MVP user stories (US1, US2, US3, US5) are functional

---

## Phase 7: User Story 7 - Error Handling & Clarifications (Priority: P1) 🎯 MVP

**Goal**: AI handles errors gracefully and asks clarifying questions when requests are ambiguous

**Independent Test**: Send invalid request "asdfghjkl", verify AI responds with helpful message rather than error

### Backend for User Story 7

- [ ] T044 [US7] Add error handling wrapper for OpenAI API calls in backend/app/ai/agent.py (RateLimitError, APITimeoutError, APIError)
- [ ] T045 [US7] Implement retry logic with exponential backoff for OpenAI API in backend/app/ai/agent.py
- [ ] T046 [US7] Enhance agent prompt to ask clarifying questions for ambiguous inputs in backend/app/ai/prompts.py
- [ ] T047 [US7] Add user-friendly error messages for database failures in backend/app/mcp/tools.py

### Frontend for User Story 7

- [ ] T048 [P] [US7] Add error message display component in frontend/components/ChatError.tsx
- [ ] T049 [US7] Implement frontend retry logic for failed chat requests in frontend/app/chat/page.tsx

### Testing User Story 7

Manual testing checklist:
1. Send gibberish "asdfghjkl" → verify helpful response
2. Try accessing other user's task → verify "Task not found" (not "Unauthorized")
3. Simulate database error → verify user-friendly message
4. Send "Add" with no details → verify AI asks "What task?"
5. Simulate OpenAI API outage → verify 503 error with message
6. Test empty message → verify 422 validation error
7. Test message > 2000 chars → verify 422 validation error

**Checkpoint**: Error handling and UX polish complete for MVP

---

## Phase 8: User Story 8 - Chat Frontend Integration (Priority: P1) 🎯 MVP

**Goal**: Users have a polished chat interface integrated with existing Phase II UI

**Independent Test**: Login, navigate to /chat, send message, verify message appears and AI responds

**Note**: Most frontend components already created in User Story 1, this phase adds polish and integration

### Frontend Polish for User Story 8

- [ ] T050 [P] [US8] Add loading indicator to ChatInput component in frontend/components/ChatInput.tsx
- [ ] T051 [P] [US8] Implement optimistic UI updates (show user message immediately) in frontend/app/chat/page.tsx
- [ ] T052 [P] [US8] Add timestamp display to ChatMessage component in frontend/components/ChatMessage.tsx
- [ ] T053 [US8] Implement Enter key to send (Shift+Enter for newlines) in frontend/components/ChatInput.tsx
- [ ] T054 [US8] Add authentication guard to chat page in frontend/app/chat/page.tsx (redirect to /login if not authenticated)
- [ ] T055 [US8] Style chat page to match Phase II design system (Tailwind theme) in frontend/app/chat/page.tsx

### Testing User Story 8

Manual testing checklist:
1. Access /chat without login → verify redirects to /login
2. Login and navigate to /chat → verify interface loads
3. Type message and click Send → verify message appears instantly
4. Wait for AI response → verify loading indicator shows
5. Verify AI response appears with timestamp
6. Test Enter key → verify sends message
7. Test Shift+Enter → verify creates newline
8. Send long message → verify interface auto-scrolls
9. Refresh page → verify conversation history loads from DB
10. Verify chat matches Phase II styling (colors, fonts, layout)

**Checkpoint**: Chat UI is polished and fully integrated with Phase II

---

## Phase 9: User Story 4 - Natural Language Task Deletion (Priority: P2)

**Goal**: Users can delete tasks via natural language (e.g., "Delete buy milk task")

**Independent Test**: Create task "test task", send "Delete test task", verify task removed from database

**Note**: This story uses MCP delete_task tool already implemented in foundational phase

### Backend for User Story 4

- [ ] T056 [US4] Enhance agent prompt to handle task deletion intents in backend/app/ai/prompts.py
- [ ] T057 [US4] Add confirmation flow for bulk delete operations in backend/app/ai/agent.py

### Testing User Story 4

Manual testing checklist:
1. Create task "buy milk"
2. Send "Delete buy milk" → verify task deleted
3. Send "Delete task 5" → verify task deleted by ID
4. Send "Delete all tasks" → verify AI asks confirmation
5. Confirm deletion → verify all tasks deleted
6. Try delete non-existent task → verify AI responds "Task not found"
7. Test ambiguous "delete it" → verify AI asks clarification

**Checkpoint**: Task deletion functionality complete

---

## Phase 10: User Story 6 - Conversational Context & Multi-Turn (Priority: P2)

**Goal**: AI remembers context within a conversation (e.g., "Add buy milk" then "Make it high priority")

**Independent Test**: Send "Add buy milk", then immediately send "Make it high priority", verify AI understands "it" refers to just-created task

**Note**: Context storage already implemented in foundational phase, this enhances context-aware responses

### Backend for User Story 6

- [ ] T058 [US6] Enhance agent to use conversation history for pronoun resolution in backend/app/ai/agent.py
- [ ] T059 [US6] Implement context window trimming (keep only last 10 messages) in backend/app/ai/agent.py
- [ ] T060 [US6] Add context timeout logic (5 minutes idle → clear pronouns) in backend/app/ai/agent.py

### Testing User Story 6

Manual testing checklist:
1. Send "Add buy milk" → note task ID
2. Send "Make it high priority" → verify updates correct task
3. Send "Show my tasks" → get list
4. Send "Complete the first one" → verify marks first task complete
5. Send "Add buy groceries" → wait 6 minutes
6. Send "Make it high priority" → verify AI asks clarification (context expired)
7. Create task in Phase II UI → send "Complete the grocery task" → verify AI finds it

**Checkpoint**: Multi-turn conversations with context work smoothly

---

## Phase 11: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories

### Backend Polish

- [ ] T061 [P] Add rate limiting to chat endpoint (60 requests/minute per user) in backend/app/routers/chat.py
- [ ] T062 [P] Add comprehensive logging for all MCP tool calls in backend/app/mcp/tools.py
- [ ] T063 [P] Implement conversation history auto-pruning (keep last 100 messages per user) in backend/app/database.py
- [ ] T064 Add input sanitization to prevent XSS in chat messages in backend/app/schemas.py

### Frontend Polish

- [ ] T065 [P] Add mobile responsive design to chat interface in frontend/app/chat/page.tsx
- [ ] T066 [P] Implement keyboard shortcuts (Ctrl+K to focus chat input) in frontend/app/chat/page.tsx
- [ ] T067 [P] Add empty state message for new chat sessions in frontend/components/ChatContainer.tsx

### Documentation

- [ ] T068 [P] Update backend/README.md with Phase III setup instructions (OpenAI API key, dependencies)
- [ ] T069 [P] Update frontend/README.md with chat feature usage instructions
- [ ] T070 [P] Update root README.md to add Phase III overview and link to quickstart
- [ ] T071 [P] Validate quickstart.md instructions end-to-end (follow all steps, verify they work)

### Security Hardening

- [ ] T072 [P] Audit all MCP tools for user_id validation (security checklist) in backend/app/mcp/tools.py
- [ ] T073 [P] Add security headers to chat endpoint responses in backend/app/routers/chat.py
- [ ] T074 Perform cross-user access testing (2 users trying to access each other's tasks via chat)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Stories (Phase 3-10)**: All depend on Foundational phase completion
  - US1, US2, US3, US5, US7, US8 (P1 stories) should be completed first for MVP
  - US4, US6 (P2 stories) can be added after MVP validation
  - User stories can proceed in parallel if team capacity allows
- **Polish (Phase 11)**: Depends on all desired user stories being complete

### User Story Dependencies

- **US1 (Task Creation)**: Can start after Foundational → No dependencies on other stories
- **US2 (Task Updates)**: Can start after Foundational → No dependencies (uses existing MCP tools)
- **US3 (Task Completion)**: Can start after Foundational → No dependencies (uses existing MCP tools)
- **US5 (Task Listing)**: Can start after Foundational → No dependencies (uses existing MCP tools)
- **US7 (Error Handling)**: Can start after Foundational → Enhances all stories but independently testable
- **US8 (Frontend Polish)**: Can start after US1 → Polishes existing UI components
- **US4 (Task Deletion)**: Can start after Foundational → No dependencies (uses existing MCP tools)
- **US6 (Context)**: Can start after Foundational → Enhances all stories with context awareness

### Within Each Phase

**Setup (Phase 1)**:
- All tasks can run in parallel

**Foundational (Phase 2)**:
- Database schema tasks (T006-T009) must complete before MCP tools can be tested
- API schemas (T010-T012) can run in parallel with database work
- MCP tools (T013-T020) can run in parallel after database ready
- AI agent setup (T021-T025) can run in parallel with MCP tools

**User Stories**:
- Backend tasks before frontend tasks (within each story)
- Testing after implementation (manual validation)

**Polish**:
- All polish tasks can run in parallel

### Parallel Opportunities

#### Phase 1 (Setup) - ALL tasks in parallel:
- T001, T002, T003, T004, T005

#### Phase 2 (Foundational) - Multiple parallel streams:
Stream 1: Database
- T006, T007 (sequential, same file)
- T008, T009 (after T006-T007)

Stream 2: API Schemas
- T010, T011, T012 (all parallel, different schemas)

Stream 3: MCP Tools (after database ready)
- T015, T016, T017, T018, T019, T020 (all parallel, different tools)

Stream 4: AI Agent (can start anytime)
- T021, T022, T023, T024, T025

#### Phase 3 (US1) - Frontend components in parallel:
- T031, T032, T033 (all parallel, different components)

#### Phase 11 (Polish) - Most tasks in parallel:
- T061, T062, T063, T064, T065, T066, T067, T068, T069, T070, T071, T072, T073

---

## Parallel Example: Foundational Phase

```bash
# Stream 1: Database work
Task T006: "Add ConversationHistory SQLModel to backend/app/models.py"
Task T007: "Update User model in backend/app/models.py"
Task T008: "Create Alembic migration"
Task T009: "Add database session helper"

# Stream 2: API Schemas (parallel with Stream 1)
Task T010: "Add ChatRequest schema to backend/app/schemas.py"
Task T011: "Add ChatResponse schema to backend/app/schemas.py"
Task T012: "Add ChatMetadata schema to backend/app/schemas.py"

# Stream 3: MCP Tools (after Stream 1 complete, all 6 tools in parallel)
Task T015: "Implement create_task MCP tool"
Task T016: "Implement list_tasks MCP tool"
Task T017: "Implement update_task MCP tool"
Task T018: "Implement toggle_task_completion MCP tool"
Task T019: "Implement delete_task MCP tool"
Task T020: "Implement get_task MCP tool"

# Stream 4: AI Agent (parallel with all above)
Task T021: "Create backend/app/ai/ directory"
Task T022: "Define system prompt"
Task T023: "Implement OpenAI Agent initialization"
Task T024: "Implement conversation context retrieval"
Task T025: "Implement conversation message storage"
```

---

## Implementation Strategy

### MVP First (P1 User Stories Only)

1. Complete Phase 1: Setup (T001-T005)
2. Complete Phase 2: Foundational (T006-T025) - CRITICAL, blocks all stories
3. Complete Phase 3: US1 - Task Creation (T026-T036)
4. **STOP and VALIDATE**: Test US1 independently (manual testing checklist)
5. Complete Phase 4: US2 - Task Updates (T037-T038)
6. **STOP and VALIDATE**: Test US2 independently
7. Complete Phase 5: US3 - Task Completion (T039-T040)
8. **STOP and VALIDATE**: Test US3 independently
9. Complete Phase 6: US5 - Task Listing (T041-T043)
10. **STOP and VALIDATE**: Test US5 independently
11. Complete Phase 7: US7 - Error Handling (T044-T049)
12. Complete Phase 8: US8 - Frontend Polish (T050-T055)
13. **MVP COMPLETE**: Deploy/demo if ready

### Incremental Delivery

1. Complete Setup + Foundational → Foundation ready
2. Add US1 → Test independently → Deploy/Demo (Core chatbot working!)
3. Add US2, US3, US5 → Test independently → Deploy/Demo (MVP!)
4. Add US7, US8 → Polish and error handling → Deploy/Demo (Production-ready MVP)
5. Add US4, US6 → Enhanced features → Deploy/Demo (Feature-complete)
6. Add Polish (Phase 11) → Final touches → Deploy/Demo (Polished v1.0)

### Parallel Team Strategy

With 3 developers after Foundational phase complete:

**Sprint 1: MVP Core**
- Developer A: US1 (Task Creation) - T026-T036
- Developer B: US2 + US3 (Updates + Completion) - T037-T040
- Developer C: US5 (Task Listing) - T041-T043

**Sprint 2: MVP Polish**
- Developer A: US7 (Error Handling) - T044-T049
- Developer B: US8 (Frontend Polish) - T050-T055
- Developer C: Start US4 (Deletion) - T056-T057

**Sprint 3: Enhanced Features**
- Developer A: US6 (Context) - T058-T060
- Developer B: Finish US4 - T056-T057
- Developer C: Polish tasks - T061-T074

---

## Task Count Summary

- **Phase 1 (Setup)**: 5 tasks
- **Phase 2 (Foundational)**: 20 tasks (BLOCKING)
- **Phase 3 (US1 - Task Creation - P1 MVP)**: 11 tasks
- **Phase 4 (US2 - Task Updates - P1 MVP)**: 2 tasks
- **Phase 5 (US3 - Task Completion - P1 MVP)**: 2 tasks
- **Phase 6 (US5 - Task Listing - P1 MVP)**: 3 tasks
- **Phase 7 (US7 - Error Handling - P1 MVP)**: 6 tasks
- **Phase 8 (US8 - Frontend Integration - P1 MVP)**: 6 tasks
- **Phase 9 (US4 - Task Deletion - P2)**: 2 tasks
- **Phase 10 (US6 - Context - P2)**: 3 tasks
- **Phase 11 (Polish)**: 14 tasks

**Total**: 74 tasks

**MVP Scope (P1 stories only)**: Phase 1 + 2 + 3 + 4 + 5 + 6 + 7 + 8 = 55 tasks

---

## Parallel Opportunities Identified

- **Setup Phase**: 5 parallel tasks
- **Foundational Phase**: Up to 15 parallel tasks (across 4 streams)
- **US1 Frontend**: 3 parallel tasks (T031, T032, T033)
- **Polish Phase**: 13 parallel tasks

**Estimated Time Savings**: With 3 developers working in parallel, MVP can be completed in ~40% less time compared to sequential execution.

---

## Notes

- [P] tasks = different files, no dependencies, can run in parallel
- [Story] label (US1, US2, etc.) maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Manual testing checklists provided for each user story
- Stop at any checkpoint to validate story independently before proceeding
- Avoid: vague tasks, same file conflicts, cross-story dependencies that break independence
- No automated test tasks per spec requirements (manual testing only for MVP)
- All file paths are relative to repository root
- Backend paths: backend/app/...
- Frontend paths: frontend/app/, frontend/components/, frontend/lib/
- Foundational phase (Phase 2) is CRITICAL and BLOCKS all user story work
