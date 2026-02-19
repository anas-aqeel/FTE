# Agent Specification: Setup Supabase Project & Core Tables

## 1. Purpose

Create and configure the Supabase PostgreSQL database project with all foundational core tables (users, conversations, conversation_messages, sync_state, user_settings, oauth_tokens) that serve as the data backbone for the entire AcademiQ system.

## 2. Scope

**In Scope:**
- Create new Supabase project with PostgreSQL database
- Configure database connection settings and security rules
- Implement 6 core tables with complete schemas:
  - `users` - User accounts synced from Clerk Auth
  - `conversations` - Chat conversation threads
  - `conversation_messages` - Individual messages within conversations
  - `sync_state` - Data ingestion cursors and timestamps for all sources
  - `user_settings` - User preferences including importance rules and allowlists
  - `oauth_tokens` - OAuth 2.0 access and refresh tokens for Google API access (Gmail, Calendar, Classroom)
- Set up all foreign key constraints with appropriate cascade rules
- Create database indexes on user_id and timestamp columns for query performance
- Establish migration framework (Alembic for Python or Supabase migrations)
- Generate database connection string for application use
- Commit all migration scripts to version control

**Out of Scope:**
- Data source tables (raw_emails, raw_events, emails, events, etc.) - handled by subsequent tickets
- Application code implementation
- Data ingestion logic
- Frontend components
- API endpoint implementation

## 3. Inputs

- **Supabase Account Credentials**: Login credentials for Supabase dashboard
- **Project Configuration**:
  - Project name: "AcademiQ" (or user-specified)
  - Database password
  - Region selection
- **Schema Definitions**: Table structures defined in Tech Plan Data Model section
- **Migration Framework Configuration**:
  - Alembic initialization (if using Python migrations)
  - Supabase migration tool configuration (if using built-in migrations)

## 4. Outputs

- **Supabase Project**: Fully configured and accessible PostgreSQL database instance
- **Database Tables**: 6 core tables created with schemas:
  - users (id, clerk_id, email, username, created_at, updated_at)
  - conversations (id, user_id, title, created_at, updated_at)
  - conversation_messages (id, conversation_id, role, content, created_at)
  - sync_state (id, user_id, source, scope_key, last_synced_at, last_external_id, updated_at)
  - user_settings (id, user_id, important_senders, keyword_rules, whatsapp_group_allowlist, notification_preferences, created_at, updated_at)
  - oauth_tokens (id, user_id, provider, access_token, refresh_token, expires_at, scopes, created_at, updated_at)
- **Foreign Key Constraints**:
  - conversations.user_id → users.id (ON DELETE CASCADE)
  - conversation_messages.conversation_id → conversations.id (ON DELETE CASCADE)
  - sync_state.user_id → users.id (ON DELETE CASCADE)
  - user_settings.user_id → users.id (ON DELETE CASCADE)
  - oauth_tokens.user_id → users.id (ON DELETE CASCADE)
- **Database Indexes**:
  - user_id indexes on all tables referencing users
  - Timestamp indexes (created_at, updated_at, last_synced_at)
  - Unique constraints (clerk_id on users table)
- **Connection Credentials**: SUPABASE_URL and SUPABASE_KEY environment variable values (or DATABASE_URL for direct PostgreSQL access)
- **Migration Scripts**: Versioned SQL migration files in repository
- **Documentation**: Schema documentation and migration guide

## 5. Internal Responsibilities

1. **Supabase Project Creation**:
   - Navigate to Supabase dashboard and create new project
   - Configure project settings (name, region, database password)
   - Wait for database provisioning to complete
   - Verify database connection from dashboard

2. **Migration Framework Setup**:
   - Install migration tool (Alembic for Python or use Supabase CLI)
   - Initialize migration directory structure
   - Configure connection to Supabase database
   - Create initial migration file

3. **Core Table Schema Implementation**:
   - Define users table with Clerk Auth fields (clerk_id, email, username)
   - Define conversations table with user relationship
   - Define conversation_messages table with conversation relationship
   - Define sync_state table with multi-source cursor tracking
   - Define user_settings table with JSONB configuration fields
   - Define oauth_tokens table with encrypted token storage

4. **Foreign Key Relationships**:
   - Establish user_id foreign keys with CASCADE delete behavior
   - Establish conversation_id foreign key with CASCADE delete behavior
   - Verify referential integrity constraints

