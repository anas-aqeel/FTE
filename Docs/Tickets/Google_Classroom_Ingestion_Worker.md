# Google Classroom Ingestion Worker

## Objective

Implement Google Classroom ingestion worker for assignments and announcements.

## Scope

**In Scope:**
- Google Classroom API OAuth 2.0 integration
- Classroom ingestion task (`ingest_classroom_data`):
  - Fetch assignments and announcements using cursor from sync_state
  - Save to raw_assignments table
  - For each item: call Gemini for classification, importance scoring
  - Apply rule-based scoring
  - Save to assignments and announcements tables
  - Update sync_state cursor
- Idempotency (deduplicate using classroom_assignment_id)
- Error handling and retry logic
- Scheduled task: runs hourly

**Out of Scope:**
- Gmail ingestion
- OAuth token refresh
- Cleanup task

## Spec References

- `spec:69889a16-05f6-4f4a-bf1a-8cf0daec03b5/4eb58cf0-bbfc-42e3-acd5-9fe60dd4f30b` (Tech Plan - Celery Worker, Classroom ingestion)

## Acceptance Criteria

- [ ] Google Classroom API OAuth works
- [ ] Assignments and announcements fetched hourly
- [ ] Data saved to raw_assignments
- [ ] Gemini classifies and scores each item
- [ ] Filtered data saved to assignments and announcements tables
- [ ] sync_state cursor updated
- [ ] Idempotent (no duplicates)
- [ ] Retries on failures

## Dependencies

- Previous ticket (Gmail ingestion worker)