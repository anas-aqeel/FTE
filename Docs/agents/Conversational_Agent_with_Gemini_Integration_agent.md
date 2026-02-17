# Agent Specification: Conversational Agent with Gemini Integration

## 1. Purpose

Implement the core conversational agent that powers the chat interface by querying Supabase for relevant user data (filtered tables with fallback to raw tables), constructing context-aware prompts, calling Google Gemini API to generate natural language responses, and maintaining conversation history in the Personal Assistant Agent system.

## 2. Scope

**In Scope:**
- Chat endpoint: POST /conversations/{id}/messages
  - Accepts user message, returns AI-generated response
  - Saves both user message and assistant response to conversation_messages table
- Conversational agent logic (7-step process):
  1. Retrieve conversation history from conversation_messages table (last 10 messages for context)
  2. Query filtered data tables (emails, events, assignments, announcements, whatsapp_summaries) filtered by user_id, ordered by importance_score DESC
  3. If insufficient data in filtered tables (< 5 items), fallback to raw tables (raw_emails, raw_events, etc.)
  4. Construct Gemini prompt with: system instructions + conversation history + retrieved data + user query
  5. Call Gemini API (gemini-1.5-flash or gemini-1.5-pro)
  6. Save user message to conversation_messages (role=user)
  7. Save assistant response to conversation_messages (role=assistant)
- Query logic for filtered data:
  - Filter by user_id (data isolation)
  - Order by importance_score DESC, then by timestamp DESC
  - Limit to recent items (last 30 days)
  - Include all data types (emails, events, assignments, announcements, WhatsApp summaries)
- Raw data fallback logic:
  - Query raw tables if filtered tables return < 5 items
  - Apply same filters (user_id, timestamp)
  - Limit to 20 items (avoid overwhelming prompt)
- Prompt engineering for Gemini:
  - System instructions: "You are a personal assistant helping the user stay organized..."
  - Include conversation history for context
  - Format retrieved data as structured JSON or markdown
  - Include user query
- Error handling for Gemini API failures (rate limits, invalid responses)
- Streaming responses (optional, Phase 2)
- Response time optimization (<5 seconds end-to-end)

**Out of Scope:**
- Conversation management endpoints (already implemented in previous ticket)
- Data ingestion (Celery workers)
- Frontend chat interface (separate ticket)
- Advanced features: multi-modal input (images, voice), function calling, code execution

## 3. Inputs

- **HTTP Request** (POST /conversations/{id}/messages):
  - Path param: id (conversation UUID)
  - Headers: Authorization: Bearer {jwt_token}
  - Body: {content: string} (user message, required)
- **Database Tables** (read):
  - conversations: Verify conversation belongs to user
  - conversation_messages: Fetch conversation history (last 10 messages)
  - Filtered tables: emails, events, assignments, announcements, whatsapp_summaries
  - Raw tables (fallback): raw_emails, raw_events, raw_assignments, raw_classroom_announcements, raw_messages
- **Environment Variables**:
  - GEMINI_API_KEY: Google Gemini API key
  - GEMINI_MODEL: Model name (default: gemini-1.5-flash)
  - CONVERSATION_HISTORY_LIMIT: Number of past messages to include (default: 10)
  - FILTERED_DATA_LIMIT: Max items from filtered tables (default: 20)
  - RAW_DATA_FALLBACK_LIMIT: Max items from raw tables (default: 20)
- **JWT Token**: Decoded to extract user_id for data filtering

## 4. Outputs

- **API Response**:
  ```json
  {
    "message_id": "uuid",
    "conversation_id": "uuid",
    "role": "assistant",
    "content": "Based on your calendar, you have 3 events today: 1) Team meeting at 10 AM...",
    "created_at": "2026-02-17T10:05:15Z"
  }
  ```
- **Database Writes**:
  - conversation_messages table: INSERT user message (role=user, content)
  - conversation_messages table: INSERT assistant response (role=assistant, content)
  - conversations table: UPDATE updated_at timestamp
