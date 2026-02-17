# Agent Specification: WhatsApp Service - Hourly Summarization & Backend Integration

## 1. Purpose

Implement hourly summarization logic using Gemini API to process accumulated WhatsApp message batches and integrate with FastAPI backend for data ingestion, enabling continuous WhatsApp monitoring with AI-generated summaries and importance scoring for the Personal Assistant Agent system.

This agent extends the WhatsApp Service from the previous ticket (Message Monitoring & Buffer Persistence) by adding AI processing and backend communication capabilities.

## 2. Scope

**In Scope:**
- Hourly summarization process (Node.js WhatsApp service):
  - At top of each hour (HH:00), select chats for processing:
    - All private chats (always summarized)
    - Allowlisted groups only (if whatsapp_group_allowlist is empty/null, no groups are summarized)
  - Fetch whatsapp_group_allowlist from backend API: GET /settings/whatsapp-allowlist
  - For each selected chat batch, call Gemini API to extract:
    - summary (2-3 sentences)
    - key_points (array of important items)
    - mentioned_deadlines (array of {task, deadline})
    - importance_score (0-1)
  - Build payload with raw messages + AI summary
  - Send payload to backend: POST /ingest/whatsapp
- Gemini API client integration (Node.js)
- Backend API client with authentication
- Failure recovery and resilience:
  - Resume from persisted buffer on restart
  - Retry backend POST with exponential backoff
  - Persist unsent payloads until acknowledged
  - Fallback: send raw batch if Gemini fails (backend will handle with lower importance)
- Backend ingestion endpoint implementation (FastAPI):
  - POST /ingest/whatsapp endpoint (requires API key authentication)
  - Save raw messages to raw_messages table
  - Save AI summary to whatsapp_summaries table
  - Handle duplicate detection (unique constraint on user_id, chat_name, batch_start_time, batch_end_time)
- Logging and monitoring for both services

**Out of Scope:**
- Message monitoring and buffer persistence (already implemented in previous ticket)
- OAuth refresh and cleanup tasks (separate ticket)
- Frontend components
- WhatsApp Business API integration (Phase 2)

## 3. Inputs

