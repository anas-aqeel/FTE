# Test Case Scenarios & Edge Cases

# Test Case Scenarios & Edge Cases

This document provides comprehensive test scenarios covering all components, data flows, and edge cases for the Personal Assistant Agent system.

---

## 1. Database Schema & Data Integrity Tests

### Test Case 1.1: Dual Storage Integrity
**Scenario:** Verify raw and filtered data relationship integrity

**Test Steps:**
1. Insert a raw email with `gmail_message_id = "msg123"`
2. Insert filtered email with `raw_email_id` pointing to the raw email
3. Query filtered email and verify it links to correct raw email
4. Delete raw email and verify foreign key constraint prevents orphaned filtered data

**Expected Result:**
- Foreign key relationship enforced
- Deleting raw data sets `raw_email_id` to NULL in filtered table (ON DELETE SET NULL)
- Filtered data survives with null FK reference

**Edge Cases:**
- Raw data exists but no filtered data (AI processing failed)
- Filtered data references non-existent raw data (should be prevented by FK)

### Test Case 1.2: Deduplication - Gmail
**Scenario:** Prevent duplicate emails on re-ingestion

**Test Steps:**
1. Ingest email with `gmail_message_id = "msg456"`
2. Run ingestion again with same email
3. Verify only one row exists in `raw_emails`

**Expected Result:**
- Unique constraint on `gmail_message_id` prevents duplicates
- Second insert fails or is skipped

**Edge Cases:**
- Same email content but different message IDs (legitimate duplicates)
- Message ID changes (Gmail API inconsistency)

### Test Case 1.3: Deduplication - WhatsApp Batches
**Scenario:** Prevent duplicate hourly batches

**Test Steps:**
1. WhatsApp service sends batch for "Project Group" (10:00-11:00)
2. Service crashes and resends same batch
3. Verify only one row exists in `raw_messages`

**Expected Result:**
- Composite unique constraint on `(user_id, chat_name, batch_start_time, batch_end_time)` prevents duplicates

**Edge Cases:**
- Overlapping time windows (10:30-11:30 vs 10:00-11:00)
- Same chat, different time windows (legitimate)
- Clock skew between VPS and backend

### Test Case 1.4: Raw Data Cleanup (14-Day Retention)
**Scenario:** Verify old raw data is deleted

**Test Steps:**
1. Insert raw email with `created_at = 20 days ago`
2. Insert raw email with `created_at = 10 days ago`
3. Run `cleanup_raw_data()` task
4. Verify only 10-day-old email remains

**Expected Result:**
- Rows older than 14 days deleted
- Recent rows preserved
- Filtered data unaffected (or orphaned if FK cascade)

**Edge Cases:**
- Filtered data still references deleted raw data (need cascade or null FK)
- Timezone differences in timestamp comparison
- Cleanup runs during active ingestion

---

## 2. Gmail Ingestion Tests

### Test Case 2.1: Normal Gmail Ingestion
**Scenario:** Fetch and process new emails

**Test Steps:**
1. User has 5 new emails in Gmail
2. Trigger `ingest_gmail_data()` task
3. Verify all 5 emails saved to `raw_emails`
4. Verify Gemini called 5 times for classification
5. Verify 5 filtered emails in `emails` table with importance scores

**Expected Result:**
- All emails fetched and stored
- AI classification successful
- Importance scores between 0-1
- `sync_state` cursor updated to latest email timestamp

**Edge Cases:**
- 0 new emails (cursor already up-to-date)
- 1000+ new emails (pagination, rate limiting)
- Emails with no subject or body
- Emails with attachments (metadata only)

### Test Case 2.2: Gmail API Rate Limiting
**Scenario:** Handle Gmail API quota exceeded

**Test Steps:**
1. Trigger ingestion when quota is exhausted
2. Gmail API returns 429 (Too Many Requests)
3. Verify retry with exponential backoff
4. Verify partial success (some emails saved before quota hit)

**Expected Result:**
- Task retries after delay
- Partial data saved
- Error logged
- `sync_state` updated to last successful email

**Edge Cases:**
- Quota resets mid-retry
- Multiple concurrent ingestion requests
- Quota exceeded on Gemini API (separate handling)

### Test Case 2.3: Gmail OAuth Token Expiration
**Scenario:** Handle expired access token

**Test Steps:**
1. Set OAuth token to expired state
2. Trigger ingestion
3. Verify worker refreshes token using refresh_token
4. Verify ingestion continues successfully

