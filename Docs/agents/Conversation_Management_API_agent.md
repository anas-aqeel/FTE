# Agent Specification: Conversation_Management_API

## 1. Purpose

Implement RESTful API endpoints in FastAPI for managing conversation threads and message history, enabling users to create, list, retrieve, and delete conversations with full message history in the Personal Assistant Agent system.

## 2. Scope

**In Scope:**
- Conversation management endpoints:
  - GET /conversations - list all user's conversations (paginated)
  - POST /conversations - create new conversation with optional title
  - GET /conversations/{id}/messages - get conversation history with messages (paginated)
  - DELETE /conversations/{id} - delete conversation and all messages
- Database operations on conversations and conversation_messages tables
- Auto-generate conversation titles from first user message (if not provided)
- Pagination for conversation list and message history
- Authorization: users can only access their own conversations (enforce user_id filtering)
- Input validation using Pydantic models
- Error handling for not found, unauthorized access, validation errors
- Unit tests for endpoints

**Out of Scope:**
- Chat message creation endpoint (POST /conversations/{id}/messages - handled in Conversational Agent ticket)
- Conversation update/rename (Phase 2 feature)
- Message editing or deletion (Phase 2 feature)
- Real-time updates via WebSocket (Phase 2 feature)
- Search functionality (Phase 2 feature)

## 3. Inputs

- **HTTP Requests**:
  - GET /conversations?page=1&limit=20
    - Query params: page (default: 1), limit (default: 20, max: 100)
    - Headers: Authorization: Bearer {jwt_token}
  - POST /conversations
    - Headers: Authorization: Bearer {jwt_token}
    - Body: {title?: string} (optional, auto-generated if not provided)
  - GET /conversations/{id}/messages?page=1&limit=50
    - Path param: id (conversation UUID)
    - Query params: page (default: 1), limit (default: 50, max: 200)
    - Headers: Authorization: Bearer {jwt_token}
  - DELETE /conversations/{id}
    - Path param: id (conversation UUID)
    - Headers: Authorization: Bearer {jwt_token}
- **Database Tables** (read):
  - conversations: user_id, title, created_at, updated_at
  - conversation_messages: conversation_id, role, content, created_at
  - Filter by user_id (from JWT token) for authorization
- **JWT Token**: Decoded to extract user_id for data isolation

## 4. Outputs

- **API Responses**:
  - GET /conversations:
    ```json
    {
      "conversations": [
        {
          "id": "uuid",
          "title": "Project Planning",
          "created_at": "2026-02-17T10:00:00Z",
          "updated_at": "2026-02-17T14:30:00Z",
          "message_count": 15
        }
      ],
      "total": 3,
      "page": 1,
      "limit": 20
    }
    ```
  - POST /conversations:
    ```json
    {
      "id": "uuid",
      "title": "New Conversation",
      "created_at": "2026-02-17T15:00:00Z",
      "updated_at": "2026-02-17T15:00:00Z"
    }
    ```
  - GET /conversations/{id}/messages:
    ```json
    {
      "conversation_id": "uuid",
      "messages": [
        {
          "id": "uuid",
          "role": "user",
          "content": "What's on my schedule today?",
          "created_at": "2026-02-17T10:05:00Z"
        },
        {
          "id": "uuid",
          "role": "assistant",
          "content": "You have 3 events today...",
          "created_at": "2026-02-17T10:05:15Z"
        }
      ],
      "total": 15,
      "page": 1,
      "limit": 50
    }
    ```
  - DELETE /conversations/{id}:
    ```json
    {
      "success": true,
      "message": "Conversation deleted successfully"
    }
    ```
- **Database Operations**:
  - conversations table: INSERT, SELECT, DELETE
  - conversation_messages table: SELECT (DELETE via CASCADE foreign key)
- **Logs**:
  - API request received (endpoint, user_id, params)
  - Conversation created (conversation_id, user_id)
  - Conversation deleted (conversation_id, user_id, message_count)
  - Authorization failures (user_id, conversation_id)
  - Validation errors (endpoint, error details)

## 5. Internal Responsibilities

