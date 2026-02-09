# Todo AI - Phase 3

AI-powered chatbot with OpenAI integration, MCP tools, and natural language task management.

## Quick Start

### Backend
```bash
cd backend
pip install -r requirements.txt
# Set OPENAI_API_KEY in .env
uvicorn app.main:app --reload
```

### Frontend
```bash
cd frontend
npm install
npm run dev
```

## Architecture
- **Backend**: FastAPI + SQLAlchemy + PostgreSQL + JWT auth
- **Frontend**: Next.js 16 + TypeScript + Tailwind CSS
- **AI**: OpenAI SDK for natural language understanding
- **MCP**: Model Context Protocol tools for task operations
- **Chat**: Real-time chat interface for AI task management

## Key Components
- `backend/app/ai/agent.py` - AI agent with OpenAI integration
- `backend/app/ai/prompts.py` - System prompts for the AI
- `backend/app/mcp/server.py` - MCP server implementation
- `backend/app/mcp/tools.py` - MCP tool definitions
- `backend/app/routers/chat.py` - Chat API endpoints
- `frontend/components/ChatContainer.tsx` - Chat UI container

## Environment Variables
- `DATABASE_URL` - PostgreSQL connection
- `SECRET_KEY` - JWT secret
- `OPENAI_API_KEY` - OpenAI API key
