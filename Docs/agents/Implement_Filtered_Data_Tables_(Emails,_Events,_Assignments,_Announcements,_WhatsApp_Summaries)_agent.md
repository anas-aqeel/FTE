# Agent Specification: Implement Filtered Data Tables (Emails, Events, Assignments, Announcements, WhatsApp Summaries)

## 1. Purpose

Create all filtered/processed data storage tables in Supabase PostgreSQL for storing AI-classified, importance-scored, and labeled data that serves as the primary query target for the conversational agent, enabling fast and relevant responses without repeated AI processing.

## 2. Scope

**In Scope:**
- Implement 5 filtered data tables with complete schemas:
  - `emails` - AI-processed Gmail messages with importance scores and categories
  - `events` - Classified calendar events with importance scores
  - `assignments` - Scored Google Classroom assignments with status tracking
  - `announcements` - Unified announcement table for all sources (email, classroom, whatsapp)
  - `whatsapp_summaries` - AI-generated summaries of WhatsApp chat batches
- Set up foreign key relationships to raw tables with ON DELETE SET NULL (filtered data survives raw cleanup)
- Set up foreign key relationships to users and conversations tables with ON DELETE CASCADE
- Create indexes on user_id, importance_score, is_seen, timestamps for optimized queries
- Implement database migration scripts
- Document schema, relationships, and query patterns

**Out of Scope:**
- Raw data tables (already created in previous ticket)
- Application code for data filtering/processing (handled by Celery workers)
- AI classification logic (Gemini API calls in Celery workers)
- Rule-based scoring logic (application layer)

## 3. Inputs

- **Database Connection**: DATABASE_URL from Supabase
- **Schema Definitions**: Filtered data table structures from Tech Plan Data Model section
- **Migration Framework**: Existing Alembic or Supabase migration setup
- **Existing Tables**:
  - users table (foreign key target)
  - conversations table (foreign key target)
  - raw_emails, raw_events, raw_assignments, raw_classroom_announcements, raw_messages (foreign key targets)

## 4. Outputs

- **5 Filtered Data Tables Created**:

  **emails**:
  ```sql
  id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE
  raw_email_id UUID REFERENCES raw_emails(id) ON DELETE SET NULL
  subject TEXT NOT NULL
  summary TEXT
  sender TEXT NOT NULL
  importance_score FLOAT NOT NULL CHECK (importance_score >= 0 AND importance_score <= 1)
  category TEXT
  extracted_deadline TIMESTAMP
  is_seen BOOLEAN DEFAULT FALSE
  source TEXT DEFAULT 'gmail'
  received_at TIMESTAMP NOT NULL
  processed_at TIMESTAMP DEFAULT NOW()
  ```

  **events**:
  ```sql
  id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE
  raw_event_id UUID REFERENCES raw_events(id) ON DELETE SET NULL
  title TEXT NOT NULL
  description TEXT
  start_time TIMESTAMP NOT NULL
  end_time TIMESTAMP
  location TEXT
  importance_score FLOAT NOT NULL CHECK (importance_score >= 0 AND importance_score <= 1)
  event_type TEXT
  is_seen BOOLEAN DEFAULT FALSE
  source TEXT
  processed_at TIMESTAMP DEFAULT NOW()
  ```

  **assignments**:
  ```sql
  id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE
  raw_assignment_id UUID REFERENCES raw_assignments(id) ON DELETE SET NULL
  course_name TEXT NOT NULL
  title TEXT NOT NULL
  description TEXT
  due_date TIMESTAMP
  importance_score FLOAT NOT NULL CHECK (importance_score >= 0 AND importance_score <= 1)
  status TEXT DEFAULT 'pending'
  is_seen BOOLEAN DEFAULT FALSE
  source TEXT DEFAULT 'google_classroom'
  processed_at TIMESTAMP DEFAULT NOW()
  ```

  **announcements**:
  ```sql
  id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE
  source_type TEXT NOT NULL
  source_id UUID NOT NULL
  title TEXT NOT NULL
  content TEXT NOT NULL
  announced_by TEXT
  importance_score FLOAT NOT NULL CHECK (importance_score >= 0 AND importance_score <= 1)
  category TEXT
  is_seen BOOLEAN DEFAULT FALSE
  announced_at TIMESTAMP NOT NULL
  processed_at TIMESTAMP DEFAULT NOW()
  ```

  **whatsapp_summaries**:
  ```sql
  id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE
  raw_message_id UUID REFERENCES raw_messages(id) ON DELETE SET NULL
  chat_name TEXT NOT NULL
  is_group BOOLEAN NOT NULL
  summary TEXT NOT NULL
  key_points TEXT[]
  mentioned_deadlines JSONB
  importance_score FLOAT NOT NULL CHECK (importance_score >= 0 AND importance_score <= 1)
  is_seen BOOLEAN DEFAULT FALSE
  batch_start_time TIMESTAMP NOT NULL
  batch_end_time TIMESTAMP NOT NULL
  processed_at TIMESTAMP DEFAULT NOW()
  ```

