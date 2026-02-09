# Quickstart Guide - Phase III: AI-Powered Todo Chatbot

## Overview

This guide will help you set up and run the Phase III AI-Powered Todo Chatbot feature. Follow these steps in order to get the chatbot up and running locally.

---

## Prerequisites

Before starting, ensure you have the following installed:

- **Python 3.11+** (for backend)
- **Node.js 20+** (for frontend)
- **PostgreSQL** (via Neon or local instance)
- **OpenAI API Key** (for GPT-4)
- **Git** (for version control)

**Phase I & II Must Be Complete**:
- Backend API running on `http://localhost:8000`
- Frontend running on `http://localhost:3000`
- User authentication working (JWT cookies)
- Task CRUD operations functional

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                   Frontend (Next.js)                        │
│  - Chat UI component                                        │
│  - POST /api/chat requests                                  │
│  - Real-time message display                                │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        │ HTTP + JWT Cookie
                        │
┌───────────────────────▼─────────────────────────────────────┐
│              Backend (FastAPI)                              │
│  - POST /api/chat endpoint                                  │
│  - JWT authentication                                       │
│  - ChatService orchestration                                │
└───────────────┬───────────────────────┬─────────────────────┘
                │                       │
                │                       │
    ┌───────────▼─────────┐  ┌─────────▼──────────┐
    │  OpenAI Agent       │  │   MCP Server       │
    │  - GPT-4            │  │   - Tool registry  │
    │  - Function calling │  │   - User context   │
    └───────────┬─────────┘  └──────────┬─────────┘
                │                       │
                │                       │
                └───────────┬───────────┘
                            │
                ┌───────────▼─────────────────────┐
                │  Neon PostgreSQL Database       │
                │  - tasks table                  │
                │  - conversation_history table   │
                └─────────────────────────────────┘
```

---

## Step 1: Clone Repository and Checkout Branch

```bash
# Navigate to project directory
cd /path/to/to-do

# Ensure you're on the correct branch
git checkout 002-phase2-web

# Create Phase III branch
git checkout -b 003-phase3-ai-chatbot
```

---

## Step 2: Backend Setup

### 2.1 Install Dependencies

```bash
cd backend

# Install new dependencies for Phase III
pip install openai-agents-sdk==0.6.4
pip install mcp-sdk==1.0.0
pip install alembic==1.13.0
```

Update `requirements.txt`:
```txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
sqlmodel==0.0.8
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
python-dotenv==1.0.0
alembic==1.13.0
openai-agents-sdk==0.6.4
mcp-sdk==1.0.0
slowapi==0.1.9
```

### 2.2 Configure Environment Variables

Add to `backend/.env`:

```bash
# OpenAI Configuration
OPENAI_API_KEY=sk-your-openai-api-key-here
OPENAI_MODEL=gpt-4o

# MCP Server Configuration
MCP_SERVER_PORT=5000

# Rate Limiting
RATE_LIMIT_PER_MINUTE=60
```

**Get OpenAI API Key**:
1. Go to [platform.openai.com](https://platform.openai.com)
2. Create account or sign in
3. Navigate to API Keys
4. Create new secret key
5. Copy key to `.env` file

### 2.3 Database Migration

```bash
# Navigate to backend directory
cd backend

# Create Alembic migration for conversation_history table
alembic revision -m "Add conversation_history table"

# Edit the generated migration file (in backend/alembic/versions/)
# Copy the upgrade/downgrade code from data-model.md

# Run migration
alembic upgrade head

# Verify migration
psql $DATABASE_URL -c "SELECT * FROM conversation_history LIMIT 1;"
```

**Migration Code** (add to generated migration file):
```python
def upgrade():
    op.create_table(
        'conversation_history',
        sa.Column('id', sa.Integer(), nullable=False),
        sa.Column('user_id', sa.Integer(), nullable=False),
        sa.Column('role', sa.String(length=20), nullable=False),
        sa.Column('content', sa.Text(), nullable=False),
        sa.Column('created_at', sa.DateTime(), nullable=False),
        sa.ForeignKeyConstraint(['user_id'], ['users.id'], ),
        sa.PrimaryKeyConstraint('id')
    )
    op.create_index('ix_conversation_history_user_id', 'conversation_history', ['user_id'])
    op.create_index('ix_conversation_history_user_id_created_at',
                    'conversation_history', ['user_id', 'created_at'],
                    postgresql_ops={'created_at': 'DESC'})

def downgrade():
    op.drop_index('ix_conversation_history_user_id_created_at', table_name='conversation_history')
    op.drop_index('ix_conversation_history_user_id', table_name='conversation_history')
    op.drop_table('conversation_history')
```

---

## Step 3: Verify Database Schema

```bash
# Connect to database
psql $DATABASE_URL

# List tables
\dt

# Expected output:
#  Schema |        Name            | Type  | Owner
# --------+------------------------+-------+-------
#  public | users                  | table | ...
#  public | tasks                  | table | ...
#  public | conversation_history   | table | ...

# Describe conversation_history table
\d conversation_history

