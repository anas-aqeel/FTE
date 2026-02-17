# Celery Setup & Gmail Ingestion Worker

## Objective

Set up Celery with Redis and implement Gmail ingestion worker (emails + optional event extraction from invites).

## Scope

**In Scope:**
- Celery + Redis setup
- Celery Beat configuration for scheduling
- Gmail API OAuth 2.0 integration
- Gmail ingestion task (`ingest_gmail_data`):
  - Fetch emails using cursor from sync_state
  - Save to raw_emails table
  - For each email: call Gemini for classification, importance scoring, deadline extraction
  - Apply rule-based scoring (sender importance, keywords from user_settings)
  - Combine AI + rule scores
  - Save to emails table
  - Optionally extract events from invites → raw_events
  - Update sync_state cursor
- Idempotency (deduplicate using gmail_message_id)
- Error handling and retry logic
- Scheduled task: runs hourly

**Out of Scope:**
- Google Classroom ingestion
- OAuth token refresh task
- Cleanup task
- Urgent notifications

## Spec References

- `spec:69889a16-05f6-4f4a-bf1a-8cf0daec03b5/4eb58cf0-bbfc-42e3-acd5-9fe60dd4f30b` (Tech Plan - Celery Worker, Gmail ingestion)

## Acceptance Criteria

- [ ] Celery worker runs successfully
- [ ] Celery Beat schedules hourly Gmail ingestion
- [ ] Gmail API OAuth works
- [ ] Emails are fetched (last 30 days on first sync)
- [ ] Self-notification emails filtered out (X-Assistant-Notification header)
- [ ] Emails saved to raw_emails
- [ ] Gemini classifies and scores each email
- [ ] Rule-based scoring applied
- [ ] Filtered data saved to emails table
- [ ] Events extracted from invites (if present)
- [ ] sync_state cursor updated
- [ ] Idempotent (no duplicates on re-run)
- [ ] Retries on API failures

## Dependencies

- Database schema complete (all 3 database tickets)