- **Gemini API Call**:
  - Request: Prompt with system instructions, history, data, user query
  - Response: Natural language text response
- **Logs**:
  - Chat request received (conversation_id, user_id, message length)
  - Conversation history retrieved (message count)
  - Filtered data queried (item counts per table)
  - Raw data fallback triggered (if applicable)
  - Gemini API call (model, latency, token count)
  - Response saved (message_id, length)
  - Total response time
  - Errors (validation, database, Gemini API)

## 5. Internal Responsibilities

1. **Endpoint Definition**: Define POST /conversations/{id}/messages in routers/conversations.py
2. **Authorization**: Use get_current_user dependency, verify conversation belongs to user
3. **Input Validation**: Validate message content (not empty, max length 5000 chars)
4. **Retrieve Conversation History**: Query conversation_messages table for conversation_id, order by created_at ASC, limit to last 10 messages
5. **Query Filtered Data**:
   - Query emails table: WHERE user_id={user_id} AND received_at > NOW() - INTERVAL '30 days' ORDER BY importance_score DESC, received_at DESC LIMIT 5
   - Query events table: WHERE user_id={user_id} AND start_time > NOW() - INTERVAL '7 days' ORDER BY importance_score DESC, start_time ASC LIMIT 5
   - Query assignments table: WHERE user_id={user_id} AND (due_date > NOW() OR status='pending') ORDER BY importance_score DESC, due_date ASC LIMIT 5
   - Query announcements table: WHERE user_id={user_id} AND announced_at > NOW() - INTERVAL '7 days' ORDER BY importance_score DESC, announced_at DESC LIMIT 5
   - Query whatsapp_summaries table: WHERE user_id={user_id} AND batch_end_time > NOW() - INTERVAL '7 days' ORDER BY importance_score DESC, batch_end_time DESC LIMIT 5
6. **Raw Data Fallback**: If total filtered items < 5, query raw tables with same filters (limit 20 items)
7. **Construct Gemini Prompt**:
   - System instructions: Define agent persona and capabilities
   - Conversation history: Format as "User: ... \n Assistant: ..."
   - Retrieved data: Format as JSON or markdown sections (Emails: [...], Events: [...], etc.)
   - User query: Include user's message at end
   - Example prompt structure:
     ```
     System: You are a personal assistant. Help the user stay organized by answering questions about their emails, calendar, assignments, and messages.

     Conversation History:
     User: What's my schedule today?
     Assistant: You have 3 events today...

     Retrieved Data:
     Emails (3):
     - Subject: Project deadline reminder, From: Alice, Importance: 0.8
     Events (2):
     - Title: Team meeting, Time: 10:00 AM, Importance: 0.7
     Assignments (1):
     - Title: Math homework, Due: 2026-02-20, Importance: 0.9

     User Query: {user_message}
     ```
8. **Call Gemini API**:
   - Use google-generativeai Python SDK
   - Model: gemini-1.5-flash (fast, cost-effective)
   - Temperature: 0.3 (balance creativity and consistency)
   - Max output tokens: 1000
   - Handle errors: rate limits (retry with backoff), invalid API key (log and alert), timeouts (retry once)
9. **Save Messages to Database**:
   - INSERT user message: conversation_id, role='user', content={user_message}, created_at=NOW()
   - INSERT assistant response: conversation_id, role='assistant', content={gemini_response}, created_at=NOW()
   - UPDATE conversations table: SET updated_at=NOW() WHERE id={conversation_id}
10. **Return Response**: Send assistant message as JSON response

## 6. Dependencies

**External Services:**
- Google Gemini API (requires GEMINI_API_KEY)
- Supabase PostgreSQL (all data tables)

**Libraries/Tools:**
- FastAPI (already set up)
- google-generativeai Python SDK
- Supabase Python client

**Ticket Dependencies:**
- FastAPI_Project_Setup_&_Authentication (provides auth middleware)
- Conversation_Management_API (provides conversation endpoints)
- All database tickets (provides data tables)
- All ingestion tickets (provides data to query)

**Blocks:**
- Next.js_Frontend_-_Chat_Interface_&_Conversation_Management (needs chat endpoint)

