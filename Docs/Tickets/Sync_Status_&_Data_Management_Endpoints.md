# Sync Status & Data Management Endpoints

## Objective

Implement endpoints for sync status, on-demand ingestion trigger, and marking announcements as seen.

## Scope

**In Scope:**
- Sync status endpoint: `GET /sync/status` - returns last sync timestamps for all sources (from sync_state table)
- Ingestion trigger endpoint: `POST /ingest/trigger` - queues Celery tasks for on-demand ingestion
- Mark seen endpoint: `PATCH /announcements/{id}/seen` - mark announcement as seen
- Ingestion status endpoint: `GET /ingest/status` - check if ingestion is in progress
- Celery client setup (for queuing tasks)

**Out of Scope:**
- Actual ingestion logic (Celery workers - they operate independently)

## Spec References

- `spec:69889a16-05f6-4f4a-bf1a-8cf0daec03b5/4eb58cf0-bbfc-42e3-acd5-9fe60dd4f30b` (Tech Plan - FastAPI Backend, Data Management section)

## Acceptance Criteria

- [ ] `/sync/status` returns last sync times for Gmail, Classroom, WhatsApp
- [ ] `/ingest/trigger` queues Celery tasks successfully
- [ ] `/announcements/{id}/seen` marks announcement as seen
- [ ] `/ingest/status` shows if ingestion is running
- [ ] All endpoints require authentication
- [ ] Celery client configured

## Dependencies

- Previous ticket (conversational agent)
