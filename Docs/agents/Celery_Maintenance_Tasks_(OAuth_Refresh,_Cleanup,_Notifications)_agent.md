# Agent Specification: Celery Maintenance Tasks (OAuth Refresh, Cleanup, Notifications)

## 1. Purpose

Implement three critical Celery maintenance tasks for the AcademiQ system: OAuth token refresh to prevent authentication failures, raw data cleanup to manage storage costs and privacy compliance, and urgent email notifications to alert users of high-importance items detected during ingestion.

## 2. Scope

**In Scope:**
- OAuth token refresh task (`refresh_oauth_tokens`):
  - Runs daily (scheduled via Celery Beat)
  - Queries oauth_tokens table for tokens expiring within 24 hours
  - Refreshes Gmail and Google Classroom tokens using refresh_token
  - Updates oauth_tokens table with new access_token and expires_at
  - Logs refresh operations and errors
- Raw data cleanup task (`cleanup_raw_data`):
  - Runs daily (scheduled via Celery Beat)
  - Deletes raw data older than 14 days from all raw_* tables
  - Logs deletion counts per table
  - Preserves filtered data (ON DELETE SET NULL foreign key relationship)
- Urgent notification task (`send_urgent_notifications`):
  - Runs frequently (every 15 minutes) or triggered inline during ingestion
  - Queries filtered tables for unseen items with importance_score > threshold (0.8)
  - Sends email notification via Gmail API to user's email address
  - Adds custom header `X-Assistant-Notification: true` to prevent re-ingestion loop
  - Marks items as notified
- Celery Beat schedule configuration for all three tasks
- Error handling and retry logic
- Logging and monitoring

**Out of Scope:**
- Main data ingestion tasks (Gmail, Classroom - already implemented)
- WhatsApp service tasks (separate service)
- Frontend notification display (Phase 2)
- SMS or push notifications (Phase 2)

## 3. Inputs

**OAuth Token Refresh Task:**
- **Database Tables**: oauth_tokens (read access_token, refresh_token, expires_at; write new access_token, expires_at)
- **Environment Variables**: GMAIL_OAUTH_CLIENT_ID, GMAIL_OAUTH_CLIENT_SECRET, DATABASE_URL
- **Google OAuth 2.0 Token Endpoint**: POST https://oauth2.googleapis.com/token

**Raw Data Cleanup Task:**
- **Database Tables**: raw_emails, raw_events, raw_assignments, raw_classroom_announcements, raw_messages
- **Environment Variables**: DATABASE_URL, RAW_DATA_RETENTION_DAYS (default: 14)

**Urgent Notification Task:**
- **Database Tables** (read): emails, events, assignments, announcements, whatsapp_summaries
- **Database Tables** (write): Update notification_sent flag
- **Environment Variables**: DATABASE_URL, URGENT_THRESHOLD (default: 0.8), USER_EMAIL_ADDRESS
- **Gmail API**: OAuth 2.0 authenticated access, messages.send endpoint

## 4. Outputs

**OAuth Token Refresh Task:**
- Database Updates: oauth_tokens table (UPDATE access_token, expires_at)
- Logs: Task started, tokens refreshed, failures, task completed
- Task Result: {tokens_refreshed: int, errors: int}

**Raw Data Cleanup Task:**
- Database Deletes: DELETE rows older than 14 days from all raw_* tables
- Logs: Task started, rows deleted per table, task completed
- Task Result: {raw_emails_deleted: int, raw_events_deleted: int, ...}

**Urgent Notification Task:**
- Email Sent via Gmail API (with X-Assistant-Notification header)
- Database Updates: Set notification_sent=TRUE
- Logs: Task started, emails sent, failures, task completed
- Task Result: {notifications_sent: int, errors: int}

## 5. Internal Responsibilities

**OAuth Token Refresh Task:**
1. Query oauth_tokens table for tokens expiring within 24 hours
2. For each token: call Google OAuth token endpoint with refresh_token
3. Parse response and UPDATE oauth_tokens with new access_token
4. Handle errors and log results

**Raw Data Cleanup Task:**
1. Calculate cutoff timestamp: NOW() - INTERVAL '{RAW_DATA_RETENTION_DAYS} days'
2. DELETE FROM each raw_* table WHERE created_at < cutoff
3. Log deletion counts per table

**Urgent Notification Task:**
1. Query filtered tables for urgent unseen items (importance_score > 0.8)
2. Build and send email for each item via Gmail API
3. Add X-Assistant-Notification header
4. Update notification_sent flag

## 6. Dependencies

**External Services:**
- Google OAuth 2.0 token endpoint
- Gmail API (for notifications)
- Supabase PostgreSQL

**Libraries/Tools:**
- Celery with Redis (already set up)
- google-auth and google-api-python-client
- supabase-py

**Ticket Dependencies:**
- Celery_Setup_&_Gmail_Ingestion_Worker (provides Celery infrastructure)
- All database tickets

**Blocks:**
- None (maintenance tasks run independently)

## 7. Execution Model

**Type**: Scheduled Background Tasks (Celery Beat)

**Scheduled Execution:**
- refresh_oauth_tokens: Daily at 2:00 AM
- cleanup_raw_data: Daily at 3:00 AM
- send_urgent_notifications: Every 15 minutes

**Concurrency**: Sequential execution per task type

**Retry Policy**: Max 3 retries with exponential backoff

**Idempotency**: All tasks safe to run multiple times

## 8. Failure Handling

**OAuth Refresh Failures:**
- Invalid refresh_token: Alert user (manual re-authentication required)
- Network errors: Retry task

**Cleanup Failures:**
- Database errors: Retry task

**Notification Failures:**
- Gmail API errors: Retry individual email
- Rate limit: Wait and retry on next run

## 9. Observability

**Logging:**
- Task lifecycle events (start, complete, errors)
- Token refreshes (success/failure per provider)
- Cleanup deletion counts
- Notification emails sent

**Metrics** (if added):
- oauth_tokens_refreshed_total
- raw_data_rows_deleted_total
- urgent_notifications_sent_total

## 10. Security Considerations

**OAuth Token Security:**
- Refresh tokens never logged
- New access tokens encrypted at rest

**Email Notification Security:**
- X-Assistant-Notification header prevents re-ingestion loop (critical)
- Only send to user's own email

**Data Privacy:**
- 14-day raw data retention reduces privacy risk

## 11. Scaling Considerations

**Current MVP:**
- Single user
- Low task frequency

**Future Scaling:**
- Multi-user: Schedule tasks per user
- Higher notification frequency for real-time alerts

## 12. Future Extensions

**Phase 2:**
- Push notifications (browser, mobile)
- SMS notifications (Twilio)
- Configurable notification preferences
- Digest emails (daily summary)
- Adaptive cleanup retention
- Proactive token refresh (7 days before expiration)

**Advanced Features:**
- Multiple notification channels
- Per-item-type preferences
- Snooze notifications
- Notification history tracking
- OAuth health monitoring dashboard
