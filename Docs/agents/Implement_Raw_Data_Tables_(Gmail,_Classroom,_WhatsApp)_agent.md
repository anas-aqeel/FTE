# Agent Specification: Implement Raw Data Tables (Gmail, Classroom, WhatsApp)

## 1. Purpose

Create all raw data storage tables in Supabase PostgreSQL for storing unprocessed, original data from Gmail, Google Classroom, and WhatsApp sources, serving as the safety net and audit trail for the entire data ingestion pipeline.

## 2. Scope

**In Scope:**
- Implement 5 raw data tables with complete schemas:
  - `raw_emails` - Unprocessed Gmail messages
  - `raw_events` - Calendar events extracted from Gmail or direct Google Calendar API
  - `raw_assignments` - Google Classroom assignments
  - `raw_classroom_announcements` - Google Classroom course announcements
  - `raw_messages` - WhatsApp message batches (hourly aggregations)
- Set up unique constraints on external IDs to prevent duplicates:
  - gmail_message_id (unique)
  - google_event_id (unique)
  - classroom_assignment_id (unique)
  - classroom_announcement_id (unique)
  - Composite unique constraint for WhatsApp: (user_id, chat_name, batch_start_time, batch_end_time)
- Create indexes on user_id, timestamps, and external ID columns for query performance
- Set up foreign key constraints with ON DELETE CASCADE for user_id references
- Create database migration scripts
- Document schema and constraints

**Out of Scope:**
- Filtered/processed data tables (separate ticket)
- Application code for data ingestion (Celery workers - separate tickets)
- Data validation logic (handled by application layer)
- Data retention cleanup logic (handled by Celery maintenance task)

## 3. Inputs

- **Database Connection**: DATABASE_URL from core tables ticket (Supabase connection string)
- **Schema Definitions**: Raw data table structures from Tech Plan Data Model section
- **Migration Framework**: Existing Alembic or Supabase migration setup
- **Core Tables**: Existing users table (foreign key target for user_id)

## 4. Outputs

- **5 Raw Data Tables Created**:

  **raw_emails**:
  ```sql
  id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE
  gmail_message_id TEXT NOT NULL UNIQUE
  subject TEXT
  body TEXT
  sender TEXT
  recipients TEXT[]
  received_at TIMESTAMP NOT NULL
  labels TEXT[]
  raw_metadata JSONB
  created_at TIMESTAMP DEFAULT NOW()
  ```

  **raw_events**:
  ```sql
  id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE
  google_event_id TEXT NOT NULL UNIQUE
  title TEXT
  description TEXT
  start_time TIMESTAMP NOT NULL
  end_time TIMESTAMP
  location TEXT
  attendees TEXT[]
  raw_metadata JSONB
  created_at TIMESTAMP DEFAULT NOW()
  ```

  **raw_assignments**:
  ```sql
  id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE
  classroom_assignment_id TEXT NOT NULL UNIQUE
  course_name TEXT NOT NULL
  title TEXT NOT NULL
  description TEXT
  due_date TIMESTAMP
  materials JSONB
  raw_metadata JSONB
  created_at TIMESTAMP DEFAULT NOW()
  ```

  **raw_classroom_announcements**:
  ```sql
  id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE
  classroom_announcement_id TEXT NOT NULL UNIQUE
  course_name TEXT NOT NULL
  title TEXT
  text TEXT NOT NULL
  announced_by TEXT
  announced_at TIMESTAMP NOT NULL
  raw_metadata JSONB
  created_at TIMESTAMP DEFAULT NOW()
  ```

  **raw_messages**:
  ```sql
  id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE
  chat_name TEXT NOT NULL
  is_group BOOLEAN NOT NULL
  sender TEXT
  message_batch JSONB NOT NULL
  batch_start_time TIMESTAMP NOT NULL
  batch_end_time TIMESTAMP NOT NULL
  created_at TIMESTAMP DEFAULT NOW()
  UNIQUE(user_id, chat_name, batch_start_time, batch_end_time)
  ```

- **Database Indexes**:
  - user_id indexes on all 5 tables
  - Timestamp indexes (received_at, start_time, due_date, announced_at, batch_start_time, created_at)
  - External ID indexes (gmail_message_id, google_event_id, classroom_assignment_id, classroom_announcement_id)
  - Composite index on raw_messages (user_id, chat_name, batch_start_time)

