# Agent Specification: Google Classroom Ingestion Worker

## 1. Purpose

Implement Celery worker task for Google Classroom data ingestion that fetches assignments and announcements from all enrolled courses, performs AI-based classification and importance scoring, and stores both raw and processed data in Supabase PostgreSQL for the Personal Assistant Agent system.

## 2. Scope

**In Scope:**
- Google Classroom API OAuth 2.0 integration with token management
- Classroom ingestion task (`ingest_classroom_data`):
  - Fetch assignments and announcements using cursor from sync_state (or last 30 days if no cursor)
  - Enumerate all courses user is enrolled in
  - For each course: fetch coursework (assignments) and announcements
  - Save raw data to raw_assignments and raw_classroom_announcements tables
  - For each item: call Gemini API for classification, importance scoring
  - Apply rule-based scoring based on course importance and due date proximity
  - Combine AI score + rule-based score (weighted average)
  - Save filtered/processed data to assignments and announcements tables
  - Update sync_state cursor per course (scope_key per course_id)
- Idempotency using classroom_assignment_id and classroom_announcement_id (deduplicate on insert)
- Error handling and retry logic with exponential backoff
- Scheduled task: runs hourly via Celery Beat
- Logging and monitoring infrastructure

**Out of Scope:**
- Gmail ingestion (already implemented in previous ticket)
- OAuth token refresh task (separate ticket)
- Raw data cleanup task (separate ticket)
- Urgent notifications (separate ticket)
- Submission status tracking (Phase 2 feature)
- Grade tracking (Phase 2 feature)

## 3. Inputs

- **Environment Variables**:
  - DATABASE_URL: Supabase PostgreSQL connection string
  - GEMINI_API_KEY: Google Gemini API key
  - REDIS_URL: Redis connection string
  - CLASSROOM_OAUTH_CLIENT_ID: Google OAuth client ID (same as Gmail)
  - CLASSROOM_OAUTH_CLIENT_SECRET: Google OAuth client secret (same as Gmail)
  - INITIAL_SYNC_DAYS: Days to fetch on first sync (default: 30)
- **Database Tables** (read):
  - users: user_id for data isolation
  - oauth_tokens: Google Classroom OAuth access_token and refresh_token (provider='google_classroom')
  - sync_state: last_synced_at and last_external_id for cursor-based fetching (scope_key per course)
  - user_settings: Potential course importance rules (future enhancement)
- **Google Classroom API**:
  - OAuth 2.0 authenticated access to user's Classroom data
  - courses.list endpoint: Get all enrolled courses
  - courseWork.list endpoint: Get assignments per course
  - announcements.list endpoint: Get announcements per course
  - Query: filter by updateTime > last_synced_at or last 30 days on first sync
- **Gemini API**:
  - Input: Assignment/announcement title, description, course name, due date
  - Output: importance_score (0-1), category (urgent, deadline, general, event), summary (1-2 sentences)

## 4. Outputs

- **Raw Data Tables** (write):
  - raw_assignments: Insert new assignments with classroom_assignment_id, course_name, title, description, due_date, materials, raw_metadata
  - raw_classroom_announcements: Insert new announcements with classroom_announcement_id, course_name, title, text, announced_by, announced_at, raw_metadata
- **Filtered Data Tables** (write):
  - assignments: Insert processed assignments with importance_score, status='pending', is_seen=FALSE, source='google_classroom'
  - announcements: Insert processed announcements with source_type='classroom', source_id (raw_classroom_announcements.id), importance_score, category, is_seen=FALSE
- **Sync State Updates** (write):
  - sync_state: Update per course (source='classroom', scope_key='course:{course_id}')
  - Set last_synced_at to current timestamp
  - Set last_external_id to most recent assignment/announcement ID
- **Task Results**:
  - Celery task result: {courses_processed: 5, assignments_fetched: 15, announcements_fetched: 8, items_processed: 23, errors: 0}