**WhatsApp Service (Node.js):**
- **Environment Variables**:
  - BACKEND_API_URL: FastAPI backend base URL (e.g., http://backend:8000)
  - GEMINI_API_KEY: Google Gemini API key
  - WHATSAPP_API_KEY: Shared secret for authenticating to backend
  - USER_ID: User UUID for data association (single-user MVP)
  - DATA_DIR: Local directory for buffer persistence (default: ./data)
- **Persisted Message Buffers**:
  - SQLite database or JSON files from previous ticket
  - Hourly batches per chat: {chat_name, is_group, messages[], batch_start_time, batch_end_time}
- **Backend API Responses**:
  - GET /settings/whatsapp-allowlist: {allowlist: string[]} (group chat names to summarize)
  - POST /ingest/whatsapp: {success: boolean, message: string}

**Backend API (FastAPI):**
- **HTTP Request** (POST /ingest/whatsapp):
  - Headers: X-API-Key: {WHATSAPP_API_KEY}
  - Body:
    ```json
    {
      "user_id": "uuid",
      "chat_name": "Project Group",
      "is_group": true,
      "messages": [
        {"sender": "Alice", "content": "...", "timestamp": "2026-02-17T10:05:00Z"},
        {"sender": "Bob", "content": "...", "timestamp": "2026-02-17T10:15:00Z"}
      ],
      "batch_start_time": "2026-02-17T10:00:00Z",
      "batch_end_time": "2026-02-17T11:00:00Z",
      "summary": "Alice and Bob discussed project timeline...",
      "key_points": ["Project deadline moved to Friday", "Need to submit draft by Thursday"],
      "mentioned_deadlines": [{"task": "Submit draft", "deadline": "2026-02-20T23:59:00Z"}],
      "importance_score": 0.75
    }
    ```
- **Database Tables** (read/write):
  - raw_messages: Write raw message batches
  - whatsapp_summaries: Write AI-generated summaries
  - user_settings: Read whatsapp_group_allowlist for filtering

## 4. Outputs

**WhatsApp Service (Node.js):**
- **Gemini API Calls**:
  - Request: Summarize chat batch (messages array, chat context)
  - Response: {summary, key_points, mentioned_deadlines, importance_score}
- **Backend API Calls**:
  - POST /ingest/whatsapp with payload (raw + summary)
  - Retry on failure (exponential backoff)
- **Persisted Unsent Payloads**:
  - SQLite table or JSON files: unsent_payloads with retry metadata
  - Cleared after successful backend acknowledgment
- **Logs**:
  - Hourly flush started (timestamp, chat count)
  - Allowlist fetched (group names)
  - Chat selected for summarization (chat_name, is_group, message count)
  - Gemini API call (chat_name, latency)
  - Backend POST (chat_name, status, retries)
  - Unsent payload persisted (chat_name, retry count)
  - Hourly flush completed (duration, summary)

**Backend API (FastAPI):**
- **Database Writes**:
  - raw_messages table: INSERT new row with message_batch JSONB, batch_start_time, batch_end_time
    - Use ON CONFLICT (user_id, chat_name, batch_start_time, batch_end_time) DO NOTHING for idempotency
  - whatsapp_summaries table: INSERT summary, key_points, mentioned_deadlines, importance_score, is_seen=FALSE
    - Use ON CONFLICT DO NOTHING (if duplicate via raw_message_id FK)
- **HTTP Response**:
  - 200 OK: {success: true, message: "Ingested successfully", raw_message_id: "uuid", summary_id: "uuid"}
  - 400 Bad Request: {success: false, error: "Missing required fields"}
  - 403 Forbidden: {success: false, error: "Invalid API key"}
  - 409 Conflict: {success: false, error: "Duplicate batch"} (idempotency, not a real error)
  - 500 Internal Server Error: {success: false, error: "Database error"}
- **Logs**:
  - WhatsApp ingestion request received (chat_name, message count, user_id)
  - Raw message inserted (raw_message_id)
  - Summary inserted (summary_id)
  - Duplicate detected (chat_name, batch times)
  - Errors (validation errors, database errors)

## 5. Internal Responsibilities

**WhatsApp Service (Node.js):**

1. **Hourly Flush Timer**:
   - Set up interval timer checking every minute (or use cron-like scheduler)
   - At top of hour (HH:00), trigger summarization process
   - Lock to prevent concurrent flushes (semaphore or flag)

2. **Fetch Allowlist from Backend**:
   - Call GET /settings/whatsapp-allowlist with X-API-Key header
   - Parse response: {allowlist: string[]} (group chat names)
   - Cache allowlist for 1 hour (refresh hourly)
   - If request fails: use cached allowlist, log warning
   - If allowlist empty/null: only summarize private chats (no groups)

3. **Select Chats for Summarization**:
   - Iterate through all chat buffers (in-memory or persisted)
   - For each chat:
     - If is_group=false (private chat): always include
     - If is_group=true: check if chat_name in allowlist, include if yes
   - Skip chats with 0 messages in current hour
   - Log selected chats (count, names)

4. **Gemini API Client Setup**:
   - Install @google/generative-ai library (Node.js SDK)
   - Initialize Gemini client with GEMINI_API_KEY
   - Set model: gemini-1.5-flash (cost-effective for summarization)
   - Configure temperature: 0 (consistent results)

5. **Summarize Chat Batch with Gemini**:
   - For each selected chat:
     - Build prompt: "Summarize this WhatsApp chat: Chat Name: {chat_name}, Messages: [{sender: '...', content: '...', timestamp: '...'}]. Extract: summary (2-3 sentences), key_points (array), mentioned_deadlines (array of {task, deadline}), importance_score (0-1)."
     - Call Gemini API with JSON response format
     - Parse response: {summary, key_points, mentioned_deadlines, importance_score}
     - Handle API errors (rate limits, invalid responses)
     - Retry failed API calls (max 2 retries with backoff)
     - If Gemini fails after retries: set summary=null, key_points=[], importance_score=0.3 (fallback)

6. **Build Backend Payload**:
   - For each processed chat:
     - Construct JSON payload with:
       - user_id (from env)
       - chat_name, is_group
       - messages array (from buffer)
       - batch_start_time, batch_end_time
       - summary, key_points, mentioned_deadlines, importance_score (from Gemini or fallback)

7. **Send Payload to Backend**:
   - Call POST /ingest/whatsapp with X-API-Key header
   - Send payload as JSON body
   - Handle responses:
     - 200 OK: Success, delete chat buffer from persistence
     - 409 Conflict: Duplicate (idempotency), treat as success, delete buffer
     - 4xx/5xx Error: Retry with exponential backoff (1s, 2s, 4s, max 3 retries)
   - Log backend POST results (status, retries)

8. **Persist Unsent Payloads**:
   - If backend POST fails after all retries:
     - Persist payload to unsent_payloads table/file
     - Include: payload JSON, retry_count, last_attempted_at
     - Log warning (unsent payload persisted)
   - On next hourly flush or service restart:
     - Load unsent_payloads
     - Retry sending to backend (before processing new batches)
     - Increment retry_count
     - Max retry attempts: 10 (after 10 hours, alert and discard)

9. **Clear Buffers After Successful Send**:
   - After backend acknowledges (200 OK or 409 Conflict):
     - Delete chat buffer from SQLite or JSON file
     - Remove entry from in-memory buffer map
     - Log buffer cleared (chat_name)

10. **Error Handling**:
    - Catch Gemini API errors: log, use fallback values, continue
    - Catch backend API errors: log, persist unsent payload, continue other chats
    - Catch network errors: retry with backoff, persist if all retries fail
    - Never lose data: always persist unsent payloads before giving up

**Backend API (FastAPI):**

1. **Endpoint Definition**:
   - Define POST /ingest/whatsapp endpoint in new router (routers/whatsapp.py)
   - Apply verify_api_key dependency (from FastAPI Auth ticket)
   - Accept WhatsAppIngestionRequest Pydantic model

2. **Request Validation**:
   - Validate required fields: user_id, chat_name, messages, batch_start_time, batch_end_time
   - Validate data types (user_id is UUID, timestamps are ISO 8601)
   - Validate messages array not empty
   - Return 400 Bad Request if validation fails

3. **Insert Raw Messages**:
   - Build raw_messages row:
     - user_id, chat_name, is_group, sender (null for group chats), message_batch (JSONB), batch_start_time, batch_end_time
   - INSERT into raw_messages table
   - Use ON CONFLICT (user_id, chat_name, batch_start_time, batch_end_time) DO NOTHING
   - Capture returned raw_message_id (or query if duplicate)
   - Log insertion (raw_message_id, duplicate status)

4. **Insert WhatsApp Summary**:
   - Build whatsapp_summaries row:
     - user_id, raw_message_id (FK), chat_name, is_group, summary, key_points, mentioned_deadlines, importance_score, is_seen=FALSE, batch_start_time, batch_end_time, processed_at=NOW()
   - INSERT into whatsapp_summaries table
   - Use ON CONFLICT DO NOTHING (if duplicate via raw_message_id FK)
   - Capture returned summary_id
   - Log insertion (summary_id)

5. **Handle Duplicates Gracefully**:
   - If INSERT returns 0 rows (duplicate):
     - Return 200 OK with message: "Duplicate batch (already ingested)"
     - Log duplicate detection (chat_name, batch times)
   - Idempotency: WhatsApp service can safely retry without creating duplicates

6. **Error Handling**:
   - Catch database errors: return 500 Internal Server Error
   - Catch validation errors: return 400 Bad Request with details
   - Catch unauthorized errors: return 403 Forbidden (invalid API key)
   - Log all errors with context (chat_name, user_id, error message)

7. **Response**:
   - Return 200 OK with JSON: {success: true, raw_message_id, summary_id}
   - Include IDs for debugging and monitoring

## 6. Dependencies

**External Services:**
- Google Gemini API (requires GEMINI_API_KEY)
- FastAPI Backend (requires BACKEND_API_URL and WHATSAPP_API_KEY)
- Supabase PostgreSQL (data storage)

**Libraries/Tools:**
- **WhatsApp Service**:
  - Node.js 18+
  - @google/generative-ai (Gemini Node.js SDK)
  - axios or node-fetch (HTTP client for backend calls)
  - SQLite3 or fs module (buffer persistence)
- **Backend API**:
  - FastAPI (already set up)
  - Supabase Python client

**Ticket Dependencies:**
- WhatsApp_Service_-_Message_Monitoring_&_Buffer_Persistence (provides message buffering infrastructure)
- FastAPI_Project_Setup_&_Authentication (provides API key authentication middleware)
- Implement_Raw_Data_Tables_(Gmail,_Classroom,_WhatsApp) (provides raw_messages table)
- Implement_Filtered_Data_Tables_(Emails,_Events,_Assignments,_Announcements,_WhatsApp_Summaries) (provides whatsapp_summaries table)

**Blocks:**
- Conversational_Agent_with_Gemini_Integration (needs whatsapp_summaries data)
- Sync_Status_&_Data_Management_Endpoints (may query WhatsApp sync status)

## 7. Execution Model

**WhatsApp Service (Node.js):**

**Type**: Background Service (Long-Running Daemon) with Hourly Cron

**Lifecycle**:
1. **Startup**:
   - Load environment variables
   - Initialize Gemini API client
   - Initialize backend API client (axios with retry)
   - Load unsent_payloads from persistence
   - Start hourly flush timer (check every minute for top of hour)

2. **Runtime (Continuous)**:
   - Message monitoring continues from previous ticket
   - Every minute: check if current time is HH:00
   - At top of hour: trigger summarization process
   - Retry unsent payloads before processing new batches
   - Process new batches (fetch allowlist, select chats, summarize, send)
   - Continue monitoring messages

3. **Shutdown**:
   - Flush all in-memory buffers to disk
   - Persist any unsent payloads
   - Close HTTP connections
   - Log shutdown completion

**Backend API (FastAPI):**

**Type**: HTTP Service (Always Running)

**Lifecycle**:
- POST /ingest/whatsapp endpoint available 24/7
- Handles incoming WhatsApp ingestion requests synchronously
- Returns response immediately (< 1 second)

**Concurrency**:
- WhatsApp service: Single-threaded (Node.js event loop)
- Backend API: Asynchronous (handles multiple concurrent requests)

**Idempotency**:
- WhatsApp service can retry POST safely (backend deduplicates via unique constraint)
- Backend INSERT uses ON CONFLICT DO NOTHING

## 8. Failure Handling

**WhatsApp Service:**

**Gemini API Failures**:
- **Rate Limit or Quota Exceeded**:
  - Retry with exponential backoff (max 2 retries)
  - If still fails: use fallback (summary=null, importance_score=0.3)
  - Log warning (Gemini failed, fallback used)
  - Send payload to backend with fallback values
- **Invalid Response**:
  - Parse error: use fallback values
  - Log parsing error with raw response
  - Continue with fallback
- **Network Timeout**:
  - Retry with increased timeout
  - If fails: use fallback

**Backend API Failures**:
- **Network Timeout**:
  - Retry POST with exponential backoff (1s, 2s, 4s, max 3 retries)
  - If all retries fail: persist unsent payload
  - Log error (backend unreachable)
- **4xx Errors** (Bad Request, Forbidden):
  - Log error with request details
  - Do not retry (configuration issue or invalid data)
  - Persist unsent payload for manual review
- **5xx Errors** (Internal Server Error):
  - Retry POST (transient error)
  - If all retries fail: persist unsent payload
  - Log error
- **Duplicate Batch (409 Conflict)**:
  - Treat as success (idempotency)
  - Delete buffer
  - Log duplicate

**Allowlist Fetch Failures**:
- **Backend Unreachable**:
  - Use cached allowlist from previous fetch (1 hour cache)
  - Log warning (using stale allowlist)
  - Continue summarization
- **Empty Allowlist**:
  - Only summarize private chats (no groups)
  - Log info (no groups allowlisted)

**Buffer Persistence Failures**:
- **Disk Full**:
  - Log critical error
  - Alert (email or log aggregation)
  - Attempt to delete old unsent payloads (>7 days)
  - Continue buffering in memory (risk of data loss on crash)
- **SQLite Locked**:
  - Retry write with backoff
  - If fails: log error, buffer in memory

**Unsent Payload Recovery**:
- On service restart: load unsent_payloads
- Retry sending before processing new batches
- Increment retry_count
- After 10 failed attempts (10 hours): log error, discard payload, alert

**Backend API:**

**Database Errors**:
- **Connection Failure**:
  - Return 500 Internal Server Error
  - Log error with details
  - WhatsApp service will retry
- **Constraint Violation (duplicate)**:
  - Expected for ON CONFLICT clauses
  - Return 200 OK or 409 Conflict (both acceptable)
  - Log duplicate detection
- **Transaction Rollback**:
  - Return 500 Internal Server Error
  - Log rollback reason
  - WhatsApp service will retry

**Validation Errors**:
- **Missing Required Fields**:
  - Return 400 Bad Request with field names
  - Log validation error
  - WhatsApp service should fix and retry
- **Invalid Data Types**:
  - Return 400 Bad Request
  - Log validation error

## 9. Observability

**WhatsApp Service (Node.js):**

**Logging**:
- **Hourly Flush Events**:
  - Flush started (timestamp, chat buffer count)
  - Allowlist fetched (group names, cache status)
  - Chats selected (count, private vs group)
  - Gemini API call (chat_name, latency, status)
  - Backend POST (chat_name, status, retries, duration)
  - Buffer cleared (chat_name)
  - Flush completed (duration, summary statistics)
- **Error Events**:
  - Gemini API failures (chat_name, error, fallback used)
  - Backend API failures (chat_name, error, retry count)
  - Unsent payload persisted (chat_name, retry count)
  - Allowlist fetch failures (using cached allowlist)

**Metrics** (if added):
- whatsapp_flush_count (counter)
- whatsapp_chats_summarized_total (counter, labels: chat_type)
- gemini_api_calls_total (counter, labels: status)
- gemini_api_latency_seconds (histogram)
- backend_post_requests_total (counter, labels: status)
- backend_post_latency_seconds (histogram)
- unsent_payloads_count (gauge)

**Backend API (FastAPI):**

**Logging**:
- **Ingestion Requests**:
  - Request received (chat_name, message count, user_id)
  - Raw message inserted (raw_message_id, duplicate status)
  - Summary inserted (summary_id)
  - Response sent (status, duration)
- **Error Events**:
  - Validation errors (field names, error messages)
  - Database errors (error message, query context)
  - API key validation failures (IP address)

**Metrics** (if Prometheus added):
- whatsapp_ingestion_requests_total (counter, labels: status)
- whatsapp_ingestion_duration_seconds (histogram)
- whatsapp_messages_ingested_total (counter, labels: chat_type)
- whatsapp_duplicates_detected_total (counter)

**Health Check**:
- Include WhatsApp ingestion status in GET /health endpoint
- Show last successful ingestion timestamp

## 10. Security Considerations

**API Key Authentication**:
- WHATSAPP_API_KEY must match between WhatsApp service and backend
- Use cryptographically random key (32+ bytes)
- Never log API key
- Rotate periodically (best practice)

**Gemini API Key Security**:
- GEMINI_API_KEY stored in environment variable
- Never log API key
- Monitor API key usage for anomalies

**Data Privacy**:
- WhatsApp message content is highly sensitive personal data
- Messages transmitted over HTTPS to backend
- Backend stores messages in database (encrypted at rest by Supabase)
- Access restricted to authorized services (API key required)

**Network Security**:
- WhatsApp service → Backend: Use HTTPS in production
- Backend API: Validate API key on every request
- Rate limiting on /ingest/whatsapp endpoint (prevent abuse)

**Allowlist Security**:
- Allowlist determines which group chats are summarized (cost control)
- Only user can modify allowlist (via frontend settings page in future)
- Default: empty allowlist (no groups, only private chats)

**Error Message Sanitization**:
- Never expose message content in error logs (redact)
- Never expose API keys in logs or responses
- Use generic error messages for external monitoring

## 11. Scaling Considerations

**Current MVP Constraints**:
- Single user (no multi-tenancy)
- Modest WhatsApp message volume (~100-500 messages/hour across all chats)
- Hourly summarization (not real-time)
- Single WhatsApp service instance

**Performance Characteristics**:
- Gemini API: ~1-2 seconds per chat summarization
- Backend POST: ~100-200ms per request
- Total hourly flush: ~1-3 minutes for 10 chats

**Vertical Scaling**:
- Increase VPS CPU/memory for faster Gemini API calls
- Use faster network connection (reduce backend POST latency)

**Horizontal Scaling** (future multi-user):
- Run separate WhatsApp service instance per user (each on own VPS)
- Centralized backend API handles ingestion from all instances
- Scale backend horizontally (multiple FastAPI instances behind load balancer)

**Optimization Opportunities**:
- **Parallel Summarization**: Summarize multiple chats in parallel (Promise.all)
- **Batch Backend POSTs**: Send multiple chat payloads in single request (batch endpoint)
- **Reduce Gemini Calls**: Only summarize chats with >5 messages (skip low-activity chats)
- **Caching**: Cache summaries for identical message batches (content hash)

**Bottlenecks**:
- Gemini API calls are slowest operation (~1-2s per chat)
- Sequential processing increases flush time linearly with chat count
- Backend POST network latency

**Cost Management**:
- Gemini API costs scale with chat count and message length
- Monitor token usage (messages concatenated for prompt)
- Allowlist prevents unbounded costs (control which groups summarized)
- Consider summarizing only active chats (>10 messages/hour threshold)

## 12. Future Extensions

**Phase 2 Enhancements**:
- **Real-Time Summarization**:
  - Summarize on-demand (user asks: "Summarize Project Group chat")
  - Eliminate hourly delay for urgent chats
- **Adaptive Summarization Frequency**:
  - Increase frequency for high-activity chats (every 30 minutes)
  - Decrease frequency for low-activity chats (every 4 hours)
- **User-Configurable Allowlist**:
  - Frontend settings page to manage whatsapp_group_allowlist
  - Enable/disable specific group chats for summarization
  - Per-chat importance weighting
- **Multi-User Support**:
  - Run separate WhatsApp service per user
  - Centralized backend handles ingestion from all users
  - Per-user Gemini API quotas

**Advanced Features**:
- **Message Threading**:
  - Detect conversation threads within chat (topic detection)
  - Summarize per thread instead of entire batch
- **Sentiment Analysis**:
  - Detect urgent/angry messages (escalate importance)
  - Track sentiment trends over time
- **Action Item Extraction**:
  - Extract actionable tasks from messages (beyond just deadlines)
  - Link to assignments or events tables
- **Media Summarization**:
  - Transcribe voice messages (speech-to-text)
  - Extract text from images (OCR)
  - Summarize video content
- **Contact Importance**:
  - Weight importance based on sender (important contacts score higher)
  - Learn from user behavior (frequently messaged contacts)

**Monitoring and Alerting**:
- **Proactive Alerts**:
  - Alert on Gemini API quota approaching limit
  - Alert on backend ingestion failures (>3 failed chats)
  - Alert on unsent payload backlog (>10 payloads)
- **Performance Dashboards**:
  - Grafana dashboard for WhatsApp metrics
  - Summarization latency per chat (graph)
  - Gemini API cost tracking
- **Anomaly Detection**:
  - Detect unusual message volumes (spam attacks)
  - Detect summarization quality degradation

**Reliability Improvements**:
- **Circuit Breaker**:
  - Stop calling Gemini API if error rate exceeds threshold
  - Fallback to raw message batch (no summary)
- **Dead Letter Queue**:
  - Move failed payloads to DLQ after 10 retries
  - Manual review and retry mechanism
- **Checkpointing**:
  - Save summarization progress mid-flush (resume on failure)

**Operational Improvements**:
- **Manual Trigger**:
  - API endpoint to trigger summarization on-demand
  - CLI command for one-off flush
- **Configuration Management**:
  - Store summarization frequency in database (per-user settings)
  - Adjust dynamically based on activity patterns
- **Testing Framework**:
  - Unit tests for summarization logic
  - Integration tests with mock Gemini API
  - Fixtures for sample message batches
