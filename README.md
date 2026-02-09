# Todo AI - Phase 3

An AI-powered Todo application with natural language task management using OpenAI and MCP tools.

## Features
- Everything from Phase 2 (web app, auth, CRUD)
- AI chatbot for natural language task management
- OpenAI integration for intent parsing
- MCP (Model Context Protocol) tools
- Real-time chat interface

## Tech Stack
- **Backend**: Python, FastAPI, SQLAlchemy, PostgreSQL
- **Frontend**: Next.js 16, TypeScript, Tailwind CSS
- **AI**: OpenAI SDK
- **MCP**: Model Context Protocol for tool execution

## Quick Start

### Backend
```bash
cd backend
pip install -r requirements.txt
# Configure .env: DATABASE_URL, SECRET_KEY, OPENAI_API_KEY
uvicorn app.main:app --reload
```

### Frontend
```bash
cd frontend
npm install
npm run dev
```

## Project Structure
```
backend/
├── app/
│   ├── ai/            # AI agent & prompts
│   ├── mcp/           # MCP server & tools
│   ├── routers/
│   │   ├── auth.py
│   │   ├── tasks.py
│   │   └── chat.py    # Chat endpoint
│   └── models.py
frontend/
├── app/chat/          # Chat page
├── components/
│   ├── ChatContainer.tsx
│   ├── ChatInput.tsx
│   └── ChatMessage.tsx
```

## Built with Claude Code + SpecKit