1. **Router Setup**: Create routers/conversations.py, mount to FastAPI app at /conversations
2. **Pydantic Models**: Define request/response models (ConversationCreate, ConversationResponse, MessageResponse)
3. **GET /conversations**: Query conversations table filtered by user_id, paginate with LIMIT/OFFSET, include message count via JOIN or subquery, order by updated_at DESC
4. **POST /conversations**: Validate input, INSERT into conversations table with user_id from JWT, auto-generate title as "Conversation {timestamp}" if not provided, return created conversation
5. **GET /conversations/{id}/messages**: Verify conversation belongs to user (user_id check), query conversation_messages table ordered by created_at ASC, paginate, return messages
6. **DELETE /conversations/{id}**: Verify conversation belongs to user, DELETE from conversations table (CASCADE deletes messages), return success
7. **Authorization Middleware**: Use get_current_user dependency from Auth ticket, extract user_id, filter all queries by user_id
8. **Error Handling**: Return 404 if conversation not found, 403 if user doesn't own conversation, 422 for validation errors
9. **Pagination Helper**: Reusable pagination function (calculate offset, validate page/limit params)
10. **Unit Tests**: Test each endpoint with valid/invalid inputs, test authorization, test pagination

## 6. Dependencies

**External Services:**
- Supabase PostgreSQL (conversations, conversation_messages tables)

**Libraries/Tools:**
- FastAPI (already set up)
- Supabase Python client

**Ticket Dependencies:**
- FastAPI_Project_Setup_&_Authentication (provides get_current_user dependency, JWT validation)
- Setup_Supabase_Project_&_Core_Tables (provides conversations, conversation_messages tables)

**Blocks:**
- Conversational_Agent_with_Gemini_Integration (needs conversation endpoints to function)
- Next.js_Frontend_-_Chat_Interface_&_Conversation_Management (needs API endpoints)

## 7. Execution Model

**Type**: HTTP Service (Synchronous REST API)

**Lifecycle**: Endpoints always available as part of FastAPI app

**Concurrency**: Asynchronous request handling (FastAPI ASGI)

**Response Time**: <100ms for conversation list, <200ms for message history (depends on pagination limit)

## 8. Failure Handling

**Database Errors:**
- Connection failures: Return 500 Internal Server Error, retry if transient
- Query errors: Log error, return 500

**Authorization Failures:**
- Invalid JWT: Return 401 Unauthorized (handled by Auth middleware)
- User doesn't own conversation: Return 403 Forbidden

**Not Found Errors:**
- Conversation doesn't exist: Return 404 Not Found

**Validation Errors:**
- Invalid UUID format: Return 422 Unprocessable Entity
- Invalid pagination params: Return 400 Bad Request

## 9. Observability

**Logging:**
- API requests (method, path, user_id, response_time)
- Conversation operations (created, deleted, accessed)
- Authorization checks (user_id, conversation_id, allowed/denied)
- Errors (type, context, stack trace)

**Metrics** (if added):
- conversation_requests_total (counter, labels: endpoint, status)
- conversation_response_time_seconds (histogram, labels: endpoint)
- conversations_created_total (counter)
- conversations_deleted_total (counter)

## 10. Security Considerations

**Authorization:**
- Enforce user_id filtering on all queries (prevent cross-user data access)
- Verify conversation ownership before read/delete operations

**Data Privacy:**
- Conversation messages may contain sensitive data
- Only user who owns conversation can access messages
- No public access to conversations (JWT required)

**Input Validation:**
- Validate all UUIDs (prevent SQL injection via malformed IDs)
- Validate pagination params (prevent resource exhaustion)
- Sanitize user-provided titles (prevent XSS if rendered in UI)

## 11. Scaling Considerations

**Current MVP:**
- Single user
- Low conversation count (~10-50 conversations)
- Low message volume per conversation (~50-500 messages)

**Performance:**
- Conversation list: O(1) with pagination and indexes on user_id, updated_at
- Message history: O(n) where n=limit (50-200 messages)

**Future Scaling:**
- Add database indexes on user_id, updated_at, created_at
- Implement caching for frequently accessed conversations (Redis)
- Use cursor-based pagination for large datasets (more efficient than OFFSET)

## 12. Future Extensions

**Phase 2:**
- Conversation update/rename (PUT /conversations/{id})
- Conversation archiving (PATCH /conversations/{id}/archive)
- Search conversations by title or content
- Real-time updates via WebSocket (new messages, conversation updates)
- Message editing/deletion
- Conversation sharing (multi-user access)
- Conversation folders/tags
- Export conversation to PDF/text