- **Logs**:
  - Task started (timestamp, user_id, courses_count)
  - Courses enumerated (course names, IDs)
  - Assignments fetched per course (count)
  - Announcements fetched per course (count)
  - Gemini API calls (count, latency)
  - Rule-based scoring applied (due date proximity adjustments)
  - Data saved to database (raw counts, filtered counts)
  - Sync state updated per course
  - Task completed (duration, summary)
  - Errors (API failures, database errors)

## 5. Internal Responsibilities

1. **Google Classroom API OAuth Setup**:
   - Reuse OAuth infrastructure from Gmail ingestion ticket
   - Load Google Classroom OAuth tokens from oauth_tokens table (provider='google_classroom')
   - Implement token refresh logic (same pattern as Gmail)
   - Build Classroom API service client
   - Handle OAuth errors (invalid grant, token revoked)

2. **Classroom Ingestion Task Definition**:
   - Define @celery_app.task(name='ingest_classroom_data')
   - Accept user_id parameter
   - Load user's OAuth tokens from database
   - Build Classroom API service client
   - Query sync_state for Classroom cursors (per course)

3. **Enumerate User's Courses**:
   - Call Classroom API courses.list with studentId='me'
   - Filter: courseState=ACTIVE (exclude archived courses)
   - Extract course metadata: id, name, section, descriptionHeading
   - Log course count and names
   - Handle API errors (permission denied, API disabled)

4. **Fetch Assignments Per Course**:
   - For each course: call courseWork.list(courseId=course_id)
   - Filter: updateTime > last_synced_at (from sync_state) or last 30 days on first sync
   - Pagination: process all pages (pageToken)
   - Extract assignment metadata:
     - classroom_assignment_id (courseWork.id)
     - title, description, dueDate, dueTime
     - materials (attachments, links)
     - maxPoints, workType
   - Collect all assignments in batch per course

5. **Fetch Announcements Per Course**:
   - For each course: call announcements.list(courseId=course_id)
   - Filter: updateTime > last_synced_at or last 30 days
   - Pagination: process all pages
   - Extract announcement metadata:
     - classroom_announcement_id (announcement.id)
     - text (announcement body)
     - creatorUserId, creationTime
     - materials (attachments)
   - Collect all announcements in batch per course

6. **Raw Data Insertion**:
   - For each assignment: INSERT into raw_assignments table
     - Use ON CONFLICT (classroom_assignment_id) DO NOTHING for idempotency
     - Store materials as JSONB (links, attachments)
     - Store raw_metadata as JSONB (full API response)
   - For each announcement: INSERT into raw_classroom_announcements table
     - Use ON CONFLICT (classroom_announcement_id) DO NOTHING
     - Store raw_metadata as JSONB
   - Batch insert for performance (100 items at once)
   - Log insertion results (rows inserted, duplicates skipped)

7. **AI Classification with Gemini**:
   - For each new assignment (not duplicate):
     - Build Gemini prompt: "Classify this assignment: Course: {course_name}, Title: {title}, Description: {description}, Due: {due_date}"
     - Request: importance_score (0-1), category (homework, project, exam, quiz, reading), urgency assessment
     - Call Gemini API with temperature=0
     - Parse JSON response
     - Handle API errors (rate limits, invalid responses)
   - For each new announcement (not duplicate):
     - Build Gemini prompt: "Classify this announcement: Course: {course_name}, Text: {text}"
     - Request: importance_score (0-1), category (urgent, deadline, general, event)
     - Call Gemini API
     - Parse response

8. **Rule-Based Importance Scoring**:
   - For assignments:
     - Check due date proximity:
       - Due within 24 hours: add 0.4 to importance_score
       - Due within 3 days: add 0.3
       - Due within 7 days: add 0.2
       - Due within 14 days: add 0.1
     - Check assignment type:
       - workType='ASSIGNMENT' or 'MULTIPLE_CHOICE_QUESTION': base importance
       - workType='SHORT_ANSWER_QUESTION': add 0.1 (quick task)
   - For announcements:
     - Check text for urgency keywords (urgent, important, deadline, exam, quiz): add 0.2
     - Recent announcements (within 24 hours): add 0.1
   - Combine AI score + rule-based adjustments
   - Cap final importance_score at 1.0