- **Foreign Key Constraints**:
  - All user_id columns → users(id) with ON DELETE CASCADE
  - All raw_*_id columns → raw_* tables with ON DELETE SET NULL (filtered data survives raw cleanup)
  - Announcements have polymorphic reference (source_type + source_id)

- **Database Indexes**:
  - user_id indexes on all 5 tables
  - importance_score indexes (for high-importance queries)
  - is_seen indexes (for filtering unseen items)
  - Timestamp indexes (received_at, start_time, due_date, announced_at, batch_start_time, processed_at)
  - Composite indexes: (user_id, importance_score DESC), (user_id, is_seen, importance_score DESC)

- **Check Constraints**:
  - importance_score CHECK (importance_score >= 0 AND importance_score <= 1)
  - status CHECK (status IN ('pending', 'in_progress', 'completed'))

- **Migration Scripts**:
  - Versioned migration file (e.g., 003_create_filtered_data_tables.sql)
  - Committed to version control
  - Includes rollback logic

## 5. Internal Responsibilities

1. **Migration Script Creation**:
   - Create new migration file using framework
   - Define all 5 filtered data tables with complete schemas
   - Specify data types (UUID, TEXT, FLOAT, BOOLEAN, TIMESTAMP, TEXT[], JSONB)
   - Set up primary keys and default values

2. **Foreign Key Relationships**:
   - **User References** (ON DELETE CASCADE):
     - All user_id columns reference users(id)
     - Deleting user removes all their filtered data
   - **Raw Data References** (ON DELETE SET NULL):
     - raw_email_id → raw_emails(id)
     - raw_event_id → raw_events(id)
     - raw_assignment_id → raw_assignments(id)
     - raw_message_id → raw_messages(id)
     - Filtered data persists when raw data is cleaned up (14-day retention)
   - **Polymorphic Announcements**:
     - source_type TEXT ('email', 'classroom', 'whatsapp')
     - source_id UUID (not enforced by FK due to polymorphism)

3. **Index Creation for Query Performance**:
   - **User Scope**: Indexes on user_id for user-scoped queries
   - **Importance Filtering**: Indexes on importance_score for high-priority queries
   - **Unseen Items**: Indexes on is_seen for filtering unread items
   - **Timestamps**: Indexes on received_at, start_time, due_date, announced_at for date-range queries
   - **Composite Indexes**:
     - (user_id, importance_score DESC) for "show me my most important items"
     - (user_id, is_seen, importance_score DESC) for "unseen important items"
     - (user_id, due_date) for "upcoming deadlines"
     - (user_id, start_time) for "today's events"

4. **Check Constraints**:
   - Importance score must be in range [0, 1]
   - Assignment status must be one of: pending, in_progress, completed
   - Validate at database level for data integrity

5. **Migration Execution**:
   - Run migration against Supabase database
   - Verify all tables created successfully
   - Verify foreign keys, indexes, and check constraints
   - Test ON DELETE SET NULL behavior

6. **Validation and Testing**:
   - Insert sample filtered row into each table
   - Test foreign key relationships (valid user_id, valid raw_*_id)
   - Test ON DELETE SET NULL (delete raw_email, verify email.raw_email_id becomes NULL)
   - Test ON DELETE CASCADE (delete user, verify all filtered data deleted)
   - Test CHECK constraints (importance_score = 1.5 should fail)
   - Verify indexes using EXPLAIN query plans
   - Clean up test data

7. **Documentation**:
   - Document filtered data schemas
   - Explain ON DELETE SET NULL rationale (filtered data persists after raw cleanup)
   - Document query patterns and index choices
   - Create ERD diagram showing filtered tables and relationships

## 6. Dependencies

**Ticket Dependencies:**
- Setup_Supabase_Project_&_Core_Tables (provides users and conversations tables)
- Implement_Raw_Data_Tables_(Gmail,_Classroom,_WhatsApp) (provides raw tables for foreign keys)

**Data Dependencies:**
- users table must exist
- conversations table must exist
- Raw data tables must exist (raw_emails, raw_events, raw_assignments, raw_classroom_announcements, raw_messages)

**Blocks:**
- FastAPI_Project_Setup_&_Authentication (needs database schema complete)
- Celery_Setup_&_Gmail_Ingestion_Worker (needs filtered tables to write processed data)
- All subsequent worker and API tickets

## 7. Execution Model

**Type**: One-Time Database Migration

