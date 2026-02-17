# Agent Specification: Sync Status & Data Management Endpoints

## 1. Purpose

Implement FastAPI endpoints for querying sync status across all data sources, triggering on-demand data ingestion, marking announcements as seen, and checking ingestion job status, providing operational visibility and control for the Personal Assistant Agent system.

## 2. Scope

**In Scope:**
- Sync status endpoint: GET /sync/status
  - Returns last sync timestamps for all sources (Gmail, Classroom, WhatsApp) from sync_state table
  - Includes sync status: synced, in_progress, error, never_synced
- Ingestion trigger endpoint: POST /ingest/trigger
  - Queues Celery tasks for on-demand ingestion (all sources or specific source)
  - Returns task IDs for tracking
  - Requires JWT authentication (user-initiated)
- Mark seen endpoint: PATCH /announcements/{id}/seen
  - Marks announcement as seen (is_seen=TRUE) in database
  - Can be extended to other filtered tables (emails, events, assignments, whatsapp_summaries)
- Ingestion status endpoint: GET /ingest/status
  - Checks if ingestion tasks are currently running or queued
  - Returns task status from Celery result backend
  - Shows progress for long-running tasks
- Celery client setup in FastAPI (for queuing tasks and checking status)
- Error handling and input validation
- Authorization (all endpoints require JWT authentication)

**Out of Scope:**
- Actual ingestion logic (Celery workers - already implemented)
- WhatsApp sync status (WhatsApp service tracks separately)
- Detailed task logs (available via Celery flower or log aggregation)
- Task cancellation (Phase 2 feature)

## 3. Inputs

- **HTTP Requests**:
  - GET /sync/status
    - Headers: Authorization: Bearer {jwt_token}
  - POST /ingest/trigger
    - Headers: Authorization: Bearer {jwt_token}
    - Body: {sources?: string[]} (optional: ['gmail', 'classroom'], default: all)
  - PATCH /announcements/{id}/seen
    - Path param: id (announcement UUID)
    - Headers: Authorization: Bearer {jwt_token}
  - GET /ingest/status?task_ids=uuid1,uuid2
    - Query param: task_ids (comma-separated Celery task UUIDs, optional)
    - Headers: Authorization: Bearer {jwt_token}
- **Database Tables** (read):
  - sync_state: source, last_synced_at, user_id (filter by current user)
  - announcements, emails, events, assignments, whatsapp_summaries: is_seen flag
- **Database Tables** (write):
  - announcements (and other filtered tables): UPDATE is_seen=TRUE
- **Celery Backend**:
  - Queue tasks via celery_app.send_task()
  - Check task status via AsyncResult(task_id).state

## 4. Outputs

- **API Responses**:
  - GET /sync/status:
    ```json
    {
      "gmail": {
        "last_synced_at": "2026-02-17T14:00:00Z",
        "status": "synced"
      },
      "classroom": {
        "last_synced_at": "2026-02-17T14:05:00Z",
        "status": "synced"
      },
      "whatsapp": {
        "last_synced_at": "2026-02-17T15:00:00Z",
        "status": "synced"
      }
    }
    ```
  - POST /ingest/trigger:
    ```json
    {
      "success": true,
      "task_ids": {
        "gmail": "uuid1",
        "classroom": "uuid2"
      },
      "message": "Ingestion tasks queued successfully"
    }
    ```
  - PATCH /announcements/{id}/seen:
    ```json
    {
      "success": true,
      "message": "Announcement marked as seen"
    }
    ```
  - GET /ingest/status:
    ```json
    {
      "tasks": [
        {
          "task_id": "uuid1",
          "name": "ingest_gmail_data",
          "status": "SUCCESS",
          "result": {"emails_processed": 25}
        },
        {
          "task_id": "uuid2",
          "name": "ingest_classroom_data",
          "status": "PENDING"
        }
      ]
    }
    ```
- **Celery Task Queuing**:
  - ingest_gmail_data task queued
  - ingest_classroom_data task queued
- **Database Updates**:
  - announcements (or other filtered tables): UPDATE is_seen=TRUE WHERE id={id}
- **Logs**:
  - Sync status queried (user_id)
  - Ingestion triggered (user_id, sources, task_ids)
  - Announcement marked seen (announcement_id, user_id)
  - Task status checked (task_ids)

## 5. Internal Responsibilities