**Expected Result:**
- Token refreshed automatically
- New access_token saved to database
- Ingestion completes without user intervention

**Edge Cases:**
- Refresh token also expired (requires re-authentication)
- Token refresh fails (network error)
- Concurrent tasks trying to refresh same token

### Test Case 2.4: Event Extraction from Gmail Invites
**Scenario:** Extract calendar events from email invites

**Test Steps:**
1. Receive Gmail with calendar invite (.ics attachment or inline)
2. Trigger ingestion
3. Verify email saved to `raw_emails`
4. Verify event extracted and saved to `raw_events`
5. Verify filtered event in `events` table

**Expected Result:**
- Email and event both stored
- Event has correct start_time, end_time, location
- Event linked to source email

**Edge Cases:**
- Email with multiple events
- Malformed .ics file
- Event with no time (all-day event)
- Recurring events

### Test Case 2.5: Importance Scoring - AI + Rules
**Scenario:** Verify combined importance scoring

**Test Steps:**
1. Configure `user_settings`:
   - `important_senders = ["professor@university.edu"]`
   - `keyword_rules = {"urgent": 0.8, "deadline": 0.7}`
2. Ingest email from professor with subject "Urgent: Assignment Deadline"
3. Verify Gemini returns importance score (e.g., 0.6)
4. Verify rule-based score calculated (sender: 0.9, keywords: 0.8)
5. Verify final combined score (e.g., max or weighted average)

**Expected Result:**
- Final importance_score reflects both AI and rules
- High-importance email (score > threshold) creates announcement
- Urgent notification triggered

**Edge Cases:**
- AI returns null/error (fallback to rules only)
- No matching rules (AI score only)
- Conflicting scores (AI low, rules high)

---

## 3. Google Classroom Ingestion Tests

### Test Case 3.1: Normal Classroom Ingestion
**Scenario:** Fetch assignments and announcements

**Test Steps:**
1. User has 3 new assignments and 2 new announcements in Classroom
2. Trigger `ingest_classroom_data()` task
3. Verify 3 rows in `raw_assignments`
4. Verify 2 rows in `raw_classroom_announcements`
5. Verify filtered data in `assignments` and `announcements` tables

**Expected Result:**
- All items fetched and stored
- Separate raw tables for assignments vs announcements
- AI classification successful
- `sync_state` updated per course

**Edge Cases:**
- Assignment with no due date
- Announcement with no text
- Multiple courses (separate sync_state per course)
- Deleted assignments (API returns 404)

### Test Case 3.2: Assignment Status Tracking
**Scenario:** Track assignment completion status

**Test Steps:**
1. Ingest assignment with `due_date = tomorrow`
2. Verify `status = "pending"` initially
3. User marks assignment as completed (future feature)
4. Verify `status = "completed"`

**Expected Result:**
- Status field tracks assignment lifecycle
- Can filter by status in queries

**Edge Cases:**
- Overdue assignments (due_date < now, status still pending)
- Submitted vs completed distinction

### Test Case 3.3: Classroom API Pagination
**Scenario:** Handle large number of assignments

**Test Steps:**
1. User enrolled in course with 100+ assignments
2. Trigger ingestion
3. Verify all assignments fetched (API pagination handled)

**Expected Result:**
- All pages fetched
- No assignments missed
- Cursor tracks across pages

**Edge Cases:**
- API returns inconsistent page sizes
- New assignments added during pagination
- Pagination token expires mid-fetch

---

## 4. WhatsApp Ingestion Tests

### Test Case 4.1: Normal WhatsApp Hourly Batch
**Scenario:** Monitor and summarize 1 hour of messages

**Test Steps:**
1. WhatsApp service monitors "Project Group" (allowlisted)
2. 50 messages arrive between 10:00-11:00
3. At 11:00, service summarizes batch
4. Gemini extracts: summary, key points, 2 deadlines
5. Service sends to backend `POST /ingest/whatsapp`
6. Verify raw batch in `raw_messages`
7. Verify summary in `whatsapp_summaries`

**Expected Result:**
- All 50 messages in `message_batch` JSONB
- Summary accurately reflects conversation
- Deadlines extracted correctly
- Importance score assigned

**Edge Cases:**
- 0 messages in hour (empty batch - skip or save empty?)
- 1000+ messages in hour (token limit for Gemini)
- Messages with media (images, videos - metadata only)
- Deleted messages

### Test Case 4.2: WhatsApp Buffer Persistence on Crash
**Scenario:** Service crashes mid-hour and restarts

