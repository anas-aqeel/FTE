# Agent Specification: Celery Setup & Gmail Ingestion Worker

## 1. Purpose

Establish Celery distributed task queue with Redis broker and implement the Gmail ingestion worker that fetches emails and calendar event invites, performs AI-based classification and importance scoring, applies rule-based filtering, and stores both raw and processed data in Supabase PostgreSQL for the AcademiQ system.

## 2. Scope

**In Scope:**
- Celery + Redis setup and configuration
- Celery Beat configuration for task scheduling
- Gmail API OAuth 2.0 integration with token management
- Gmail ingestion task (`ingest_gmail_data`):
  - Fetch emails using cursor from sync_state table (or last 30 days if no cursor)
  - Save raw emails to raw_emails table
  - Filter out self-notification emails (X-Assistant-Notification header)
  - For each email: call Gemini API for classification, importance scoring, deadline extraction, summary generation
  - Apply rule-based scoring using important_senders and keyword_rules from user_settings
  - Combine AI score + rule-based score (weighted average)
  - Save filtered/processed data to emails table
  - Optionally extract calendar events from Gmail invites → raw_events and events tables
  - Update sync_state cursor (last_synced_at, last_external_id)
- Idempotency using gmail_message_id (deduplicate on insert)
- Error handling and retry logic with exponential backoff
- Scheduled task: runs hourly via Celery Beat
- Logging and monitoring infrastructure

**Out of Scope:**
- Google Classroom ingestion (separate ticket)
- OAuth token refresh task (separate ticket)
- Raw data cleanup task (separate ticket)
- Urgent email notifications (separate ticket)
- FastAPI endpoint implementation (already handled in FastAPI ticket)
- Frontend components

## 3. Inputs

