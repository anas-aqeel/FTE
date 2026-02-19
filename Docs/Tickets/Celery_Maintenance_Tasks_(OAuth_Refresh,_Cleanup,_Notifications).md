# Celery Maintenance Tasks (OAuth Refresh, Cleanup, Notifications)

## Objective

Implement Celery maintenance tasks for OAuth token refresh, raw data cleanup, and urgent email notifications.

## Scope

**In Scope:**
- OAuth token refresh task (`refresh_oauth_tokens`):
  - Runs daily
  - Refreshes expiring Gmail/Classroom tokens
  - Updates oauth_tokens table
- Raw data cleanup task (`cleanup_raw_data`):
  - Runs daily
  - Deletes raw data older than 14 days
- Urgent notification task (`send_urgent_notifications`):
  - Runs frequently or inline during ingestion
  - Sends email via Gmail API for items with importance_score > threshold
  - Adds custom header `X-Assistant-Notification: true` to prevent re-ingestion loop
  - Marks notifications as sent

**Out of Scope:**
- Main ingestion tasks

## Spec References

- `spec:69889a16-05f6-4f4a-bf1a-8cf0daec03b5/4eb58cf0-bbfc-42e3-acd5-9fe60dd4f30b` (Tech Plan - Scheduled Tasks, Architectural Decisions 11, 12, 14)

## Acceptance Criteria

- [ ] OAuth tokens refreshed daily before expiration
- [ ] Raw data older than 14 days deleted daily
- [ ] Urgent items trigger email notification
- [ ] Email sent via Gmail API
- [ ] All tasks scheduled correctly

## Dependencies

- Previous ticket (Classroom ingestion)