5. **Index Creation**:
   - Create indexes on user_id columns across all tables
   - Create indexes on timestamp columns (created_at, updated_at, last_synced_at)
   - Create unique index on users.clerk_id and index on users.email
   - Create composite indexes where needed (e.g., user_id + source on sync_state)

6. **Migration Execution**:
   - Run migration to apply schema changes
   - Verify all tables created successfully
   - Verify all constraints and indexes in place
   - Test rollback capability

7. **Connection Configuration**:
   - Extract SUPABASE_URL and SUPABASE_KEY from Supabase dashboard
   - Extract DATABASE_URL (PostgreSQL URI) for direct database access
   - Document connection credentials and security requirements
   - Create .env.example with SUPABASE_URL, SUPABASE_KEY, and DATABASE_URL placeholders

8. **Validation and Testing**:
   - Verify table creation with SELECT queries
   - Test foreign key constraints with sample data
   - Test unique constraints (duplicate clerk_id)
   - Verify cascade delete behavior
   - Document schema in README or separate schema documentation file

## 6. Dependencies

**External Services:**
- Supabase account (free tier sufficient for MVP)
- PostgreSQL 14+ (provided by Supabase)

**Development Tools:**
- Migration framework:
  - Alembic (Python) OR
  - Supabase CLI with migration support
- Database client for verification (psql, DBeaver, or Supabase dashboard)

**Ticket Dependencies:**
- None - This is the foundational ticket that all other tickets depend on

**Blocks:**
- Implement_Raw_Data_Tables_(Gmail,_Classroom,_WhatsApp) ticket
- Implement_Filtered_Data_Tables_(Emails,_Events,_Assignments,_Announcements,_WhatsApp_Summaries) ticket
- FastAPI_Project_Setup_&_Authentication ticket
- All subsequent backend and worker tickets

## 7. Execution Model

**Type**: One-time Database Setup and Migration

**Execution Steps**:
1. **Initial Setup** (one-time):
   - Create Supabase project via web dashboard
   - Configure project settings
   - Set up local migration framework

2. **Migration Development** (one-time per schema change):
   - Write migration script defining all 6 tables
   - Test migration locally if using local PostgreSQL
   - Review migration SQL for correctness

3. **Migration Deployment** (one-time):
   - Run migration against Supabase database
   - Verify schema changes applied
   - Document DATABASE_URL for application use

4. **Future Updates** (as needed):
   - Create new migration files for schema changes
   - Apply migrations in sequence
   - Maintain migration history in version control

**Concurrency**: Single-threaded, sequential execution required

**Idempotency**: Migrations should be idempotent (use IF NOT EXISTS, or track applied migrations)

**Rollback**: Migration framework should support rollback for failed migrations

## 8. Failure Handling

**Supabase Project Creation Failures**:
- **Issue**: Project provisioning fails or times out
- **Handling**: Retry project creation or contact Supabase support; verify account limits not exceeded

**Migration Execution Failures**:
- **Issue**: SQL syntax errors, constraint violations, or connection failures
- **Handling**:
  - Review migration script for syntax errors
  - Verify database connection string
  - Check for existing tables/constraints that conflict
  - Rollback migration and fix errors before re-applying
  - Log detailed error messages for debugging

**Connection Failures**:
- **Issue**: Unable to connect to Supabase database
- **Handling**:
  - Verify DATABASE_URL is correct
  - Check network connectivity
  - Verify Supabase project is active
  - Check firewall rules and IP allowlisting

**Constraint Violation Errors**:
- **Issue**: Foreign key or unique constraint fails during migration
- **Handling**:
  - Verify migration order (create parent tables before child tables)
  - Check for duplicate data (e.g., duplicate usernames)
  - Review constraint definitions for correctness

**Data Loss Prevention**:
- All migrations must be reversible
- Test migrations on local or staging database first
- Backup database before applying migrations to production
- Use transactions to ensure atomic migration execution

## 9. Observability

**Logging**:
- Migration script execution logs (start, completion, errors)
- SQL statements executed during migration
- Table creation confirmations
- Index creation confirmations
- Constraint creation confirmations
- Connection string generation log (redact password)

