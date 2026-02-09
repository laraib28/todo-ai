---
name: task-matching
description: AI task matching and natural language processing
---

# Task Matching

The AI agent matches natural language input to todo operations.

## Supported intents
- **Add task**: "add", "create", "new task"
- **List tasks**: "show", "list", "what are my tasks"
- **Complete task**: "complete", "done", "finish"
- **Delete task**: "remove", "delete"
- **Search**: "find", "search"

## Testing
```bash
curl -X POST http://localhost:8000/api/chat \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"message": "add a task to buy groceries with high priority"}'
```