# Expected columns:
#  Column     |            Type             | Nullable
# ------------+-----------------------------+----------
#  id         | integer                     | not null
#  user_id    | integer                     | not null
#  role       | character varying(20)       | not null
#  content    | text                        | not null
#  created_at | timestamp without time zone | not null

# Exit psql
\q
```

---

## Step 4: Start MCP Server

```bash
# Navigate to backend directory
cd backend

# Start MCP server (in separate terminal)
python -m app.mcp_server

# Expected output:
# INFO:     MCP Server started on port 5000
# INFO:     Registered 6 tools: create_task, list_tasks, update_task, ...
```

**Keep this terminal running** - the MCP server must be active for chat to work.

---

## Step 5: Start Backend API

```bash
# Navigate to backend directory (in new terminal)
cd backend

# Start FastAPI server
uvicorn app.main:app --reload --port 8000

# Expected output:
# INFO:     Uvicorn running on http://127.0.0.1:8000
# INFO:     Application startup complete.
```

**Verify Backend**:
```bash
# Health check
curl http://localhost:8000/health

# Expected: {"status": "ok"}

# API docs
open http://localhost:8000/docs
```

---

## Step 6: Frontend Setup

### 6.1 Install Dependencies

```bash
cd frontend

# Install chat UI dependencies
npm install

# No new dependencies needed - using existing Next.js, React, Tailwind
```

### 6.2 Update Environment Variables

Add to `frontend/.env.local`:

```bash
NEXT_PUBLIC_API_URL=http://localhost:8000
```

### 6.3 Start Frontend

```bash
# Navigate to frontend directory
cd frontend

# Start Next.js development server
npm run dev

# Expected output:
# ▲ Next.js 14.0.0
# - Local:        http://localhost:3000
# - Ready in 2.1s
```

**Verify Frontend**:
```bash
open http://localhost:3000
```

---

## Step 7: Test Chat Endpoint

### 7.1 Register/Login User

```bash
# Register new user
curl -X POST http://localhost:8000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "testpassword123"
  }'

# Expected: {"id": 1, "email": "test@example.com", ...}
# Note: JWT cookie will be set automatically
```

### 7.2 Test Chat Endpoint (with cookie)

```bash
# Save cookie from registration/login
COOKIE="access_token=eyJ..."

# Send chat message
curl -X POST http://localhost:8000/api/chat \
  -H "Content-Type: application/json" \
  -H "Cookie: $COOKIE" \
  -d '{
    "message": "Create a task to buy groceries tomorrow"
  }'

# Expected response:
# {
#   "message": "I've created a new task: 'Buy groceries' with medium priority for tomorrow.",
#   "metadata": {
#     "action": "task_created",
#     "task_id": 1
#   }
# }
```

### 7.3 Verify Task Created

```bash
# List tasks
curl -X GET http://localhost:8000/api/tasks \
  -H "Cookie: $COOKIE"

# Expected: Array of tasks including "Buy groceries"
```

### 7.4 Test Chat UI

1. Open browser: `http://localhost:3000`
2. Login with test credentials
3. Navigate to "Chat" page
4. Type: "Create a task to buy groceries"
5. Verify AI response appears
6. Verify task appears in task list

---

## Step 8: Verify All Systems

### 8.1 Check Running Processes

```bash
# Backend API (should be running on port 8000)
curl http://localhost:8000/health

# MCP Server (should be running on port 5000)
curl http://localhost:5000/health

# Frontend (should be running on port 3000)
curl http://localhost:3000
```

### 8.2 Check Database

```bash
# Connect to database
psql $DATABASE_URL

# Check conversation history
SELECT * FROM conversation_history ORDER BY created_at DESC LIMIT 5;

# Expected: Recent chat messages (user and assistant roles)

# Check tasks
SELECT * FROM tasks ORDER BY created_at DESC LIMIT 5;

# Expected: Tasks created via chat
```

---

## Troubleshooting

### Issue: "OpenAI API key not found"

**Solution**:
```bash
# Verify .env file exists
cat backend/.env | grep OPENAI_API_KEY

# If missing, add:
echo "OPENAI_API_KEY=sk-your-key-here" >> backend/.env

# Restart backend server
```

---

### Issue: "MCP Server not responding"

**Solution**:
```bash
# Check if MCP server is running
ps aux | grep mcp_server

# If not running, start it:
cd backend
python -m app.mcp_server

# Check logs for errors
```

---

### Issue: "Database migration failed"

**Solution**:
```bash
# Check Alembic status
cd backend
alembic current

# If behind, run migrations:
alembic upgrade head

# If errors, check connection:
psql $DATABASE_URL -c "SELECT 1;"
```

---

### Issue: "Chat returns 401 Unauthorized"

**Solution**:
```bash
# Verify JWT cookie is set
# In browser DevTools > Application > Cookies
# Should see: access_token=eyJ...

# If missing, login again:
curl -X POST http://localhost:8000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "test@example.com", "password": "testpassword123"}'
```

---

### Issue: "Chat returns 500 Internal Server Error"