**Verification Queries**:
```sql
-- Verify tables created
SELECT table_name FROM information_schema.tables
WHERE table_schema = 'public' AND table_type = 'BASE TABLE';

-- Verify foreign keys
SELECT constraint_name, table_name, constraint_type
FROM information_schema.table_constraints
WHERE constraint_schema = 'public';

-- Verify indexes
SELECT tablename, indexname FROM pg_indexes
WHERE schemaname = 'public';

-- Count rows in each table (should be 0 initially)
SELECT COUNT(*) FROM users;
SELECT COUNT(*) FROM conversations;
```

**Monitoring**:
- Supabase dashboard for database health
- Database size and storage usage
- Connection pool status
- Query performance metrics (baseline for future optimization)

**Documentation**:
- Schema diagram (ERD) showing table relationships
- Migration history log
- Connection configuration guide
- Troubleshooting guide for common migration issues

## 10. Security Considerations

**Credentials Management**:
- Database password stored securely (never commit to Git)
- DATABASE_URL environment variable used in applications
- .env.example provided with placeholder, not actual credentials
- Consider using secret management service (e.g., AWS Secrets Manager) for production

**Access Token Security**:
- oauth_tokens table stores encrypted access_token and refresh_token fields
- Encryption at rest provided by Supabase (PostgreSQL transparent data encryption)
- Consider application-level encryption for sensitive fields (use pgcrypto extension)

**Database Access Control**:
- Supabase Row Level Security (RLS) policies not required for MVP (all access via backend API)
- Backend application uses service role key (full database access)
- Frontend never accesses database directly
- Consider implementing RLS policies in Phase 2 for defense-in-depth

**Connection Security**:
- All connections to Supabase use SSL/TLS encryption
- DATABASE_URL includes sslmode parameter
- No unencrypted connections allowed

**User Data Isolation**:
- All tables include user_id foreign key for data isolation
- Application logic must enforce user_id filtering on all queries
- ON DELETE CASCADE ensures user data is completely removed on account deletion

**User Authentication**:
- User authentication is handled entirely by Clerk Auth (no passwords stored in database)
- `users.clerk_id` stores the Clerk user identifier (e.g., `user_2AbCdEfGhIjKlMnOpQrSt`)
- Users are created via Clerk webhook when they sign up, not manually seeded
- `users.email` is required and synced from Clerk

## 11. Scaling Considerations

**Current MVP Constraints**:
- Single user (no multi-tenancy required)
- Single Supabase project
- Free tier sufficient for Phase 1 (up to 500 MB database size)

**Future Scaling Path**:
- Upgrade to Supabase paid tier for larger storage and connection limits
- Implement database connection pooling (PgBouncer) for high-concurrency workloads
- Add read replicas for query performance (Supabase Pro tier)
- Partition large tables by user_id if multi-user support is added

**Performance Optimization**:
- Indexes already created on high-query columns (user_id, timestamps)
- Monitor query performance using Supabase dashboard
- Add additional indexes based on query patterns observed in production
- Consider materialized views for complex aggregations (future enhancement)

**Storage Management**:
- Monitor database size growth
- Implement data retention policies (e.g., archive old conversation messages)
- Raw data cleanup task (14-day retention) handled by Celery worker in separate ticket
- Consider moving large JSONB fields (raw_metadata) to object storage if needed

## 12. Future Extensions

**Phase 2 Enhancements**:
- **Row Level Security (RLS)**: Implement Supabase RLS policies for multi-user support
- **Real-time Subscriptions**: Enable Supabase real-time for live sync status updates in frontend
- **Audit Logging**: Add audit_log table to track all schema changes and data modifications
- **Multi-tenancy**: Partition data by tenant if supporting multiple users or organizations
- **Backup and Restore**: Implement automated backup strategy with point-in-time recovery
- **Database Monitoring**: Integrate with monitoring tools (Datadog, Prometheus) for alerts on slow queries, connection pool exhaustion, and disk space

**Advanced Features**:
- **Full-Text Search**: Add PostgreSQL full-text search indexes for message content and email bodies
- **Time-Series Data**: Optimize sync_state for time-series query patterns
- **Encryption at Rest**: Implement application-level encryption for oauth_tokens using pgcrypto
- **Database Versioning**: Implement semantic versioning for schema changes
- **Blue-Green Deployments**: Support zero-downtime migrations for production deployments
- **Cross-Region Replication**: Replicate database to multiple regions for disaster recovery