- **Foreign Key Constraints**:
  - All user_id columns reference users(id) with ON DELETE CASCADE
  - Ensures data is deleted when user account is removed

- **Migration Scripts**:
  - Versioned migration file (e.g., 002_create_raw_data_tables.sql or Alembic revision)
  - Committed to version control
  - Includes rollback logic

- **Schema Documentation**:
  - Table descriptions and column definitions
  - Constraint explanations (why composite unique for WhatsApp)
  - Index rationale (query patterns expected)

## 5. Internal Responsibilities

1. **Migration Script Creation**:
   - Create new migration file using migration framework
   - Define all 5 raw data tables with complete column definitions
   - Specify data types (UUID, TEXT, TIMESTAMP, JSONB, BOOLEAN, TEXT[])
   - Set up primary keys (UUID with gen_random_uuid())

2. **Unique Constraints Implementation**:
   - Add UNIQUE constraint on gmail_message_id (prevent duplicate emails)
   - Add UNIQUE constraint on google_event_id (prevent duplicate events)
   - Add UNIQUE constraint on classroom_assignment_id (prevent duplicate assignments)
   - Add UNIQUE constraint on classroom_announcement_id (prevent duplicate announcements)
   - Add composite UNIQUE constraint on raw_messages (user_id, chat_name, batch_start_time, batch_end_time) to prevent duplicate WhatsApp batches

3. **Foreign Key Setup**:
   - Add foreign key constraints on all user_id columns
   - Reference users(id) table created in core tables ticket
   - Set ON DELETE CASCADE behavior (delete raw data when user deleted)
   - Verify referential integrity

4. **Index Creation**:
   - Create indexes on user_id columns for user-scoped queries
   - Create indexes on timestamp columns (received_at, start_time, due_date, announced_at, batch_start_time, created_at)
   - Create indexes on external ID columns for deduplication checks
   - Create composite index on (user_id, chat_name, batch_start_time) for WhatsApp query optimization

5. **Migration Execution**:
   - Run migration against Supabase database
   - Verify all tables created successfully
   - Verify constraints and indexes in place
   - Test constraint enforcement with sample INSERT statements

6. **Validation and Testing**:
   - Insert sample row into each table
   - Test unique constraints (attempt duplicate insert, expect failure)
   - Test foreign key constraints (attempt invalid user_id, expect failure)
   - Test ON DELETE CASCADE (delete test user, verify raw data deleted)
   - Verify indexes created using EXPLAIN query plans
   - Clean up test data

7. **Documentation**:
   - Document table schemas in README or schema documentation file
   - Explain unique constraint rationale (why composite for WhatsApp)
   - Document expected query patterns (inform future optimization)
   - Add migration notes (dependencies, rollback procedure)

## 6. Dependencies

**Ticket Dependencies:**
- Setup_Supabase_Project_&_Core_Tables (MUST complete first - provides users table and DATABASE_URL)

**Data Dependencies:**
- users table must exist (foreign key target for user_id)
- Migration framework must be initialized

**Blocks:**
- Implement_Filtered_Data_Tables (needs raw tables for foreign key references)
- Celery_Setup_&_Gmail_Ingestion_Worker (needs raw_emails and raw_events tables)
- Google_Classroom_Ingestion_Worker (needs raw_assignments and raw_classroom_announcements tables)
- WhatsApp_Service_-_Hourly_Summarization (needs raw_messages table via backend API)

## 7. Execution Model

**Type**: One-Time Database Migration

**Execution Steps**:
1. **Migration Creation** (local development):
   - Generate new migration file using framework (alembic revision or supabase migration new)
   - Write SQL DDL statements for all 5 tables
   - Define constraints and indexes
   - Write rollback/downgrade SQL

2. **Migration Review**:
   - Review SQL syntax for correctness
   - Verify column types match Tech Plan specifications
   - Check constraint definitions
   - Validate index selections

3. **Migration Deployment** (one-time):
   - Run migration against Supabase database
   - Verify schema changes applied (SELECT from information_schema)
   - Test constraints with sample data
   - Commit migration script to Git

