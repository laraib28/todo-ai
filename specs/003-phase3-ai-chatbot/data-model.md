# Data Model - Phase III: AI-Powered Todo Chatbot

## Overview

This document defines the database schema additions and modifications required for Phase III: AI-Powered Todo Chatbot feature.

## Existing Entities (Phase I & II)

### User
**Table**: `users`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | INTEGER | PRIMARY KEY, AUTO_INCREMENT | Unique user identifier |
| email | VARCHAR(255) | UNIQUE, NOT NULL, INDEX | User email address (lowercase) |
| hashed_password | VARCHAR(255) | NOT NULL | Bcrypt hashed password |
| created_at | TIMESTAMP | NOT NULL, DEFAULT utcnow() | Account creation timestamp |

**Relationships**:
- One-to-Many with Task (tasks)
- One-to-Many with ConversationHistory (conversation_history) [NEW]

**Validation Rules**:
- Email must be unique and valid format
- Password must be hashed before storage
- Email stored in lowercase for case-insensitive lookups

---

### Task
**Table**: `tasks`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | INTEGER | PRIMARY KEY, AUTO_INCREMENT | Unique task identifier |
| user_id | INTEGER | FOREIGN KEY(users.id), NOT NULL, INDEX | Owner of the task |
| title | VARCHAR(200) | NOT NULL, MIN_LENGTH=1 | Task title |
| description | VARCHAR(2000) | DEFAULT="" | Task description |
| priority | VARCHAR(10) | DEFAULT="medium" | Task priority (low/medium/high) |
| is_complete | BOOLEAN | DEFAULT=False | Task completion status |
| created_at | TIMESTAMP | NOT NULL, DEFAULT utcnow() | Task creation timestamp |
| updated_at | TIMESTAMP | NOT NULL, DEFAULT utcnow() | Last update timestamp |

**Relationships**:
- Many-to-One with User (user)

**Validation Rules**:
- Title must be 1-200 characters
- Description max 2000 characters
- Priority must be one of: "low", "medium", "high"
- All queries MUST filter by user_id for security

**State Transitions**:
- is_complete: False → True (mark complete)
- is_complete: True → False (mark incomplete)

---

## New Entities (Phase III)

### ConversationHistory
**Table**: `conversation_history`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | INTEGER | PRIMARY KEY, AUTO_INCREMENT | Unique message identifier |
| user_id | INTEGER | FOREIGN KEY(users.id), NOT NULL, INDEX | Owner of the conversation |
| role | VARCHAR(20) | NOT NULL | Message role: "user" or "assistant" |
| content | TEXT | NOT NULL | Message content |
| created_at | TIMESTAMP | NOT NULL, DEFAULT utcnow() | Message creation timestamp |

**Relationships**:
- Many-to-One with User (user)

**Validation Rules**:
- role must be one of: "user", "assistant"
- content cannot be empty
- All queries MUST filter by user_id for security
- Maximum 10 most recent messages per user loaded for context

**Indexes**:
- Composite index on (user_id, created_at DESC) for efficient recent message queries
- Single index on user_id for foreign key constraint

**Retention Policy**:
- Keep last 100 messages per user (configurable)
- Older messages can be archived or deleted
- MUST maintain chronological order by created_at

**Security**:
- Row-level security: ALWAYS filter by user_id from JWT
- No cross-user message access allowed
- Sanitize user input to prevent XSS

---

## Schema Diagram

```
┌─────────────────────┐
│       User          │
├─────────────────────┤
│ id (PK)             │
│ email (UNIQUE)      │
│ hashed_password     │
│ created_at          │
└──────────┬──────────┘
           │
           │ 1:N
           │
    ┌──────┴──────────────────────────────┐
    │                                     │
    │                                     │
┌───▼───────────────┐          ┌─────────▼──────────────┐
│      Task         │          │  ConversationHistory   │
├───────────────────┤          ├────────────────────────┤
│ id (PK)           │          │ id (PK)                │
│ user_id (FK)      │          │ user_id (FK)           │
│ title             │          │ role                   │
│ description       │          │ content                │
│ priority          │          │ created_at             │
│ is_complete       │          └────────────────────────┘
│ created_at        │
│ updated_at        │
└───────────────────┘
```

---

## SQLModel Implementation

### ConversationHistory Model

```python
from sqlmodel import Field, SQLModel, Relationship
from datetime import datetime
from typing import Optional

class ConversationHistory(SQLModel, table=True):
    """Conversation history model for AI chat sessions."""

    __tablename__ = "conversation_history"

    id: Optional[int] = Field(default=None, primary_key=True)
    user_id: int = Field(foreign_key="users.id", index=True)
    role: str = Field(max_length=20)  # "user" or "assistant"
    content: str = Field(sa_column_kwargs={"type_": Text})
    created_at: datetime = Field(default_factory=datetime.utcnow, index=True)

    # Relationship
    user: Optional["User"] = Relationship(back_populates="conversation_history")
```

### Updated User Model (add relationship)