9. **Filtered Data Insertion**:
   - For each processed assignment: INSERT into assignments table
     - Include: course_name, title, description, due_date, importance_score, status='pending', is_seen=FALSE, source='google_classroom', processed_at=NOW(), raw_assignment_id (FK)
     - Use ON CONFLICT DO NOTHING
   - For each processed announcement: INSERT into announcements table
     - Include: source_type='classroom', source_id (raw_classroom_announcements.id), title (extracted from text), content=text, announced_by, importance_score, category, is_seen=FALSE, announced_at, processed_at=NOW()
     - Use ON CONFLICT DO NOTHING
   - Batch insert for performance

10. **Sync State Update Per Course**:
    - After processing each course:
      - UPSERT sync_state table (source='classroom', scope_key='course:{course_id}')
      - Set last_synced_at = NOW()
      - Set last_external_id = most recent assignment/announcement ID
    - Use ON CONFLICT (user_id, source, scope_key) DO UPDATE
    - Log sync state updates per course

11. **Error Handling and Retry**:
    - Catch Classroom API errors (rate limit, network failure): retry task
    - Catch Gemini API errors: retry individual item processing, skip if still fails
    - Catch database errors: rollback transaction, retry task
    - Log all errors with context (user_id, course_id, item_id, error_message)
    - Continue processing other courses if one course fails

12. **Task Result and Logging**:
    - Return task result: {courses_processed: int, assignments_fetched: int, announcements_fetched: int, items_processed: int, errors: int, duration_seconds: float}
    - Log task completion with detailed statistics per course
    - Store result in Redis

## 6. Dependencies

**External Services:**
- Google Classroom API (requires OAuth 2.0 authentication)
- Google Gemini API (requires GEMINI_API_KEY)
- Redis (message broker for Celery)
- Supabase PostgreSQL (data storage)

**Libraries/Tools:**
- Python 3.10+
- Celery with Redis broker (already set up in Gmail ingestion ticket)
- google-auth and google-api-python-client for Classroom API
- google-generativeai for Gemini API
- supabase-py for database access

**Ticket Dependencies:**
- Setup_Supabase_Project_&_Core_Tables (provides users, sync_state, oauth_tokens tables)
- Implement_Raw_Data_Tables_(Gmail,_Classroom,_WhatsApp) (provides raw_assignments, raw_classroom_announcements tables)
- Implement_Filtered_Data_Tables_(Emails,_Events,_Assignments,_Announcements,_WhatsApp_Summaries) (provides assignments, announcements tables)
- Celery_Setup_&_Gmail_Ingestion_Worker (provides Celery infrastructure)

**Blocks:**
- Celery_Maintenance_Tasks_(OAuth_Refresh,_Cleanup,_Notifications) (uses Celery infrastructure)
- WhatsApp_Service_-_Hourly_Summarization_&_Backend_Integration (parallel development)

## 7. Execution Model

**Type**: Background Task Worker (Celery Distributed Task Queue)

**Lifecycle**:
1. **Startup**:
   - Celery worker and Beat already running from Gmail ingestion setup
   - Load environment variables
   - Initialize Classroom API client (lazy initialization on first task)
   - Task registered in Celery app

2. **Runtime**:
   - Celery Beat scheduler enqueues ingest_classroom_data task every hour
   - Worker picks up task from queue
   - Execute task logic (enumerate courses, fetch assignments/announcements, process, save)
   - Return task result to Redis
   - Continue listening for next task

3. **Shutdown**:
   - Same as Gmail ingestion worker (graceful shutdown)

**Concurrency**:
- Reuses Celery worker pool (4 processes)
- Task can run in parallel with Gmail ingestion task (independent tasks)