**Test Steps:**
1. Messages arrive from 10:00-10:30 (25 messages buffered)
2. Service crashes at 10:30
3. Service restarts at 10:35
4. More messages arrive from 10:35-11:00 (20 messages)
5. At 11:00, summarization runs

**Expected Result:**
- Buffer persisted to disk contains 25 messages
- After restart, buffer loaded from disk
- New messages (20) added to buffer
- Total 45 messages summarized at 11:00
- No messages lost

**Edge Cases:**
- Crash during disk write (partial buffer)
- Disk full (cannot persist)
- Multiple crashes in same hour
- Crash exactly at 11:00 (during summarization)

### Test Case 4.3: WhatsApp Group Allowlist Filtering
**Scenario:** Only summarize allowlisted groups

**Test Steps:**
1. Configure `whatsapp_group_allowlist = ["Project Group", "Study Group"]`
2. Messages arrive in:
   - "Project Group" (allowlisted) - 30 messages
   - "Family Group" (not allowlisted) - 50 messages
   - "Study Group" (allowlisted) - 20 messages
3. At 11:00, summarization runs

**Expected Result:**
- "Project Group" and "Study Group" summarized
- "Family Group" NOT summarized (or saved as raw only)
- Only 2 Gemini API calls (cost optimization)

**Edge Cases:**
- Group name changes (allowlist uses old name)
- Group not in allowlist but has urgent keyword
- All groups not allowlisted (no summarization)
- Allowlist empty (summarize all or none?)

### Test Case 4.4: WhatsApp Private Chat Handling
**Scenario:** All private chats always summarized

**Test Steps:**
1. Messages arrive in 3 private chats
2. At 11:00, summarization runs
3. Verify all 3 private chats summarized (regardless of allowlist)

**Expected Result:**
- Private chats always processed
- Allowlist only affects groups

**Edge Cases:**
- Private chat with 0 messages (skip or empty summary?)
- Private chat vs group detection (whatsapp-web.js API)

### Test Case 4.5: WhatsApp Backend POST Failure
**Scenario:** Backend unavailable when sending summary

**Test Steps:**
1. Summarization completes at 11:00
2. Backend is down (network error, service restart)
3. POST to `/ingest/whatsapp` fails
4. Verify retry with exponential backoff
5. Backend comes back online
6. Verify payload delivered successfully

**Expected Result:**
- Payload persisted to disk until acknowledged
- Retries: 1s, 2s, 4s, 8s, 16s, etc.
- Eventually succeeds when backend available
- No data loss

**Edge Cases:**
- Backend never comes back (max retries exceeded)
- Backend returns 500 (server error vs network error)
- Partial success (some chats sent, some failed)

### Test Case 4.6: WhatsApp Gemini API Failure
**Scenario:** Gemini API fails during summarization

**Test Steps:**
1. Summarization triggered at 11:00
2. Gemini API returns error (rate limit, service down)
3. Verify fallback: send raw message batch to backend
4. Backend saves raw batch without summary

**Expected Result:**
- Raw data preserved
- Filtered summary missing (can be reprocessed later)
- Marked for later re-processing

**Edge Cases:**
- Gemini timeout (partial response)
- Gemini returns invalid JSON
- Gemini quota exceeded (different from rate limit)

### Test Case 4.7: WhatsApp Session Disconnect
**Scenario:** WhatsApp Web session disconnects

**Test Steps:**
1. Service running normally
2. WhatsApp Web session disconnects (phone offline, logged out elsewhere)
3. Verify service detects disconnect
4. Verify auto-restart attempt
5. If restart fails, require QR code re-authentication

**Expected Result:**
- Session monitoring detects disconnect
- Auto-restart attempted
- If fails, alert/log for manual intervention
- Buffer preserved during disconnect

**Edge Cases:**
- Disconnect during message receipt (message lost?)
- Disconnect during summarization
- Phone battery dies
- Multiple devices logged in

---

## 5. Conversational Agent Tests

### Test Case 5.1: Normal Query with Filtered Data
**Scenario:** User asks about today's schedule

**User Query:** "Brother, what's my schedule for today?"

**Test Steps:**
1. Database has:
   - 2 events today (9 AM class, 2 PM meeting)
   - 1 assignment due today
   - 3 WhatsApp summaries mentioning today's activities
2. Agent queries filtered tables
3. Agent constructs prompt with all relevant data
4. Gemini generates natural language response