## 7. Execution Model

**Type**: HTTP Service (Synchronous REST API)

**Lifecycle**: Endpoint always available

**Concurrency**: Asynchronous (FastAPI async def)

**Response Time**: 2-5 seconds (database queries + Gemini API call + save)

**Timeout**: 30 seconds (Gemini API may take 5-10 seconds for long prompts)

## 8. Failure Handling

**Gemini API Failures:**
- Rate Limit (429): Retry with exponential backoff (max 2 retries), return 503 if still fails
- Invalid API Key (401): Log error, alert, return 500
- Network Timeout: Retry once with increased timeout, return 500 if fails
- Invalid Response: Parse error, log raw response, return generic error message

**Database Errors:**
- Connection failure: Return 500, retry if transient
- Query errors: Log error, return 500

**Authorization Failures:**
- Conversation doesn't belong to user: Return 403 Forbidden
- Conversation not found: Return 404 Not Found

**Validation Errors:**
- Empty message: Return 400 Bad Request
- Message too long (>5000 chars): Return 400

**Insufficient Data:**
- If both filtered and raw tables empty: Still call Gemini (agent can say "I don't have any data yet")

## 9. Observability

**Logging:**
- Chat requests (conversation_id, user_id, message_length)
- Data queries (table, item_count, query_time)
- Fallback triggered (reason, fallback_count)
- Gemini API calls (model, prompt_tokens, response_tokens, latency)
- Responses saved (message_id, response_length)
- Total response time (breakdown: db_query_time, gemini_time, save_time)
- Errors (type, context, stack trace)

**Metrics** (if added):
- chat_requests_total (counter)
- chat_response_time_seconds (histogram, labels: status)
- gemini_api_calls_total (counter, labels: model, status)
- gemini_api_latency_seconds (histogram)
- gemini_tokens_used_total (counter, labels: type=prompt/response)
- data_queries_total (counter, labels: table, fallback)

## 10. Security Considerations

**Authorization:**
- Enforce conversation ownership (user_id check)
- Filter all data queries by user_id (prevent cross-user data leaks)

**Data Privacy:**
- Retrieved data may contain sensitive information (emails, messages)
- Data sent to Gemini API (Google's servers)
- Consider data retention policy for conversation history

**API Key Security:**
- GEMINI_API_KEY stored in environment variable
- Never log API key
- Monitor API usage for anomalies

**Input Validation:**
- Sanitize user messages (prevent prompt injection attacks)
- Limit message length (prevent resource exhaustion)

**Rate Limiting** (future):
- Limit chat requests per user (e.g., 100 requests/hour)
- Prevent abuse of Gemini API quota

## 11. Scaling Considerations

**Current MVP:**
- Single user
- Low chat frequency (~10-50 messages/day)
- Modest data volume

**Performance:**
- Database queries: <200ms (optimized with indexes)
- Gemini API: 2-5 seconds (dominant factor)
- Total response time: <6 seconds

**Optimization Opportunities:**
- Parallel data queries (fetch from all tables concurrently)
- Cache filtered data (Redis, 5-minute TTL)
- Use faster Gemini model (gemini-1.5-flash)
- Reduce prompt size (summarize data before including)

**Future Scaling:**
- Multi-user: Concurrent Gemini API calls (no blocking)
- High volume: Implement request queuing (max 5 concurrent per user)
- Cost management: Monitor token usage, implement per-user quotas

## 12. Future Extensions

**Phase 2:**
- Streaming responses (Server-Sent Events or WebSocket)
- Multi-modal input (images, voice messages)
- Function calling (allow Gemini to trigger actions: mark as seen, create reminder)
- Conversation summarization (long conversations)
- Context window management (intelligent pruning of old messages)
- User feedback loop (thumbs up/down, improve responses)
- Custom system instructions per user
- Multi-turn tool use (Gemini requests more data, agent fetches, Gemini responds)
- Code execution (Gemini generates code, agent runs safely)
- External knowledge integration (web search, Wikipedia)