**Scheduled Execution**:
- Celery Beat schedule:
  ```python
  beat_schedule = {
      'ingest-classroom-hourly': {
          'task': 'ingest_classroom_data',
          'schedule': crontab(minute=5),  # Every hour at :05 (offset from Gmail)
          'args': (user_id,)
      }
  }
  ```
- Offset from Gmail task by 5 minutes to avoid concurrent API calls

**Idempotency**: Task is idempotent:
- ON CONFLICT clauses prevent duplicate inserts
- Cursor-based fetching per course avoids re-processing
- Sync state update per course is idempotent (UPSERT)

**Retry Policy**:
- Max retries: 3
- Retry backoff: exponential (1min, 2min, 4min)
- Retry on: Classroom API errors, database connection errors
- Do not retry on: Invalid OAuth token (requires manual intervention)

## 8. Failure Handling

**Classroom API Failures**:
- **Rate Limit (429 error)**:
  - Retry task with exponential backoff
  - Log rate limit event
  - Wait for retry-after header if provided
- **Invalid OAuth Token (401 error)**:
  - Attempt token refresh using refresh_token
  - If refresh fails: log error, alert user (manual re-authentication required)
  - Do not retry task
- **Permission Denied (403 error)**:
  - Log error with course_id
  - Skip course (continue processing other courses)
  - Alert user (course access may have been revoked)
- **Course Not Found (404 error)**:
  - Log warning (course may have been deleted)
  - Skip course
  - Continue processing
- **Network Timeout**:
  - Retry task (max 3 retries)
  - Increase timeout on retry

**Gemini API Failures**:
- **Rate Limit or Quota Exceeded**:
  - Retry individual item processing (max 2 retries)
  - If still fails: skip item (save to raw table but not filtered table)
  - Log skipped items for manual review
- **Invalid Response Format**:
  - Parse error: use default values (category='general', importance_score=0.5)
  - Log parsing error with raw response
  - Continue processing other items
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
  - Log duplicate count per course
  - Continue processing
- **Transaction Rollback**:
  - Retry task (entire transaction)
  - Log rollback reason

**Course Enumeration Failures**:
- **No Courses Returned**:
  - Log warning (user may not be enrolled in any courses)
  - Complete task successfully (nothing to process)
- **API Error on courses.list**:
  - Retry task
  - If persistent: log error and alert

**Assignment/Announcement Parsing Errors**:
- **Malformed Data**:
  - Skip item (do not insert to raw table)
  - Log parsing error with item ID
  - Continue processing other items
- **Missing Required Fields** (title, course name):
  - Use defaults (title='(No Title)', course_name='Unknown Course')
  - Log warning
  - Continue processing

**Task Timeout**:
- **Soft Timeout** (600s): Log warning, attempt graceful completion
- **Hard Timeout** (900s): Kill task, log error, retry task
- Process courses sequentially to avoid long-running tasks
- Implement pagination to process items in batches

**Partial Course Failure**:
- **One Course Fails**:
  - Log error for that course
  - Continue processing other courses
  - Task result includes error count and failed course IDs
  - Retry failed course on next hourly run (cursor not updated)

## 9. Observability

**Logging**:
- **Task Lifecycle**:
  - Task started (timestamp, user_id, task_id)
  - Courses enumerated (count, course names)
  - Processing per course started (course_name, course_id)
  - Processing per course completed (assignments count, announcements count, duration)
  - Task completed (total duration, summary statistics)
  - Task failed (error message, retry count)
- **Classroom API Operations**:
  - courses.list API call (expected count)
  - courseWork.list per course (filter, count, pagination)
  - announcements.list per course (filter, count, pagination)
  - API errors (error code, message, course_id)
  - Token refresh events
- **Gemini API Operations**:
  - Gemini API calls (count, item_type, course_name)
  - API latency (per call, average per course)
  - Classification results (category distribution, importance scores per course)
  - API errors and retries