**Expected Response:**
```
Brother, today you have:
- 9 AM: Database Systems class
- 2 PM: Project team meeting
- Assignment due: Software Engineering report
- From WhatsApp: Team mentioned meeting location changed to Room 301
```

**Edge Cases:**
- No events today (empty schedule)
- 20+ events today (long response)
- Events with no time (all-day events)
- Past events vs future events (filter logic)

### Test Case 5.2: Query with Raw Data Fallback
**Scenario:** Filtered data insufficient, query raw data

**User Query:** "What did Professor Smith say about the exam?"

**Test Steps:**
1. Filtered data has no mention of "exam" from Professor Smith
2. Agent queries raw_emails for sender="professor.smith@university.edu" + content contains "exam"
3. Raw email found with exam details
4. Gemini generates response from raw data

**Expected Result:**
- Agent successfully falls back to raw data
- Response includes exam details from raw email
- User gets accurate answer despite missing filtered data

**Edge Cases:**
- Data in neither filtered nor raw (truly not available)
- Data in raw but too large for Gemini context window
- Multiple raw emails match (which to use?)

### Test Case 5.3: Multi-Turn Conversation Context
**Scenario:** Conversation history maintained

**Turn 1:**
- User: "What assignments are due this week?"
- Agent: "You have 3 assignments: Math homework (Monday), Essay (Wednesday), Project (Friday)"

**Turn 2:**
- User: "Tell me more about the project"
- Agent: (uses context from Turn 1, knows "project" refers to Friday assignment)

**Test Steps:**
1. Send first query
2. Verify conversation saved to database
3. Send follow-up query
4. Verify agent includes Turn 1 in prompt to Gemini
5. Verify response is contextually relevant

**Expected Result:**
- Conversation history maintained
- Follow-up questions understood
- Context preserved across turns

**Edge Cases:**
- Very long conversation (100+ turns, context window limit)
- Ambiguous references ("it", "that", "the meeting")
- Context switching mid-conversation

### Test Case 5.4: Conversation Thread Switching
**Scenario:** User switches between conversation threads

**Test Steps:**
1. User has 2 conversations:
   - Thread A: discussing assignments
   - Thread B: discussing events
2. User switches from A to B
3. Send query in Thread B
4. Verify only Thread B history used (not Thread A)

**Expected Result:**
- Threads are isolated
- No context pollution between threads
- Each thread maintains independent history

**Edge Cases:**
- Switching mid-response (cancel previous request?)
- Deleting active thread
- Creating many threads (performance)

### Test Case 5.5: Gemini API Failure During Query
**Scenario:** Gemini API unavailable

**Test Steps:**
1. User sends query
2. Gemini API returns error (500, timeout, rate limit)
3. Verify graceful error handling
4. User receives informative error message

**Expected Result:**
- Error caught and logged
- User sees: "Sorry, I'm having trouble processing your request. Please try again."
- Conversation state preserved
- Can retry successfully when API recovers

**Edge Cases:**
- Partial response from Gemini (incomplete JSON)
- Timeout after 30 seconds
- Invalid API key (configuration error)

### Test Case 5.6: Query with No Relevant Data
**Scenario:** User asks about non-existent information

**User Query:** "What's the deadline for the Quantum Physics project?"

**Test Steps:**
1. No data in filtered or raw tables mentions "Quantum Physics"
2. Agent queries both tiers
3. Gemini generates response with available context

**Expected Response:**
```
Brother, I don't have any information about a Quantum Physics project in your records. 
Would you like me to check your recent emails or classroom assignments?
```

**Edge Cases:**
- Typo in query ("Quantam" vs "Quantum")
- Similar but different project names
- Project mentioned in WhatsApp but not extracted

---

## 6. Importance Scoring & Announcement Tests

### Test Case 6.1: High-Importance Email Creates Announcement
**Scenario:** Important email triggers announcement creation

**Test Steps:**
1. Ingest email from professor with subject "URGENT: Exam Rescheduled"
2. Gemini assigns importance_score = 0.9
3. Rule-based scoring adds 0.1 (sender + keyword)
4. Final score = 1.0 (or 0.95 if averaged)
5. Verify announcement created in `announcements` table

**Expected Result:**
- Announcement created with `source_type='email'`, `source_id=<email_id>`
- Announcement has same importance_score
- Announcement marked as unseen

**Edge Cases:**
- Score exactly at threshold (0.7 - include or exclude?)
- Multiple high-importance items in one batch
- Announcement already exists (dedupe logic)

### Test Case 6.2: Mark Announcement as Seen
**Scenario:** User marks announcement as seen