**Execution Steps**:
1. **Migration Creation** (local development):
   - Generate new migration file
   - Write SQL DDL for all 5 filtered tables
   - Define foreign keys with appropriate cascade rules
   - Define indexes and constraints
   - Write rollback/downgrade SQL

2. **Migration Review**:
   - Verify column types match Tech Plan
   - Check foreign key cascade rules (CASCADE vs SET NULL)
   - Validate index selections (query patterns considered)
   - Review CHECK constraints (importance_score range)

3. **Migration Deployment** (one-time):
   - Run migration against Supabase
   - Verify schema changes applied
   - Test foreign key behavior with sample data
   - Commit migration script

4. **Future Updates** (as needed):
   - Create new migration for schema changes
   - Apply sequentially
   - Never modify existing migrations

**Idempotency**: Migration framework tracks applied migrations

**Rollback**: Migration includes DROP TABLE statements for rollback

**Concurrency**: Single-threaded execution

## 8. Failure Handling

**Migration Execution Failures**:
- **SQL Syntax Errors**: Review and fix SQL before re-running
- **Foreign Key Failures**: Verify parent tables exist (users, conversations, raw_*)
- **Check Constraint Conflicts**: Ensure constraint definitions are valid (importance_score >= 0 AND <= 1)
- **Index Name Conflicts**: Use unique index names across all tables

**Foreign Key Cascade Errors**:
- **ON DELETE SET NULL**: Verify raw_*_id columns are nullable
- **ON DELETE CASCADE**: Ensure cascade behavior is intentional (deleting user removes all data)
- **Circular Dependencies**: Verify no circular FK references exist

**Index Creation Failures**:
- **Disk Space**: Ensure sufficient disk space for index creation
- **Column Type**: Verify indexed columns support indexing (JSONB requires GIN index, not B-tree)
- **Composite Index Order**: Ensure correct column order in composite indexes (most selective first)

**Data Type Mismatches**:
- **Issue**: Application expects different data type than schema
- **Handling**: Fix schema and re-run migration

**Check Constraint Violations**:
- **Issue**: Application code inserts invalid data (importance_score = 1.2)
- **Handling**: This is expected; application must validate before INSERT

## 9. Observability

**Migration Logs**:
- Migration started (timestamp, migration ID)
- Tables created (table names)
- Foreign keys added (source → target, cascade rule)
- Indexes created (index names, columns)
- Check constraints added
- Migration completed or failed (error details)

**Verification Queries**:
```sql
-- List filtered data tables
SELECT table_name FROM information_schema.tables
WHERE table_schema = 'public' AND table_name IN ('emails', 'events', 'assignments', 'announcements', 'whatsapp_summaries');

-- Verify columns
SELECT table_name, column_name, data_type, is_nullable
FROM information_schema.columns
WHERE table_name IN ('emails', 'events', 'assignments', 'announcements', 'whatsapp_summaries')
ORDER BY table_name, ordinal_position;

-- Check foreign keys and cascade rules
SELECT tc.table_name, tc.constraint_name, kcu.column_name, ccu.table_name AS foreign_table,
       rc.delete_rule
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu ON tc.constraint_name = kcu.constraint_name
JOIN information_schema.constraint_column_usage ccu ON tc.constraint_name = ccu.constraint_name
JOIN information_schema.referential_constraints rc ON tc.constraint_name = rc.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY'
  AND tc.table_name IN ('emails', 'events', 'assignments', 'announcements', 'whatsapp_summaries');

-- Check indexes
SELECT tablename, indexname, indexdef
FROM pg_indexes
WHERE tablename IN ('emails', 'events', 'assignments', 'announcements', 'whatsapp_summaries');

-- Verify check constraints
SELECT tc.table_name, tc.constraint_name, cc.check_clause
FROM information_schema.table_constraints tc
JOIN information_schema.check_constraints cc ON tc.constraint_name = cc.constraint_name
WHERE tc.table_name IN ('emails', 'events', 'assignments', 'announcements', 'whatsapp_summaries');
```

**Testing Validation**:
- Insert test filtered row (success expected)
- Insert with invalid importance_score (constraint violation expected)
- Insert with invalid user_id (foreign key violation expected)
- Delete raw_email, verify email.raw_email_id becomes NULL
- Delete user, verify CASCADE delete works
- Query with indexes: EXPLAIN SELECT * FROM emails WHERE user_id = ... AND importance_score > 0.7;

## 10. Security Considerations

**Data Privacy**:
- Filtered tables contain processed but still sensitive data
- Access restricted to backend and Celery workers (no direct frontend access)
- User data isolated by user_id foreign key
- Application layer must enforce user_id filtering on all queries

**Data Retention**:
- Filtered data has longer retention than raw data (indefinite for MVP)
- Raw data deleted after 14 days, but filtered data persists (ON DELETE SET NULL)
- Consider implementing filtered data archival policy in Phase 2 (e.g., 1 year retention)