- **Database Operations**:
  - Raw inserts per table (count, duplicates)
  - Filtered inserts per table (count)
  - Sync state updates per course (new cursor values)
  - Database errors
- **Rule-Based Scoring**:
  - Due date proximity adjustments (count, average adjustment)
  - Urgency keyword matches in announcements (count)
  - Final importance score distribution

**Metrics** (if Prometheus integration added):
- classroom_courses_processed_total (counter, labels: user_id)
- classroom_assignments_fetched_total (counter, labels: user_id, course_name)
- classroom_announcements_fetched_total (counter, labels: user_id, course_name)
- classroom_api_calls_total (counter, labels: endpoint, status)
- classroom_api_latency_seconds (histogram, labels: endpoint)
- gemini_api_calls_total (counter, labels: item_type, status)
- database_inserts_total (counter, labels: table, user_id)

**Task Result** (stored in Redis):
```python
{
  "courses_processed": 5,
  "assignments_fetched": 15,
  "announcements_fetched": 8,
  "assignments_processed": 14,
  "announcements_processed": 8,
  "gemini_calls": 22,
  "gemini_errors": 0,
  "due_date_adjustments": 7,
  "duration_seconds": 38.5,
  "errors": [],
  "course_details": [
    {"course_name": "Math 101", "assignments": 3, "announcements": 2},
    {"course_name": "Physics 201", "assignments": 5, "announcements": 1}
  ]
}
```

**Monitoring Dashboard** (future):
- Assignments processed per course (graph)
- Due date distribution (upcoming deadlines)
- Classroom API usage and costs (graph)
- Error rate trends per course
- Task queue length (backlog)

**Debugging**:
- Enable debug logging (LOG_LEVEL=debug)
- Dump raw assignment/announcement data on parsing errors
- Gemini API request/response logging (if enabled)
- Task execution traces (Celery flower UI)

## 10. Security Considerations

**OAuth Token Security**:
- Same security considerations as Gmail ingestion
- Classroom OAuth tokens stored in oauth_tokens table (provider='google_classroom')
- Never log access_token or refresh_token
- Token refresh updates token in database immediately

**API Key Security**:
- GEMINI_API_KEY stored in environment variable
- Never log GEMINI_API_KEY
- Monitor API key usage for anomalies

**Classroom API Permissions**:
- Request minimal scopes: classroom.courses.readonly, classroom.coursework.me.readonly, classroom.announcements.readonly
- User must consent to OAuth permissions during one-time setup
- Tokens revocable by user in Google account settings

**Data Privacy**:
- Assignment descriptions and announcements may contain sensitive academic information
- Raw data contains full content (PII)
- Access restricted to backend and Celery workers
- User data isolated by user_id foreign key
- Consider encrypting assignment descriptions at rest (future enhancement)

**Rate Limiting**:
- Classroom API has rate limits (similar to Gmail)
- Gemini API token limits
- Monitor API usage to avoid quota exhaustion
- Implement backoff and retry logic for rate limit errors

**Error Message Sanitization**:
- Never expose OAuth tokens or API keys in error logs
- Redact sensitive data from error messages (assignment descriptions, announcement text)
- Use generic error messages for external monitoring

**Course Access Control**:
- Only fetch courses user is enrolled in (studentId='me')
- Do not fetch courses user is teaching (teacherId='me') unless explicitly enabled
- Respect course permissions (archived courses excluded)

## 11. Scaling Considerations

**Current MVP Constraints**:
- Single user (no multi-tenancy)
- Modest course count (~3-10 courses)
- Modest assignment/announcement volume (~50-100 items per course per month)
- Hourly ingestion schedule (not real-time)

**Performance Characteristics**:
- Classroom API: ~100-200ms per API call (courses.list, courseWork.list)
- Gemini API: ~500-1000ms per classification call
- Database insert: ~10ms per batch insert (100 items)
- Total task duration: ~3-8 minutes for 5 courses with 20 items each