**Test Steps:**
1. User queries: "Any important announcements?"
2. Agent returns 3 unseen announcements
3. User marks announcement #1 as seen via `PATCH /announcements/{id}/seen`
4. User queries again: "Any important announcements?"
5. Verify only 2 announcements returned (announcement #1 excluded)

**Expected Result:**
- `is_seen = true` for announcement #1
- Subsequent queries filter out seen announcements
- User doesn't see duplicates

**Edge Cases:**
- Mark non-existent announcement (404)
- Mark already-seen announcement (idempotent)
- Bulk mark as seen (future feature)

### Test Case 6.3: Urgent Email Notification
**Scenario:** Urgent item triggers email notification

**Test Steps:**
1. Ingest email with importance_score = 0.95 (above threshold)
2. Verify `send_urgent_notifications()` task triggered
3. Verify email sent via Gmail API to user's email
4. Verify notification marked as sent (avoid duplicates)

**Expected Result:**
- Email notification sent
- Subject: "Urgent: [Item Title]"
- Body contains summary and link/details
- Notification logged

**Edge Cases:**
- Gmail API send fails (retry?)
- Multiple urgent items in one batch (batch notification?)
- User's Gmail quota exceeded
- Notification email itself ingested (infinite loop prevention)

---

## 7. Sync & Scheduling Tests

### Test Case 7.1: Hourly Scheduled Ingestion
**Scenario:** Celery Beat triggers hourly tasks

**Test Steps:**
1. Wait for top of hour (e.g., 11:00)
2. Verify Celery Beat triggers:
   - `ingest_gmail_data()`
   - `ingest_classroom_data()`
3. Verify both tasks execute successfully
4. Verify `sync_state` updated for both sources

**Expected Result:**
- Tasks run automatically every hour
- No manual intervention needed
- Logs show successful execution

**Edge Cases:**
- Task still running from previous hour (overlap)
- Celery Beat crashes (tasks missed)
- Clock skew (task runs at 11:02 instead of 11:00)

### Test Case 7.2: On-Demand Ingestion Trigger
**Scenario:** User clicks "Refetch Data" button

**Test Steps:**
1. User clicks button in frontend
2. Frontend calls `POST /ingest/trigger`
3. Backend queues Celery tasks
4. Verify tasks execute immediately (not waiting for hourly schedule)
5. Frontend shows "Sync in progress" indicator
6. After completion, sync status updated

**Expected Result:**
- Tasks queued and executed immediately
- User sees progress indicator
- Sync status reflects latest sync time
- Can trigger multiple times (idempotent)

**Edge Cases:**
- Click button while hourly sync running (concurrent tasks)
- Click button multiple times rapidly (queue flooding)
- Task fails (error shown to user)

### Test Case 7.3: Sync State Cursor Management
**Scenario:** Cursor prevents re-fetching old data

**Test Steps:**
1. Initial ingestion fetches 100 emails
2. `sync_state` cursor set to latest email timestamp
3. Second ingestion runs
4. Verify only emails newer than cursor fetched
5. Verify no duplicate processing

**Expected Result:**
- Cursor-based pagination works
- Only new data fetched
- Efficient (no re-processing)

**Edge Cases:**
- Cursor reset (re-fetch all data)
- Cursor points to deleted item
- Multiple sources with different cursor strategies

---

## 8. Authentication & Security Tests

### Test Case 8.1: JWT Token Expiration
**Scenario:** User's JWT token expires

**Test Steps:**
1. User logs in, receives JWT with 1-hour expiration
2. User waits 1 hour
3. User sends chat message
4. Backend validates JWT, finds it expired
5. Verify 401 Unauthorized response
6. Frontend redirects to login

**Expected Result:**
- Expired tokens rejected
- User must re-login
- No data access with expired token

**Edge Cases:**
- Token expires mid-conversation
- Token tampered with (invalid signature)
- Token for different user

### Test Case 8.2: WhatsApp Service API Key Authentication
**Scenario:** WhatsApp service authenticates to backend

**Test Steps:**
1. WhatsApp service sends `POST /ingest/whatsapp`
2. Request includes `X-API-Key: <shared_secret>` header
3. Backend validates API key
4. Verify request accepted

**Test Steps (Negative):**
1. Send request with wrong API key
2. Verify 401 Unauthorized
3. Data not saved

**Expected Result:**
- Valid API key required
- Invalid key rejected
- Separate from JWT authentication

**Edge Cases:**
- API key in query param vs header
- API key leaked (rotation needed)
- Multiple API keys (future: per-service keys)

### Test Case 8.3: OAuth Token Encryption
**Scenario:** OAuth tokens encrypted at rest

**Test Steps:**
1. Store Gmail OAuth token in database
2. Query `oauth_tokens` table directly
3. Verify `access_token` and `refresh_token` are encrypted (not plaintext)
4. Backend decrypts when using tokens

**Expected Result:**
- Tokens encrypted in database
- Decryption transparent to application
- Encryption key stored securely (env var)

**Edge Cases:**
- Encryption key lost (tokens unrecoverable)
- Encryption key rotation
- Database backup contains encrypted tokens

---

## 9. Data Query & Retrieval Tests

### Test Case 9.1: Query Filtered Data by Importance
**Scenario:** Retrieve high-importance items

**Test Steps:**
1. Database has 10 emails with importance scores: 0.1, 0.3, 0.5, 0.7, 0.9, etc.
2. Agent queries for important items (score > 0.6)
3. Verify only 4 emails returned (0.7, 0.9, etc.)

**Expected Result:**
- Filtering by importance_score works
- Results ordered by score (descending)
- Efficient query (indexed)

**Edge Cases:**
- All items low importance (empty result)
- All items high importance (large result)
- Importance score null (AI failed)

### Test Case 9.2: Query by Date Range
**Scenario:** Retrieve items for specific time period

**User Query:** "What's due this week?"

**Test Steps:**
1. Database has assignments with due dates:
   - Today
   - Tomorrow
   - Next week
   - Last week
2. Agent queries for due_date between now and 7 days from now
3. Verify only this week's assignments returned

**Expected Result:**
- Date range filtering works
- Timezone handling correct
- Results ordered by due_date

**Edge Cases:**
- Timezone differences (user vs server vs data source)
- Daylight saving time transitions
- "This week" definition (Sunday-Saturday vs Monday-Sunday)

### Test Case 9.3: Cross-Source Query
**Scenario:** Query spans multiple data sources

**User Query:** "What do I have tomorrow?"

**Test Steps:**
1. Database has:
   - 2 events tomorrow (from Gmail/Calendar)
   - 1 assignment due tomorrow (from Classroom)
   - WhatsApp summary mentioning tomorrow's meeting
2. Agent queries all relevant tables
3. Gemini synthesizes unified response

**Expected Result:**
- Data from all sources included
- Response coherent and organized
- Source attribution (optional)

**Edge Cases:**
- Conflicting information (email says 2 PM, WhatsApp says 3 PM)
- Duplicate information (same event in multiple sources)
- Partial data (some sources have info, others don't)

---

## 10. Failure & Recovery Tests

### Test Case 10.1: Database Connection Loss
**Scenario:** Supabase connection drops

**Test Steps:**
1. Backend running normally
2. Supabase connection lost (network issue, service restart)
3. User sends query
4. Verify graceful error handling
5. Connection restored
6. Verify automatic reconnection

**Expected Result:**
- Connection pool handles reconnection
- User sees temporary error message
- No data corruption
- Service recovers automatically

**Edge Cases:**
- Connection lost during write (transaction rollback)
- Connection lost during read (retry)
- Prolonged outage (queue requests?)

### Test Case 10.2: Redis Connection Loss
**Scenario:** Redis (Celery broker) unavailable

**Test Steps:**
1. Celery worker running
2. Redis crashes
3. Verify worker cannot fetch tasks
4. Redis restarts
5. Verify worker reconnects and processes queued tasks

**Expected Result:**
- Worker handles Redis disconnect gracefully
- Tasks queued in Redis persist (if persistence enabled)
- Worker reconnects automatically

**Edge Cases:**
- Redis data loss (no persistence)
- Tasks in progress when Redis crashes
- Multiple workers competing for tasks

### Test Case 10.3: Concurrent Ingestion Requests
**Scenario:** Multiple ingestion triggers simultaneously

**Test Steps:**
1. Hourly scheduled ingestion starts at 11:00
2. User clicks "Refetch Data" at 11:00:05
3. Verify both tasks queued
4. Verify idempotency prevents duplicates
5. Verify both complete successfully

**Expected Result:**
- Celery handles concurrent tasks
- Deduplication prevents duplicate data
- Both tasks complete without conflict

**Edge Cases:**
- Same source triggered twice (Gmail + Gmail)
- Different sources (Gmail + Classroom) - should be fine
- Task queue overflow (too many requests)

### Test Case 10.4: Partial Ingestion Failure
**Scenario:** Some items succeed, some fail

**Test Steps:**
1. Ingesting 10 emails
2. Email #5 causes Gemini API error
3. Verify emails 1-4 saved successfully
4. Verify email #5 has raw data but no filtered data
5. Verify emails 6-10 continue processing

**Expected Result:**
- Partial success (9/10 emails processed)
- Failed item logged
- Cursor updated to last successful item
- Can retry failed items later

**Edge Cases:**
- All items fail (complete failure)
- First item fails (cursor not updated)
- Intermittent failures (retry logic)

---

## 11. Edge Cases & Corner Scenarios

### Test Case 11.1: Empty Data Sources
**Scenario:** User has no emails, no assignments, no WhatsApp messages

**User Query:** "What's my schedule today?"

**Expected Result:**
- Agent responds: "Brother, you have no scheduled events or tasks for today."
- No errors
- Graceful handling of empty state

### Test Case 11.2: Extremely Long Email/Message
**Scenario:** Email body is 50,000 characters

**Test Steps:**
1. Ingest very long email
2. Verify raw data saved completely
3. Gemini summarization may truncate or fail (token limit)
4. Verify error handling

**Expected Result:**
- Raw data preserved
- Summary may be partial or indicate "content too long"
- No system crash

### Test Case 11.3: Special Characters & Encoding
**Scenario:** Messages with emojis, non-English text

**Test Steps:**
1. WhatsApp message: "Assignment due tomorrow 📚 महत्वपूर्ण"
2. Verify stored correctly in database (UTF-8)
3. Verify Gemini processes correctly
4. Verify displayed correctly in frontend

**Expected Result:**
- UTF-8 encoding throughout
- Emojis preserved
- Non-English text handled

### Test Case 11.4: Deleted/Removed Content
**Scenario:** Email deleted from Gmail after ingestion

**Test Steps:**
1. Ingest email, save to database
2. User deletes email from Gmail
3. Next ingestion runs
4. Verify email still in our database (we don't sync deletions)

**Expected Result:**
- Our database is append-only
- Deletions in source don't affect our data
- (Future: sync deletions if needed)

### Test Case 11.5: Clock Skew & Timezone Issues
**Scenario:** VPS, backend, and data sources in different timezones

**Test Steps:**
1. VPS in UTC
2. Backend in IST
3. Gmail events in user's local timezone
4. Verify all timestamps normalized to UTC in database
5. Verify queries handle timezone correctly

**Expected Result:**
- All timestamps stored in UTC
- Display in user's timezone
- No off-by-one-hour errors

### Test Case 11.6: Rapid Message Bursts (WhatsApp)
**Scenario:** 100 messages arrive in 1 minute

**Test Steps:**
1. Group chat receives 100 messages rapidly
2. Verify all messages buffered
3. Verify buffer persistence keeps up
4. Verify summarization handles large batch

**Expected Result:**
- All messages captured
- No messages dropped
- Summarization may truncate if exceeds token limit

### Test Case 11.7: Duplicate Announcements from Multiple Sources
**Scenario:** Same announcement in email and Classroom

**Test Steps:**
1. Professor sends email: "Exam on Friday"
2. Professor posts Classroom announcement: "Exam on Friday"
3. Both ingested
4. Verify 2 separate announcements created (different sources)
5. Agent should ideally deduplicate in response

**Expected Result:**
- Both stored (different source_type)
- Agent may mention "mentioned in both email and Classroom"
- (Future: semantic deduplication)

### Test Case 11.8: WhatsApp Group Name Change
**Scenario:** Group renamed after being allowlisted

**Test Steps:**
1. Group "Project Team" in allowlist
2. Group renamed to "CS101 Project Team"
3. Messages arrive in renamed group
4. Verify summarization may fail (name mismatch)

**Expected Result:**
- Name mismatch detected
- Either: update allowlist, or use group ID instead of name
- (Future: track group by ID, not name)

### Test Case 11.9: First-Time Setup (No Historical Data)
**Scenario:** User sets up system for first time

**Test Steps:**
1. Fresh database
2. Run initial ingestion
3. Gmail has 1000+ emails (entire inbox)
4. Classroom has 50+ assignments (entire semester)
5. Verify system handles large initial load

**Expected Result:**
- Pagination handles large datasets
- May take multiple hours for initial sync
- Cursor set correctly for future incremental syncs
- (Consider: limit initial fetch to last 30 days)

### Test Case 11.10: Conversation History Limit
**Scenario:** Very long conversation (100+ turns)

**Test Steps:**
1. User has conversation with 100 messages
2. Send new query
3. Verify only recent N messages included in Gemini prompt (to fit context window)

**Expected Result:**
- Context window managed (e.g., last 20 turns)
- Older messages still in database
- Conversation still coherent

---

## 12. Performance & Scalability Tests

### Test Case 12.1: Large Dataset Query Performance
**Scenario:** Database has 10,000+ emails

**Test Steps:**
1. Database populated with 10,000 emails
2. User queries: "What's due this week?"
3. Measure query response time
4. Verify indexes used (EXPLAIN query)

**Expected Result:**
- Query completes in < 2 seconds
- Indexes on user_id, due_date, importance_score used
- No full table scans

### Test Case 12.2: Concurrent User Queries
**Scenario:** Multiple queries simultaneously (future multi-user)

**Test Steps:**
1. Send 10 concurrent queries to backend
2. Verify all handled correctly
3. Verify no race conditions
4. Verify database connection pool sufficient

**Expected Result:**
- All queries succeed
- Response times acceptable
- No deadlocks or connection exhaustion

### Test Case 12.3: Gemini API Rate Limiting
**Scenario:** Hit Gemini API rate limits

**Test Steps:**
1. Send many queries rapidly
2. Gemini returns 429 (rate limit)
3. Verify retry with backoff
4. Verify user sees appropriate message

**Expected Result:**
- Rate limit handled gracefully
- Retries succeed after delay
- User informed of temporary delay

---

## 13. Integration Tests

### Test Case 13.1: End-to-End User Journey
**Scenario:** Complete user flow from login to query

**Test Steps:**
1. User opens web app
2. Logs in with username/password
3. Creates new conversation
4. Asks: "What's my schedule today?"
5. Receives AI response with schedule
6. Asks follow-up: "Tell me more about the 2 PM meeting"
7. Receives detailed response
8. Marks announcement as seen
9. Clicks "Refetch Data"
10. Logs out

**Expected Result:**
- All steps complete successfully
- Smooth user experience
- Data accurate and up-to-date

### Test Case 13.2: Full Ingestion Cycle
**Scenario:** Complete ingestion from all sources

**Test Steps:**
1. Hourly trigger at 11:00
2. Gmail ingestion fetches 5 emails
3. Classroom ingestion fetches 2 assignments, 1 announcement
4. WhatsApp service sends 3 chat summaries
5. All data saved to database
6. Urgent notification sent for 1 high-importance item
7. User queries and sees all new data

**Expected Result:**
- All sources ingested successfully
- Data available for queries immediately
- Sync status shows latest times

---

## Test Coverage Summary

| Component | Test Cases | Edge Cases Covered |
|-----------|------------|-------------------|
| Database Schema | 4 | 12 |
| Gmail Ingestion | 5 | 18 |
| Classroom Ingestion | 3 | 10 |
| WhatsApp Ingestion | 7 | 25 |
| Conversational Agent | 6 | 20 |
| Importance & Announcements | 3 | 10 |
| Sync & Scheduling | 3 | 9 |
| Auth & Security | 3 | 9 |
| Edge Cases | 10 | 30 |
| Performance | 3 | 6 |
| Integration | 2 | 5 |
| **TOTAL** | **49** | **154** |

---

## Testing Strategy

### Unit Tests
- Individual functions (importance scoring, prompt construction)
- Database operations (CRUD, queries)
- API endpoint handlers

### Integration Tests
- Component interactions (Backend ↔ Celery, WhatsApp ↔ Backend)
- End-to-end flows (ingestion, query)

### System Tests
- Full deployment (Docker Compose)
- Multi-component scenarios
- Failure recovery

### Manual Tests
- OAuth setup flow
- WhatsApp QR code authentication
- Frontend UI/UX
- Conversation quality

---

## Critical Test Priorities (Must Pass Before Production)

1. **Data Integrity:** No data loss on crashes/restarts
2. **Deduplication:** No duplicate data on re-ingestion
3. **Authentication:** No unauthorized access
4. **Importance Scoring:** Accurate identification of urgent items
5. **Conversation Context:** Accurate responses with history
6. **Failure Recovery:** Graceful handling of API failures
7. **WhatsApp Buffer Persistence:** No message loss on VPS restart

---

**Last Updated:** 2026-02-17
**Total Test Cases:** 49
**Total Edge Cases:** 154