**Row Level Security (RLS)**:
- Not required for MVP (all access via trusted backend)
- Consider enabling RLS in Phase 2 for defense-in-depth:
  - Policy: user can only SELECT/UPDATE/DELETE their own rows (WHERE user_id = current_user_id())

**Query Injection Prevention**:
- Use parameterized queries in application code
- ORM or prepared statements prevent SQL injection
- Validate importance_score range before INSERT (application layer)

**Encryption**:
- Data encrypted at rest by Supabase (PostgreSQL transparent data encryption)
- No additional application-level encryption for filtered data
- Consider encrypting mentioned_deadlines JSONB if it contains sensitive info

## 11. Scaling Considerations

**Current MVP Constraints**:
- Single user (no multi-tenancy)
- Estimated data volume: ~50-200 filtered emails/day, ~5-20 events/day, ~10-50 assignments/month, ~10-30 WhatsApp summaries/day
- Total filtered data: ~5-20 MB/day
- Indefinite retention: ~150-600 MB/month, ~1.8-7.2 GB/year

**Performance Characteristics**:
- Indexes enable fast queries (O(log n)):
  - High-importance items: WHERE importance_score > 0.7
  - Unseen items: WHERE is_seen = FALSE
  - Upcoming deadlines: WHERE due_date > NOW() ORDER BY due_date
- Composite indexes optimize multi-filter queries:
  - WHERE user_id = ... AND is_seen = FALSE AND importance_score > 0.7

**Query Patterns** (inform future optimization):
1. **Conversational Agent Queries**:
   - "Show me my most important unseen items": (user_id, is_seen, importance_score DESC)
   - "What's my schedule today?": (user_id, start_time) on events table
   - "Upcoming deadlines": (user_id, due_date) on assignments table
   - "Recent announcements": (user_id, announced_at DESC) on announcements table
2. **Frontend Queries**:
   - Announcement list: (user_id, importance_score DESC, announced_at DESC)
   - Mark as seen: UPDATE WHERE id = ... AND user_id = ... SET is_seen = TRUE

**Future Scaling Needs**:
- **Multi-User Support**:
  - Partition tables by user_id if user count exceeds 10,000
  - Maintain separate indexes per partition
- **High Volume Users**:
  - Implement pagination (LIMIT/OFFSET or cursor-based)
  - Add materialized views for common aggregations (daily summary, weekly stats)
  - Consider archival strategy (move old data to archive tables)

**Optimization Opportunities**:
- **Partial Indexes**: Index only high-importance items (WHERE importance_score > 0.5)
- **Covering Indexes**: Include commonly queried columns in index (INCLUDE clause)
- **Materialized Views**: Pre-compute aggregations (daily deadlines, weekly announcements)
- **Query Caching**: Cache frequent queries (e.g., today's schedule) in Redis

## 12. Future Extensions

**Phase 2 Enhancements**:
- **User Feedback Loop**: Add user_rating column (user can rate AI classification accuracy)
- **Classification Confidence**: Add confidence_score column (track AI confidence in classification)
- **Revision History**: Track changes to importance_score over time (audit trail)
- **Soft Deletes**: Add deleted_at column instead of hard deletes
- **Conversation Linking**: Add conversation_id foreign key to link filtered items to conversation context

**Advanced Features**:
- **Full-Text Search**: Add tsvector columns and GIN indexes for fast text search across summaries and content
- **Semantic Search**: Add embedding columns (vector type) for semantic similarity search
- **Tags and Labels**: Add tags TEXT[] column for user-defined categorization
- **Snooze/Remind**: Add snoozed_until TIMESTAMP for deferred importance
- **Priority Scores**: Add multiple score dimensions (urgency_score, relevance_score, effort_score)

**Analytics and Insights**:
- **Trend Analysis**: Track importance_score distribution over time
- **Source Performance**: Compare classification accuracy across sources (email vs classroom vs whatsapp)
- **Deadline Prediction**: ML model to predict missed deadlines based on historical patterns
- **Anomaly Detection**: Flag unusual patterns (sudden spike in important items)

**Multi-Source Enrichment**:
- **Cross-Reference**: Link related items across tables (email about assignment → assignment table)
- **Duplicate Detection**: Identify duplicate announcements across sources
- **Context Aggregation**: Merge related items into unified view (email + calendar invite + announcement)

**Operational Improvements**:
- **Data Quality Monitoring**: Track NULL values, outlier importance_scores, missing summaries
- **Performance Monitoring**: Slow query log, query execution times, index usage stats
- **Capacity Planning**: Monitor table growth rates, predict storage needs
- **Automated Archival**: Move old filtered data (>1 year) to cold storage (S3)