**Vertical Scaling**:
- Increase Celery worker concurrency
- Use faster Redis instance
- Increase database connection pool size

**Horizontal Scaling** (future multi-user):
- Add multiple Celery worker instances
- Partition tasks by user_id
- Scale Redis for higher task throughput

**Optimization Opportunities**:
- **Batch Gemini API Calls**: Call Gemini with multiple assignments/announcements at once
- **Parallel Course Processing**: Process courses in parallel using Celery chord or group tasks
- **Caching**: Cache Gemini classification results for duplicate assignments (content hash)
- **Incremental Sync**: Use Classroom history API (if available) for faster incremental sync
- **Database Bulk Inserts**: Use PostgreSQL COPY for faster bulk inserts
- **Reduce API Calls**: Only call Gemini for high-priority assignments (near due date)

**Bottlenecks**:
- Gemini API calls are slowest operation (~1s per item)
- Classroom API rate limits
- Multiple courses increase task duration linearly
- Task queue backlog if ingestion takes longer than 1 hour

**Cost Management**:
- Gemini API costs scale with token usage (monitor costs)
- Classroom API is free up to quota limits
- Database storage grows with raw data (14-day retention policy)

## 12. Future Extensions

**Phase 2 Enhancements**:
- **Real-Time Ingestion**:
  - Use Classroom push notifications (Pub/Sub) for instant updates
  - Eliminate hourly delay
- **Multi-User Support**:
  - Schedule tasks per user (multiple user_id arguments)
  - Partition tasks across worker pool
- **Assignment Submission Tracking**:
  - Fetch submission status (turned in, graded)
  - Update assignments table with submission state
  - Alert on approaching deadlines for unsubmitted work
- **Grade Tracking**:
  - Fetch grades for submitted assignments
  - Store in new grades table
  - Notify on grade posted
- **Course Importance Weighting**:
  - User-defined course importance (weight assignments from important courses higher)
  - Store in user_settings table
  - Apply to importance scoring

**Advanced Features**:
- **Assignment Grouping**:
  - Group related assignments (e.g., multi-part projects)
  - Summarize entire project instead of individual parts
- **Attachment Processing**:
  - Download and parse assignment attachments (PDFs, documents)
  - Extract text for better classification
  - Store attachments in object storage (S3)
- **Deadline Prediction**:
  - ML model to predict missed deadlines based on historical patterns
  - Proactively alert on high-risk assignments
- **Collaboration Detection**:
  - Detect group assignments (multiple students invited)
  - Track collaborators and communication
- **Study Schedule Optimization**:
  - Generate study schedule based on upcoming assignments and exams
  - Prioritize by importance score and due date

**Monitoring and Alerting**:
- **Proactive Alerts**:
  - Alert on task failures (email or Slack)
  - Alert on OAuth token expiration
  - Alert on approaching assignment deadlines
- **Performance Dashboards**:
  - Grafana dashboard for Classroom metrics
  - Assignment ingestion rate per course
  - Gemini API cost tracking
- **Anomaly Detection**:
  - Detect unusual assignment volumes (exam week)
  - Detect classification accuracy degradation

**Reliability Improvements**:
- **Task Checkpointing**:
  - Save progress per course (resume from checkpoint on failure)
  - Avoid re-processing already-classified items
- **Dead Letter Queue**:
  - Move failed tasks to DLQ for manual review
  - Retry DLQ tasks with adjusted parameters
- **Circuit Breaker**:
  - Stop calling Gemini API if error rate exceeds threshold
  - Fallback to rule-based scoring only

**Operational Improvements**:
- **Per-Course Scheduling**:
  - Adjust ingestion frequency per course (active courses more frequent)
  - Skip inactive courses (no recent updates)
- **Configuration Management**:
  - Store course-specific settings in database
  - Enable/disable ingestion per course
- **Testing Framework**:
  - Unit tests for task logic
  - Integration tests with mock Classroom API
  - Fixtures for sample assignments and announcements