1. **Router Setup**: Create routers/sync.py and routers/data_management.py
2. **Celery Client Initialization**: Import celery_app from worker module, configure for task queuing
3. **GET /sync/status**: Query sync_state table for current user, group by source, format response with status (calculate based on last_synced_at age)
4. **POST /ingest/trigger**: Validate sources parameter, queue Celery tasks (ingest_gmail_data, ingest_classroom_data) with user_id, return task_ids
5. **PATCH /announcements/{id}/seen**: Verify announcement belongs to user, UPDATE is_seen=TRUE, return success
6. **GET /ingest/status**: Parse task_ids, query Celery result backend (AsyncResult), collect task states and results, return formatted response
7. **Sync Status Calculation**: Determine status based on last_synced_at (synced if < 2 hours ago, error if no recent sync, never_synced if no entry)
8. **Authorization**: Use get_current_user dependency, filter all operations by user_id
9. **Error Handling**: Return 404 if announcement not found or doesn't belong to user, 400 if invalid source specified, 500 if Celery errors

## 6. Dependencies

**External Services:**
- Celery with Redis backend (task queuing and status)
- Supabase PostgreSQL (sync_state, announcements tables)

**Libraries/Tools:**
- FastAPI (already set up)
- Celery Python client
- Supabase Python client

**Ticket Dependencies:**
- FastAPI_Project_Setup_&_Authentication (provides auth middleware)
- Celery_Setup_&_Gmail_Ingestion_Worker (provides Celery tasks to trigger)
- Setup_Supabase_Project_&_Core_Tables (provides sync_state table)
- Implement_Filtered_Data_Tables (provides announcements table)

**Blocks:**
- Next.js_Frontend_-_Chat_Interface_&_Conversation_Management (needs sync status and trigger endpoints)

## 7. Execution Model

**Type**: HTTP Service (Synchronous REST API)

**Lifecycle**: Endpoints always available as part of FastAPI app

**Concurrency**: Asynchronous request handling

**Response Time**:
- GET /sync/status: <100ms
- POST /ingest/trigger: <200ms (queuing tasks is fast)
- PATCH /announcements/{id}/seen: <50ms
- GET /ingest/status: <300ms (depends on number of task_ids)

## 8. Failure Handling

**Database Errors:**
- Connection failures: Return 500, log error
- Query errors: Return 500

**Celery Errors:**
- Redis connection failure: Return 500, log error
- Task queuing failure: Return 500, retry once

**Authorization Failures:**
- User doesn't own announcement: Return 403 Forbidden
- Invalid JWT: Return 401 (handled by Auth middleware)

**Validation Errors:**
- Invalid source name: Return 400 Bad Request
- Invalid task_id format: Return 400

## 9. Observability

**Logging:**
- API requests (endpoint, user_id, params)
- Sync status queries (sources returned)
- Ingestion tasks triggered (task_ids, sources)
- Mark seen operations (announcement_id, success)
- Task status checks (task_ids, results)
- Errors (type, context)

**Metrics** (if added):
- sync_status_requests_total (counter)
- ingestion_triggers_total (counter, labels: source)
- mark_seen_operations_total (counter, labels: item_type)

## 10. Security Considerations

**Authorization:**
- Enforce user_id filtering on all operations
- Verify announcement ownership before marking seen
- Only allow users to trigger their own ingestion tasks

**Rate Limiting** (future):
- Limit POST /ingest/trigger frequency (prevent abuse, API quota exhaustion)
- Max 1 trigger per source per 5 minutes

**Input Validation:**
- Validate source names (whitelist: gmail, classroom)
- Validate UUIDs (announcement_id, task_ids)

## 11. Scaling Considerations

**Current MVP:**
- Single user
- Low request frequency
- Simple queries (no heavy aggregations)

**Performance:**
- All endpoints fast (<300ms)
- Database queries optimized with indexes

**Future Scaling:**
- Add caching for sync status (Redis, 1-minute TTL)
- Implement rate limiting on trigger endpoint

## 12. Future Extensions

**Phase 2:**
- Task cancellation (POST /ingest/cancel)
- Detailed task logs (GET /ingest/logs/{task_id})
- Sync history (GET /sync/history)
- Batch mark seen (PATCH /items/seen with multiple IDs)
- Mark all as seen (PATCH /items/seen-all)
- Sync scheduling configuration (POST /sync/schedule)
- Export sync status as CSV/JSON
- WebSocket for real-time sync status updates