- **Environment Variables**:
  - DATABASE_URL: Supabase PostgreSQL connection string
  - GEMINI_API_KEY: Google Gemini API key
  - REDIS_URL: Redis connection string (default: redis://localhost:6379/0)
  - CELERY_BROKER_URL: Same as REDIS_URL
  - CELERY_RESULT_BACKEND: Same as REDIS_URL
  - GMAIL_OAUTH_CLIENT_ID: Google OAuth client ID
  - GMAIL_OAUTH_CLIENT_SECRET: Google OAuth client secret
  - INITIAL_SYNC_DAYS: Days to fetch on first sync (default: 30)
- **Database Tables** (read):
  - users: user_id for data isolation
  - oauth_tokens: Gmail OAuth access_token and refresh_token
  - sync_state: last_synced_at and last_external_id for cursor-based fetching
  - user_settings: important_senders, keyword_rules for rule-based scoring
- **Gmail API**:
  - OAuth 2.0 authenticated access to user's Gmail account
  - Gmail messages.list and messages.get endpoints
  - Query: newer_than:{last_synced_at} or last 30 days on first sync
- **Gemini API**:
  - Input: Email subject, body, sender
  - Output: category (task, announcement, meeting, personal), importance_score (0-1), extracted_deadline (nullable), summary (1-2 sentences)

## 4. Outputs

- **Raw Data Tables** (write):
  - raw_emails: Insert new emails with gmail_message_id, subject, body, sender, recipients, received_at, labels, raw_metadata
  - raw_events: Insert extracted calendar events from invites (optional)
- **Filtered Data Tables** (write):
  - emails: Insert processed emails with summary, importance_score, category, extracted_deadline, is_seen=FALSE
  - events: Insert extracted events with importance_score, event_type, is_seen=FALSE
- **Sync State Updates** (write):
  - sync_state: Update last_synced_at to current timestamp, last_external_id to most recent gmail_message_id
- **Task Results**:
  - Celery task result stored in Redis: {emails_fetched: 25, emails_processed: 24, errors: 1}
- **Logs**:
  - Task started (timestamp, user_id, cursor)
  - Emails fetched from Gmail (count)
  - Gemini API calls (count, latency)
  - Rule-based scoring applied (sender matches, keyword matches)
  - Emails saved to database (raw count, filtered count)
  - Sync state updated (new cursor)
  - Task completed (duration, summary)
  - Errors (API failures, database errors, parsing errors)

## 5. Internal Responsibilities

1. **Celery Application Setup**:
   - Initialize Celery app with Redis broker
   - Configure task serialization (JSON)
   - Set result backend to Redis
   - Configure task retry policy (max_retries=3, exponential backoff)
   - Configure task time limits (soft=600s, hard=900s)
   - Set up task routing and queues (default queue for MVP)

2. **Celery Beat Scheduler Configuration**:
   - Install and configure Celery Beat for periodic tasks
   - Define schedule: ingest_gmail_data runs every hour (crontab: 0 * * * *)
   - Store schedule in code or database (use code-based schedule for MVP)
   - Ensure only one Beat instance running (single scheduler)

3. **Gmail API OAuth Setup**:
   - Install google-auth and google-api-python-client libraries
   - Implement OAuth token loading from oauth_tokens table
   - Implement token refresh logic (refresh_token → new access_token)
   - Update oauth_tokens table when token refreshed
   - Build Gmail API service client
   - Handle OAuth errors (invalid grant, token revoked)

4. **Gmail Ingestion Task Definition**:
   - Define @celery_app.task(name='ingest_gmail_data')
   - Accept user_id parameter
   - Load user's OAuth tokens from database
   - Query sync_state for Gmail cursor (last_synced_at, last_external_id)
   - If no cursor: fetch emails from last 30 days (INITIAL_SYNC_DAYS)
   - Build Gmail API query (newer_than filter)

5. **Fetch Emails from Gmail API**:
   - Call Gmail messages.list with pagination (maxResults=100)
   - For each message ID: call messages.get with format=full
   - Extract message metadata: subject, body, sender, recipients, received_at, labels
   - Parse HTML email body (convert to plain text if needed)
   - Handle pagination (nextPageToken)
   - Collect all messages in batch

6. **Self-Notification Filtering**:
   - For each email: check headers for X-Assistant-Notification: true
   - If present: skip email (do not insert to raw_emails or process)
   - Log filtered self-notifications (count)

7. **Raw Data Insertion**:
   - For each email (not filtered): INSERT into raw_emails table
   - Use ON CONFLICT (gmail_message_id) DO NOTHING for idempotency
   - Store raw_metadata as JSONB (full Gmail API response)
   - Batch insert for performance (insert 100 emails at once)
   - Log insertion results (rows inserted, duplicates skipped)

8. **AI Classification with Gemini**:
   - For each new email (not duplicate):
     - Build Gemini prompt: "Classify this email: Subject: {subject}, Body: {body}, Sender: {sender}"
     - Request: category, importance_score (0-1), deadline (if mentioned), summary (1-2 sentences)
     - Call Gemini API with temperature=0 for consistent results
     - Parse JSON response
     - Handle API errors (rate limits, invalid responses)
     - Retry failed API calls (exponential backoff)

9. **Rule-Based Importance Scoring**:
   - Load user_settings for user_id: important_senders (list), keyword_rules (list)
   - For each email:
     - Check if sender in important_senders → add 0.3 to importance_score
     - Check if subject or body contains keyword_rules → add 0.2 per keyword (max 0.4)
     - Combine AI score + rule-based adjustments
     - Cap final importance_score at 1.0

10. **Filtered Data Insertion**:
    - For each processed email: INSERT into emails table
    - Include: subject, summary, sender, importance_score, category, extracted_deadline, is_seen=FALSE, source='gmail', received_at, processed_at=NOW(), raw_email_id (FK)
    - Use ON CONFLICT DO NOTHING for idempotency
    - Batch insert for performance

11. **Calendar Event Extraction** (optional):
    - For emails with calendar invite MIME parts (text/calendar):
      - Parse iCalendar data (icalendar library)
      - Extract event: title, description, start_time, end_time, location, attendees
      - INSERT into raw_events table
      - Call Gemini for event importance scoring
      - INSERT into events table with importance_score, event_type, is_seen=FALSE

12. **Sync State Update**:
    - After all emails processed:
      - Update sync_state table for source='gmail', scope_key='global'
      - Set last_synced_at = NOW()
      - Set last_external_id = most recent gmail_message_id
      - Use UPSERT (ON CONFLICT UPDATE) for idempotency

13. **Error Handling and Retry**:
    - Catch Gmail API errors (rate limit, network failure): retry task (max 3 retries)
    - Catch Gemini API errors: retry individual email processing (max 2 retries), skip if still fails
    - Catch database errors: rollback transaction, retry task
    - Log all errors with context (user_id, email_id, error_message)

14. **Task Result and Logging**:
    - Return task result: {emails_fetched: int, emails_processed: int, errors: int, duration_seconds: float}
    - Log task completion with summary statistics
    - Store result in Redis (accessible via Celery result backend)

## 6. Dependencies

**External Services:**
- Gmail API (requires OAuth 2.0 authentication)
- Google Gemini API (requires GEMINI_API_KEY)
- Redis (message broker for Celery)
- Supabase PostgreSQL (data storage)

**Libraries/Tools:**
- Python 3.10+
- Celery with Redis broker
- Celery Beat for task scheduling
- google-auth and google-api-python-client for Gmail API
- google-generativeai for Gemini API
- supabase-py for database access
- icalendar for calendar invite parsing (optional)

**Ticket Dependencies:**
- Setup_Supabase_Project_&_Core_Tables (provides users, sync_state, oauth_tokens, user_settings tables)
- Implement_Raw_Data_Tables_(Gmail,_Classroom,_WhatsApp) (provides raw_emails, raw_events tables)
- Implement_Filtered_Data_Tables_(Emails,_Events,_Assignments,_Announcements,_WhatsApp_Summaries) (provides emails, events tables)

**Blocks:**
- Google_Classroom_Ingestion_Worker (similar pattern, reuses Celery setup)
- Celery_Maintenance_Tasks_(OAuth_Refresh,_Cleanup,_Notifications) (uses Celery infrastructure)

## 7. Execution Model

**Type**: Background Task Worker (Celery Distributed Task Queue)

**Lifecycle**:
1. **Startup**:
   - Load environment variables
   - Initialize Celery app with Redis broker
   - Connect to Redis (test connection)
   - Initialize Supabase client
   - Start Celery worker process (celery -A app.celery worker --loglevel=info)
   - Start Celery Beat scheduler in separate process (celery -A app.celery beat --loglevel=info)

2. **Runtime (Worker)**:
   - Worker listens to task queue in Redis
   - When ingest_gmail_data task enqueued: worker picks up task
   - Execute task logic (fetch, process, save)
   - Return task result to Redis
   - Continue listening for next task

3. **Runtime (Beat Scheduler)**:
   - Beat scheduler runs continuously
   - Every hour at :00 minutes: enqueue ingest_gmail_data task for all active users
   - Store task ID in Redis
   - Monitor task completion (optional)

4. **Shutdown**:
   - Graceful shutdown on SIGTERM/SIGINT
   - Complete in-flight tasks (wait up to 60 seconds)
   - Close database connections
   - Disconnect from Redis

**Concurrency**:
- Worker concurrency: 4 processes (configurable via --concurrency flag)
- Each process handles one task at a time
- Parallel execution for multiple users (future multi-user support)

**Scheduled Execution**:
- Celery Beat schedule:
  ```python
  beat_schedule = {
      'ingest-gmail-hourly': {
          'task': 'ingest_gmail_data',
          'schedule': crontab(minute=0),  # Every hour at :00
          'args': (user_id,)
      }
  }
  ```

**Idempotency**: Task is idempotent (safe to run multiple times):
- ON CONFLICT clauses prevent duplicate inserts
- Cursor-based fetching avoids re-processing old emails
- Sync state update is idempotent (UPSERT)

**Retry Policy**:
- Max retries: 3
- Retry backoff: exponential (1min, 2min, 4min)
- Retry on: Gmail API errors, database connection errors
- Do not retry on: Invalid OAuth token (requires manual intervention)

## 8. Failure Handling

**Gmail API Failures**:
- **Rate Limit (429 error)**:
  - Retry task with exponential backoff (Celery handles automatically)
  - Wait for retry-after header value if provided
  - Log rate limit event
- **Invalid OAuth Token (401 error)**:
  - Attempt token refresh using refresh_token
  - If refresh fails: log error, alert user (manual re-authentication required)
  - Do not retry task (requires user intervention)
- **Network Timeout**:
  - Retry task (max 3 retries)
  - Increase timeout on retry
  - Log timeout errors

**Gemini API Failures**:
- **Rate Limit or Quota Exceeded**:
  - Retry individual email processing (max 2 retries)
  - If still fails: skip email (save to raw_emails but not emails table)
  - Log skipped emails for manual review
- **Invalid Response Format**:
  - Parse error: use default values (category='unknown', importance_score=0.5)
  - Log parsing error with raw response
  - Continue processing other emails
- **API Key Invalid**:
  - Log error and alert
  - Stop task execution (configuration issue)

**Database Errors**:
- **Connection Failure**:
  - Retry task (max 3 retries with backoff)
  - Check DATABASE_URL validity
  - Alert if persistent failure
- **Constraint Violation (duplicate key)**:
  - Expected for ON CONFLICT clauses
  - Log duplicate count
  - Continue processing
- **Transaction Rollback**:
  - Retry task (entire transaction)
  - Log rollback reason
  - Investigate if persistent

**Email Parsing Errors**:
- **Malformed Email**:
  - Skip email (do not insert to raw_emails)
  - Log parsing error with message_id
  - Continue processing other emails
- **Missing Required Fields** (subject, sender, received_at):
  - Use defaults (subject='(No Subject)', sender='unknown')
  - Log warning
  - Continue processing

**Self-Notification Loop Prevention**:
- Check X-Assistant-Notification header on every email
- If present: skip completely (no raw or filtered insert)
- Log filtered notification emails (count)

**Task Timeout**:
- **Soft Timeout** (600s): Log warning, attempt graceful completion
- **Hard Timeout** (900s): Kill task, log error, retry task
- Implement pagination to process emails in batches (avoid long-running tasks)

**Memory Overflow**:
- **Large Email Bodies** (>1MB):
  - Truncate body to 100KB before storage
  - Log truncation warning
  - Store full body in raw_metadata if needed
- **Batch Size Too Large**:
  - Process emails in smaller batches (100 at a time)
  - Release memory between batches

## 9. Observability

**Logging**:
- **Task Lifecycle**:
  - Task started (timestamp, user_id, task_id, cursor)
  - Task completed (duration, summary statistics)
  - Task failed (error message, retry count)
  - Task retried (retry number, backoff delay)
- **Gmail API Operations**:
  - Gmail API query (filter, expected count)
  - Messages fetched (count, IDs)
  - Pagination (page count, total messages)
  - API errors (error code, message)
  - Token refresh events
- **Gemini API Operations**:
  - Gemini API calls (count, email_id)
  - API latency (per call, average)
  - Classification results (category distribution, importance scores)
  - API errors and retries
- **Database Operations**:
  - Raw inserts (table, count, duplicates)
  - Filtered inserts (table, count)
  - Sync state updates (new cursor values)
  - Database errors
- **Rule-Based Scoring**:
  - Important sender matches (count, senders)
  - Keyword matches (count, keywords)
  - Score adjustments (average adjustment)

**Metrics** (if Prometheus integration added):
- celery_task_executions_total (counter, labels: task_name, status)
- celery_task_duration_seconds (histogram, labels: task_name)
- gmail_emails_fetched_total (counter, labels: user_id)
- gmail_emails_processed_total (counter, labels: user_id)
- gemini_api_calls_total (counter, labels: endpoint, status)
- gemini_api_latency_seconds (histogram)
- database_inserts_total (counter, labels: table, user_id)
- task_retry_count (counter, labels: task_name, error_type)

**Task Result** (stored in Redis):
```python
{
  "emails_fetched": 25,
  "emails_processed": 24,
  "emails_skipped": 1,
  "events_extracted": 3,
  "gemini_calls": 24,
  "gemini_errors": 0,
  "rule_matches": {"important_senders": 5, "keywords": 8},
  "duration_seconds": 45.2,
  "errors": []
}
```

**Monitoring Dashboard** (future):
- Real-time task execution status
- Emails processed per hour (graph)
- Gemini API usage and costs (graph)
- Error rate trends
- Task queue length (backlog)

**Debugging**:
- Enable debug logging (LOG_LEVEL=debug)
- Dump raw email data on parsing errors (debug/ directory)
- Gemini API request/response logging (if enabled)
- Task execution traces (Celery flower UI)

## 10. Security Considerations

**OAuth Token Security**:
- OAuth tokens stored in oauth_tokens table with encryption at rest (Supabase)
- Never log access_token or refresh_token values
- Token refresh updates token in database immediately
- Expired tokens refreshed automatically (no manual intervention)
- Consider application-level encryption for tokens (pgcrypto extension)

**API Key Security**:
- GEMINI_API_KEY stored in environment variable (never hardcoded)
- Never log GEMINI_API_KEY
- Rotate API key periodically (best practice)
- Monitor API key usage for anomalies

**Gmail API Permissions**:
- Request minimal scopes: gmail.readonly, gmail.send (for notifications in separate ticket)
- User must consent to OAuth permissions during one-time setup
- Tokens revocable by user in Google account settings

**Data Privacy**:
- Email content is sensitive personal data
- Raw emails contain full message bodies (PII)
- Access restricted to backend and Celery workers (no direct frontend access)
- User data isolated by user_id foreign key
- Consider encrypting email bodies at rest (future enhancement)

**Self-Notification Loop Prevention**:
- Critical security feature: prevents infinite notification loops
- X-Assistant-Notification header must be checked on every email
- Log all filtered self-notifications for audit

**Rate Limiting**:
- Gmail API has rate limits (10,000 requests/day for free tier)
- Gemini API has token limits (based on API tier)
- Monitor API usage to avoid quota exhaustion
- Implement backoff and retry logic for rate limit errors

**Error Message Sanitization**:
- Never expose OAuth tokens or API keys in error logs
- Redact sensitive data from error messages (email bodies, sender emails)
- Use generic error messages for external monitoring

**Celery Task Security**:
- Tasks accept user_id parameter (validate user exists)
- No code execution from task parameters (prevent injection)
- Task results stored in Redis (secure Redis with password)

## 11. Scaling Considerations

**Current MVP Constraints**:
- Single user (no multi-tenancy)
- Modest email volume (~100-500 emails/day)
- Hourly ingestion schedule (not real-time)
- Single Celery worker (no parallelism)

**Performance Characteristics**:
- Gmail API: ~100-200ms per message fetch
- Gemini API: ~500-1000ms per classification call
- Database insert: ~10ms per batch insert (100 emails)
- Total task duration: ~5-10 minutes for 100 emails

**Vertical Scaling**:
- Increase Celery worker concurrency (--concurrency 8)
- Increase worker memory for larger email batches
- Use faster Redis instance (Redis Cluster or AWS ElastiCache)

**Horizontal Scaling** (future multi-user):
- Add multiple Celery worker instances (distribute across machines)
- Partition tasks by user_id (each worker handles subset of users)
- Scale Redis to handle higher task throughput
- Use Celery result backend with better performance (PostgreSQL instead of Redis)

**Optimization Opportunities**:
- **Batch Gemini API Calls**: Call Gemini with multiple emails at once (reduce API overhead)
- **Parallel Processing**: Process emails in parallel using Celery chord or group tasks
- **Caching**: Cache Gemini classification results for duplicate emails (content-based hash)
- **Incremental Sync**: Use Gmail history API for incremental sync (faster than full message list)
- **Database Bulk Inserts**: Use PostgreSQL COPY for faster bulk inserts
- **Reduce API Calls**: Only call Gemini for emails matching certain criteria (skip obvious spam)

**Bottlenecks**:
- Gemini API calls are slowest operation (~1s per email)
- Gmail API rate limits (10,000 requests/day)
- Database inserts with large batches (>1000 emails)
- Task queue backlog if ingestion takes longer than 1 hour

**Cost Management**:
- Gemini API costs scale with token usage (monitor costs)
- Gmail API is free up to quota limits
- Redis memory usage for task results (set TTL on results)
- Database storage grows with raw email data (14-day retention policy)

## 12. Future Extensions

**Phase 2 Enhancements**:
- **Real-Time Ingestion**:
  - Use Gmail push notifications (Pub/Sub) for instant email ingestion
  - Eliminate hourly delay (near real-time updates)
- **Multi-User Support**:
  - Schedule tasks per user (multiple user_id arguments)
  - Partition tasks across worker pool
  - Implement per-user rate limiting
- **Smart Ingestion**:
  - Skip low-priority emails (spam filters)
  - Prioritize high-importance senders (process first)
  - Adaptive scheduling (ingest more frequently during busy times)
- **Advanced Event Extraction**:
  - Direct Google Calendar API integration (bypass Gmail)
  - Fetch all calendar events (not just invites)
  - Sync bidirectional (create/update/delete events)

**Advanced Features**:
- **Email Threading**:
  - Group related emails into threads
  - Summarize entire thread instead of individual emails
  - Track conversation context across emails
- **Attachment Processing**:
  - Download and parse email attachments (PDFs, documents)
  - Extract text from attachments for classification
  - Store attachments in object storage (S3)
- **Spam Detection**:
  - ML-based spam classifier (reduce false positives from Gmail)
  - Auto-archive low-importance emails
- **Email Sentiment Analysis**:
  - Detect urgent/angry emails (high-priority escalation)
  - Track sender sentiment over time
- **Custom Classification Rules**:
  - User-defined regex rules for categories
  - Training data from user feedback (improve Gemini accuracy)

**Monitoring and Alerting**:
- **Proactive Alerts**:
  - Alert on task failures (email or Slack notification)
  - Alert on OAuth token expiration
  - Alert on API quota approaching limit
- **Performance Dashboards**:
  - Grafana dashboard for task metrics
  - Email ingestion rate graphs
  - Gemini API cost tracking
- **Anomaly Detection**:
  - Detect unusual email volumes (spam attacks)
  - Detect classification accuracy degradation

**Reliability Improvements**:
- **Task Checkpointing**:
  - Save progress mid-task (resume from checkpoint on failure)
  - Avoid re-processing already-classified emails
- **Dead Letter Queue**:
  - Move failed tasks to DLQ for manual review
  - Retry DLQ tasks with adjusted parameters
- **Circuit Breaker**:
  - Stop calling Gemini API if error rate exceeds threshold
  - Fallback to rule-based scoring only
- **Data Validation**:
  - Validate Gemini API responses (schema validation)
  - Detect and handle malformed responses

**Operational Improvements**:
- **Task Monitoring UI**:
  - Celery Flower for real-time task monitoring
  - View task history, results, and logs
- **Manual Task Triggering**:
  - API endpoint to trigger ingestion on-demand (already in separate ticket)
  - CLI command for one-off ingestion
- **Configuration Management**:
  - Store task schedules in database (dynamic scheduling)
  - Adjust ingestion frequency per user
- **Testing Framework**:
  - Unit tests for task logic
  - Integration tests with mock Gmail API
  - Fixtures for sample emails