```python
class User(SQLModel, table=True):
    """User model for authentication."""

    __tablename__ = "users"

    id: Optional[int] = Field(default=None, primary_key=True)
    email: str = Field(unique=True, index=True, max_length=255)
    hashed_password: str = Field(max_length=255)
    created_at: datetime = Field(default_factory=datetime.utcnow)

    # Relationships
    tasks: list["Task"] = Relationship(back_populates="user")
    conversation_history: list["ConversationHistory"] = Relationship(back_populates="user")  # NEW
```

---

## Migration Strategy

### Database Migration (Alembic)

**File**: `backend/alembic/versions/003_add_conversation_history.py`

```python
"""Add conversation_history table

Revision ID: 003
Revises: 002
Create Date: 2026-01-02

"""
from alembic import op
import sqlalchemy as sa

revision = '003'
down_revision = '002'

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

## Query Patterns

### Get Recent Conversation History

```python
from sqlmodel import Session, select, desc

def get_conversation_history(session: Session, user_id: int, limit: int = 10) -> list[ConversationHistory]:
    """Get recent conversation history for a user."""
    statement = (
        select(ConversationHistory)
        .where(ConversationHistory.user_id == user_id)
        .order_by(desc(ConversationHistory.created_at))
        .limit(limit)
    )
    messages = session.exec(statement).all()
    return list(reversed(messages))  # Return chronological order
```

### Save New Message

```python
def save_message(session: Session, user_id: int, role: str, content: str) -> ConversationHistory:
    """Save a new conversation message."""
    message = ConversationHistory(
        user_id=user_id,
        role=role,
        content=content
    )
    session.add(message)
    session.commit()
    session.refresh(message)
    return message
```

### Clear User Conversation History

```python
def clear_conversation_history(session: Session, user_id: int) -> int:
    """Clear all conversation history for a user."""
    statement = delete(ConversationHistory).where(ConversationHistory.user_id == user_id)
    result = session.exec(statement)
    session.commit()
    return result.rowcount
```

---

## Data Validation

### Role Validation

```python
from enum import Enum

class MessageRole(str, Enum):
    USER = "user"
    ASSISTANT = "assistant"

# Use in schema validation
from pydantic import BaseModel, validator

class ChatMessage(BaseModel):
    role: MessageRole
    content: str

    @validator('content')
    def content_not_empty(cls, v):
        if not v or not v.strip():
            raise ValueError('Message content cannot be empty')
        return v.strip()
```

---

## Security Considerations

1. **Row-Level Security**: ALWAYS filter by user_id from authenticated JWT
2. **Input Sanitization**: Sanitize all user input before storage
3. **XSS Prevention**: Escape HTML entities in message content before rendering
4. **SQL Injection**: Use parameterized queries (SQLModel handles this)
5. **Rate Limiting**: Limit conversation messages per user per minute

---

## Performance Considerations

1. **Indexing**: Composite index on (user_id, created_at DESC) for fast recent message queries
2. **Pagination**: Load only last 10 messages for context (configurable)
3. **Archival**: Archive messages older than 30 days to separate table
4. **Connection Pooling**: Use SQLAlchemy connection pool for Neon PostgreSQL
5. **Query Optimization**: Use LIMIT and indexes for all conversation queries

---

## Testing Requirements

### Unit Tests

- Validate ConversationHistory model creation
- Test role validation (only "user" and "assistant")
- Test content validation (non-empty)
- Test user_id foreign key constraint

### Integration Tests

- Test get_conversation_history with multiple users (no cross-user access)
- Test message ordering (chronological)
- Test conversation history limit (only last 10 messages)
- Test clear_conversation_history

### Security Tests

- Test user isolation (user A cannot access user B's messages)
- Test JWT authentication for conversation endpoints
- Test input sanitization (XSS prevention)

---

## Acceptance Criteria

- [ ] ConversationHistory table created in Neon PostgreSQL
- [ ] User model updated with conversation_history relationship
- [ ] Alembic migration created and tested
- [ ] Composite index on (user_id, created_at) created
- [ ] Query patterns tested for performance (< 100ms for recent messages)
- [ ] Row-level security enforced (user isolation)
- [ ] Unit tests passing (100% coverage for new model)
- [ ] Integration tests passing (conversation CRUD operations)
- [ ] Security tests passing (no cross-user access)

---

## Open Questions

1. **Message Retention**: How long should conversation history be retained? (Proposed: 30 days)
2. **Archive Strategy**: Should old messages be archived or deleted? (Proposed: Archive to separate table)
3. **Conversation Context Limit**: Should we allow users to configure context limit? (Proposed: Admin setting)

---

## Dependencies

- SQLModel 0.0.8+
- SQLAlchemy 2.0+
- Alembic 1.13+
- Neon PostgreSQL (Serverless Postgres)

---

## References

- [SQLModel Documentation](https://sqlmodel.tiangolo.com/)
- [Alembic Migrations](https://alembic.sqlalchemy.org/)
- [Neon PostgreSQL](https://neon.tech/docs)
- Phase III Spec: `/specs/003-phase3-ai-chatbot/spec.md`
- Phase III Plan: `/specs/003-phase3-ai-chatbot/plan.md`