**Solution**:
```bash
# Check backend logs
cd backend
tail -f logs/app.log

# Common causes:
# 1. MCP server not running
# 2. OpenAI API key invalid
# 3. Database connection failed

# Restart all services:
# 1. Stop MCP server (Ctrl+C)
# 2. Stop backend (Ctrl+C)
# 3. Start MCP server
# 4. Start backend
```

---

### Issue: "Rate limit exceeded"

**Solution**:
```bash
# Wait 1 minute before trying again
# Or increase rate limit in .env:
echo "RATE_LIMIT_PER_MINUTE=100" >> backend/.env

# Restart backend
```

---

## Development Workflow

### Making Changes

1. **Backend Changes**:
   ```bash
   cd backend
   # Edit files
   # FastAPI auto-reloads
   ```

2. **Frontend Changes**:
   ```bash
   cd frontend
   # Edit files
   # Next.js auto-reloads
   ```

3. **Database Changes**:
   ```bash
   cd backend
   alembic revision -m "Description"
   # Edit migration file
   alembic upgrade head
   ```

---

## Testing

### Run Backend Tests

```bash
cd backend
pytest tests/ -v

# Run specific test file
pytest tests/test_chat.py -v

# Run with coverage
pytest --cov=app tests/
```

### Run Frontend Tests

```bash
cd frontend
npm test

# Run specific test
npm test -- chat.test.tsx

# Run with coverage
npm test -- --coverage
```

---

## Deployment Checklist

Before deploying to production:

- [ ] Set `secure=True` for JWT cookies (HTTPS only)
- [ ] Set `CORS_ORIGINS` to production domain
- [ ] Enable rate limiting (60/min per user)
- [ ] Set up monitoring (Sentry, DataDog, etc.)
- [ ] Configure database backups
- [ ] Set up SSL certificates
- [ ] Review security headers (CSP, HSTS, etc.)
- [ ] Test with production OpenAI API key
- [ ] Load test chat endpoint (100+ concurrent users)
- [ ] Set up logging aggregation
- [ ] Configure auto-scaling for backend
- [ ] Set up health check endpoints
- [ ] Document incident response procedures

---

## Useful Commands

### Backend

```bash
# Start backend
cd backend && uvicorn app.main:app --reload

# Start MCP server
cd backend && python -m app.mcp_server

# Run migrations
cd backend && alembic upgrade head

# Run tests
cd backend && pytest tests/ -v

# Check code quality
cd backend && black . && flake8 .
```

### Frontend

```bash
# Start frontend
cd frontend && npm run dev

# Build for production
cd frontend && npm run build

# Run tests
cd frontend && npm test

# Lint code
cd frontend && npm run lint
```

### Database

```bash
# Connect to database
psql $DATABASE_URL

# Backup database
pg_dump $DATABASE_URL > backup.sql

# Restore database
psql $DATABASE_URL < backup.sql

# Reset database (DANGER!)
dropdb todo_db && createdb todo_db
cd backend && alembic upgrade head
```

---

## Next Steps

After completing this quickstart:

1. **Explore Chat Features**:
   - Try creating tasks with different priorities
   - Try listing tasks by filters
   - Try updating and completing tasks

2. **Review Documentation**:
   - Read `/specs/003-phase3-ai-chatbot/spec.md` for requirements
   - Read `/specs/003-phase3-ai-chatbot/plan.md` for architecture
   - Read `/specs/003-phase3-ai-chatbot/data-model.md` for database schema

3. **Customize Agent**:
   - Modify agent instructions in `app/services/chat_service.py`
   - Add new MCP tools in `app/mcp_server.py`
   - Update chat UI in `frontend/src/components/Chat.tsx`

4. **Monitor Performance**:
   - Check OpenAI API usage
   - Monitor database query performance
   - Track chat response times

---

## Support

For issues or questions:

1. Check troubleshooting section above
2. Review Phase III specification documents
3. Check backend logs: `backend/logs/app.log`
4. Check MCP server logs: `backend/logs/mcp.log`
5. Review OpenAI API status: [status.openai.com](https://status.openai.com)

---

## References

- **Phase III Spec**: `/specs/003-phase3-ai-chatbot/spec.md`
- **Phase III Plan**: `/specs/003-phase3-ai-chatbot/plan.md`
- **Data Model**: `/specs/003-phase3-ai-chatbot/data-model.md`
- **API Spec**: `/specs/003-phase3-ai-chatbot/contracts/api-spec.md`
- **MCP Tools Spec**: `/specs/003-phase3-ai-chatbot/contracts/mcp-tools-spec.md`
- **OpenAI Agents SDK**: [github.com/openai/openai-agents-python](https://github.com/openai/openai-agents-python)
- **MCP SDK**: [github.com/modelcontextprotocol/python-sdk](https://github.com/modelcontextprotocol/python-sdk)
- **FastAPI Docs**: [fastapi.tiangolo.com](https://fastapi.tiangolo.com)
- **Next.js Docs**: [nextjs.org/docs](https://nextjs.org/docs)

---

## License

MIT License - See LICENSE file for details.