4. **Future Updates** (as needed):
   - Create new migration for schema changes (add column, modify constraint)
   - Apply migrations sequentially
   - Never modify existing migration files (create new ones)

**Idempotency**: Migration framework tracks applied migrations; safe to run multiple times

**Rollback**: Migration includes downgrade/down logic (DROP TABLE statements)

**Concurrency**: Single-threaded, sequential execution required

## 8. Failure Handling

**Migration Execution Failures**:
- **SQL Syntax Errors**:
  - Review migration SQL for syntax errors (missing commas, invalid keywords)
  - Test migration locally before applying to production
  - Use PostgreSQL syntax validator
- **Foreign Key Constraint Failures**:
  - Verify users table exists before creating raw tables
  - Check user_id column types match (UUID)
  - Verify CASCADE behavior is correct
- **Unique Constraint Conflicts**:
  - Ensure no existing data violates new unique constraints
  - For WhatsApp composite unique: verify no duplicate batches exist
- **Index Creation Failures**:
  - Check for index name conflicts
  - Verify column names are correct
  - Ensure sufficient disk space for index creation

**Data Type Mismatch**:
- **Issue**: Column types don't match application expectations
- **Handling**: Rollback migration, fix column types, re-apply

**Constraint Violation on Insert**:
- **Issue**: Application code violates unique constraints (tries to insert duplicate)
- **Handling**: This is expected behavior; application must handle conflicts gracefully

**ON DELETE CASCADE Side Effects**:
- **Issue**: Deleting user unexpectedly removes all raw data
- **Handling**: This is intentional; document cascade behavior clearly

**Migration Framework Errors**:
- **Issue**: Alembic/Supabase CLI errors (version conflicts, connection failures)
- **Handling**:
  - Verify DATABASE_URL is correct
  - Check migration framework version
  - Review migration history for inconsistencies
  - Manually rollback if needed (DROP TABLE statements)

## 9. Observability

**Migration Logs**:
- Migration started (timestamp, migration ID)
- Tables created (table names)
- Constraints added (constraint names and types)
- Indexes created (index names and columns)
- Migration completed successfully or failed (error details)

**Verification Queries**:
```sql
-- List all raw data tables
SELECT table_name FROM information_schema.tables
WHERE table_schema = 'public' AND table_name LIKE 'raw_%';

-- Verify columns for each table
SELECT column_name, data_type FROM information_schema.columns
WHERE table_name = 'raw_emails';

-- Check constraints
SELECT constraint_name, constraint_type FROM information_schema.table_constraints
WHERE table_name IN ('raw_emails', 'raw_events', 'raw_assignments', 'raw_classroom_announcements', 'raw_messages');

-- Check indexes
SELECT tablename, indexname, indexdef FROM pg_indexes
WHERE tablename LIKE 'raw_%';

-- Verify foreign keys
SELECT tc.constraint_name, tc.table_name, kcu.column_name, ccu.table_name AS foreign_table_name
FROM information_schema.table_constraints AS tc
JOIN information_schema.key_column_usage AS kcu ON tc.constraint_name = kcu.constraint_name
JOIN information_schema.constraint_column_usage AS ccu ON tc.constraint_name = ccu.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY' AND tc.table_name LIKE 'raw_%';
```

**Testing Validation**:
- Insert test row into each table (success expected)
- Attempt duplicate insert (constraint violation expected)
- Insert with invalid user_id (foreign key violation expected)
- Delete test user, verify CASCADE delete works
- Check row counts: SELECT COUNT(*) FROM raw_emails; (should be 0 initially)

**Schema Documentation**:
- ERD diagram showing raw tables and relationships
- Constraint explanation document (why each constraint exists)
- Index strategy document (query patterns and optimization)

## 10. Security Considerations

**Data Privacy**:
- Raw tables contain sensitive personal data (emails, messages, assignments)
- Access restricted to backend and Celery workers only (no direct frontend access)
- User data isolated by user_id foreign key
- Application layer must enforce user_id filtering on all queries

**Encryption**:
- Data encrypted at rest by Supabase (PostgreSQL transparent data encryption)
- No additional application-level encryption for raw data (processed in memory)
- Consider encrypting raw_metadata JSONB fields if they contain sensitive tokens

**Access Control**:
- Database credentials stored in environment variables (DATABASE_URL)
- No public internet access to database (Supabase manages firewall)
- Row Level Security (RLS) not required for MVP (all access via trusted backend)
- Consider implementing RLS in Phase 2 for defense-in-depth

**Data Retention**:
- Raw data has 14-day retention policy (enforced by Celery cleanup task)
- After 14 days, raw data deleted but filtered data persists
- Deletion tracked in application logs (audit trail)
- Consider implementing soft deletes (deleted_at column) for audit purposes

**Query Injection Prevention**:
- Use parameterized queries in application code (no string concatenation)
- ORM or prepared statements prevent SQL injection
- Validate external IDs before INSERT (ensure they match expected format)

## 11. Scaling Considerations

**Current MVP Constraints**:
- Single user (no multi-tenancy)
- Estimated data volume: ~100-500 emails/day, ~10-50 assignments/month, ~1000 WhatsApp messages/day
- Total raw data: ~10-50 MB/day (before cleanup)
- 14-day retention: ~140-700 MB total raw data storage

**Performance Characteristics**:
- Indexes ensure fast lookups by user_id and timestamps (O(log n))
- Unique constraints enable fast deduplication checks
- JSONB fields allow flexible schema without migrations (but slower queries)
- Composite index on WhatsApp (user_id, chat_name, batch_start_time) optimizes hourly queries

**Future Scaling Needs**:
- **Multi-User Support**:
  - Partition tables by user_id if user count exceeds 10,000
  - Consider separate tables per user (e.g., raw_emails_user123) for extreme isolation
- **High Volume Users**:
  - Add pagination to ingestion queries (LIMIT/OFFSET or cursor-based)
  - Implement batch inserts (bulk INSERT with ON CONFLICT DO NOTHING)
  - Consider table partitioning by timestamp (monthly partitions)

**Optimization Opportunities**:
- **JSONB Indexing**: Create GIN indexes on raw_metadata JSONB fields for faster queries
- **Materialized Views**: Create aggregated views for common queries (e.g., message counts per chat)
- **Compression**: Enable PostgreSQL table compression (TOAST) for large TEXT fields (email body)
- **Archival**: Move old raw data (>30 days) to cold storage (S3, Glacier) if retention extended

**Bottlenecks**:
- **Disk I/O**: High-volume inserts can saturate disk (mitigate with batch inserts, faster SSD)
- **Index Maintenance**: Large tables slow down INSERT due to index updates (acceptable trade-off)
- **JSONB Queries**: Querying nested JSONB is slower than relational columns (use sparingly)

## 12. Future Extensions

**Phase 2 Enhancements**:
- **Soft Deletes**: Add deleted_at column instead of hard deleting rows (retain audit trail)
- **Version History**: Track row modifications (created_at, updated_at, version number)
- **Data Lineage**: Add source_version column to track API version changes over time
- **Advanced Deduplication**: Implement content-based deduplication (hash email body, message content)
- **Schema Validation**: Add CHECK constraints for data quality (e.g., email format, timestamp ranges)

**Advanced Features**:
- **Full-Text Search**: Add tsvector columns and GIN indexes for fast text search across emails and messages
- **Compression**: Compress old raw data (>7 days) to reduce storage costs
- **Partitioning**: Implement table partitioning by timestamp for faster queries and easier archival
- **Replication**: Set up read replicas for analytics queries (avoid impacting production writes)
- **Data Anonymization**: Anonymize raw data after filtering for privacy compliance (GDPR)

**Analytics and Reporting**:
- **Metrics Tables**: Aggregate raw data into metrics tables (daily message counts, email volume trends)
- **Data Warehouse**: Export raw data to data warehouse (BigQuery, Redshift) for long-term analytics
- **ML Training Data**: Use raw data as training corpus for custom ML models (better classification)

**Operational Improvements**:
- **Automated Backups**: Scheduled backups of raw tables before cleanup runs
- **Data Recovery**: Point-in-time recovery mechanism for accidental deletions
- **Schema Migration Testing**: Automated testing of migrations on staging database before production
- **Performance Monitoring**: Track query performance metrics (slow query log, pg_stat_statements)
