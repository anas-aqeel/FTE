# Setup Supabase Project & Core Tables

## Objective

Create Supabase project and implement core database tables (users with Clerk Auth, conversations, sync_state) that are foundational for all other components.

## Scope

**In Scope:**
- Create Supabase project
- Configure database connection
- Implement core tables:
  - `users` (id, **clerk_id**, email, username, timestamps) - **Updated for Clerk Auth**
  - `conversations` (id, user_id, title, timestamps)
  - `conversation_messages` (id, conversation_id, role, content, timestamp)
  - `sync_state` (id, user_id, source, scope_key, last_synced_at, last_external_id, updated_at)
  - `user_settings` (id, user_id, important_senders, keyword_rules, whatsapp_group_allowlist, notification_preferences, timestamps)
  - `oauth_tokens` (id, user_id, provider, access_token, refresh_token, expires_at, timestamps) - **For Google API access**
- Set up foreign key constraints
- Create indexes on user_id, clerk_id, and timestamps
- Set up migration framework (Alembic or Supabase migrations)

**Out of Scope:**
- Data source tables (emails, events, assignments, messages)
- Application code

**Note:** The `users` table now stores `clerk_id` (Clerk user identifier) instead of `password_hash`. User authentication is handled by Clerk.

## Spec References

- `spec:69889a16-05f6-4f4a-bf1a-8cf0daec03b5/4eb58cf0-bbfc-42e3-acd5-9fe60dd4f30b` (Tech Plan - Data Model)
- `docs/specs/OAuth_Setup_Guide.md` (Database Schema Updates section)

## Acceptance Criteria

- [ ] Supabase project created and accessible
- [ ] 6 core tables created with correct schema
- [ ] `users` table has `clerk_id` column (NOT `password_hash`)
- [ ] Foreign keys and indexes in place
- [ ] Index on `clerk_id` for fast lookups
- [ ] Database connection string available
- [ ] Migration scripts committed

## Dependencies

None

---

## Implementation Details

### 1. Create Supabase Project

1. Go to https://supabase.com
2. Click "New Project"
3. Project name: **"Personal Assistant"**
4. Database password: Generate strong password (save securely)
5. Region: Select closest to your location
6. Click "Create new project"
7. Wait for project provisioning (~2 minutes)

### 2. Get Connection Details

From Supabase Dashboard:

```
Project URL: https://your-project.supabase.co
API Keys:
  - anon/public: eyJhbGc... (for client-side)
  - service_role: eyJhbGc... (for backend - keep secret!)

Database Connection:
  - PostgreSQL URI: postgresql://postgres:[password]@db.your-project.supabase.co:5432/postgres
```

Add to `.env`:
```bash
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_KEY=your-service-role-key  # service_role key for backend
DATABASE_URL=postgresql://postgres:[password]@db.your-project.supabase.co:5432/postgres
```

### 3. Database Schema

**Migration File:** `migrations/001_core_tables.sql`

```sql
-- ==========================================
-- Users Table (Clerk Auth)
-- ==========================================
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    clerk_id VARCHAR(255) UNIQUE NOT NULL,  -- Clerk user ID (e.g., user_2xxx...)
    email VARCHAR(255) NOT NULL,
    username VARCHAR(255),  -- Optional, fallback to email prefix
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes for performance
CREATE INDEX idx_users_clerk_id ON users(clerk_id);
CREATE INDEX idx_users_email ON users(email);

COMMENT ON TABLE users IS 'User accounts synced from Clerk Auth';
COMMENT ON COLUMN users.clerk_id IS 'Unique identifier from Clerk (e.g., user_2AbCdEfGhIjKlMnOpQrSt)';
COMMENT ON COLUMN users.email IS 'Primary email from Clerk';
COMMENT ON COLUMN users.username IS 'Username from Clerk or derived from email';

-- ==========================================
-- Conversations Table
-- ==========================================
CREATE TABLE conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(500),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_conversations_user_id ON conversations(user_id);
CREATE INDEX idx_conversations_created_at ON conversations(created_at DESC);

COMMENT ON TABLE conversations IS 'Chat conversation threads (like ChatGPT threads)';

-- ==========================================
-- Conversation Messages Table
-- ==========================================
CREATE TABLE conversation_messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    role VARCHAR(50) NOT NULL CHECK (role IN ('user', 'assistant')),
    content TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_conversation_messages_conversation_id ON conversation_messages(conversation_id);
CREATE INDEX idx_conversation_messages_created_at ON conversation_messages(created_at);

COMMENT ON TABLE conversation_messages IS 'Individual messages in conversation threads';

-- ==========================================
-- Sync State Table
-- ==========================================
CREATE TABLE sync_state (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    source VARCHAR(100) NOT NULL,  -- 'gmail', 'classroom', 'whatsapp'
    scope_key VARCHAR(255) NOT NULL,  -- 'global' or 'chat:<chat_name>' for WhatsApp
    last_synced_at TIMESTAMPTZ,
    last_external_id VARCHAR(500),  -- Last processed message/assignment ID
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(user_id, source, scope_key)
);

CREATE INDEX idx_sync_state_user_source ON sync_state(user_id, source);

COMMENT ON TABLE sync_state IS 'Tracks last sync timestamp/cursor for idempotent ingestion';
COMMENT ON COLUMN sync_state.scope_key IS 'Scope of sync - global or per-chat for WhatsApp';

-- ==========================================
-- User Settings Table
-- ==========================================
CREATE TABLE user_settings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL UNIQUE REFERENCES users(id) ON DELETE CASCADE,
    important_senders TEXT[],  -- Array of important email addresses
    keyword_rules JSONB DEFAULT '{}',  -- {"urgent": 0.8, "deadline": 0.7}
    whatsapp_group_allowlist JSONB DEFAULT '{"groups": []}',  -- {groups: [{id, name, added_at}]}
    notification_preferences JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_user_settings_user_id ON user_settings(user_id);

COMMENT ON TABLE user_settings IS 'User preferences for importance scoring and notifications';
COMMENT ON COLUMN user_settings.whatsapp_group_allowlist IS 'Groups to monitor (by ID, not name)';

-- ==========================================
-- OAuth Tokens Table (Google API Access)
-- ==========================================
CREATE TABLE oauth_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider VARCHAR(100) NOT NULL,  -- 'google' (for Gmail, Calendar, Classroom)
    access_token TEXT NOT NULL,  -- Encrypted
    refresh_token TEXT NOT NULL,  -- Encrypted
    expires_at TIMESTAMPTZ NOT NULL,
    scopes TEXT[],  -- Array of granted scopes
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(user_id, provider)
);

CREATE INDEX idx_oauth_tokens_user_id ON oauth_tokens(user_id);
CREATE INDEX idx_oauth_tokens_expires_at ON oauth_tokens(expires_at);

COMMENT ON TABLE oauth_tokens IS 'OAuth tokens for accessing Google APIs (Gmail, Classroom)';
COMMENT ON COLUMN oauth_tokens.access_token IS 'Encrypted Google API access token';
COMMENT ON COLUMN oauth_tokens.refresh_token IS 'Encrypted Google API refresh token';

-- ==========================================
-- Update Trigger for updated_at
-- ==========================================
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply trigger to all tables with updated_at
CREATE TRIGGER update_users_updated_at BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_conversations_updated_at BEFORE UPDATE ON conversations
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_sync_state_updated_at BEFORE UPDATE ON sync_state
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_user_settings_updated_at BEFORE UPDATE ON user_settings
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_oauth_tokens_updated_at BEFORE UPDATE ON oauth_tokens
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

### 4. Run Migration

**Option 1: Supabase SQL Editor**

1. Go to Supabase Dashboard → SQL Editor
2. Paste the migration SQL above
3. Click "Run"
4. Verify tables created in Table Editor

**Option 2: Alembic (for version control)**

```bash
# Install Alembic
pip install alembic psycopg2-binary

# Initialize Alembic
alembic init migrations

# Edit alembic.ini with your DATABASE_URL
# Create migration
alembic revision --autogenerate -m "Create core tables"

# Run migration
alembic upgrade head
```

### 5. Verify Schema

**SQL Query:**
```sql
-- Check all tables
SELECT table_name FROM information_schema.tables
WHERE table_schema = 'public'
ORDER BY table_name;

-- Check users table structure
\d users

-- Expected output includes:
-- clerk_id | character varying(255) | not null

-- Verify indexes
SELECT indexname, indexdef FROM pg_indexes
WHERE schemaname = 'public' AND tablename = 'users';

-- Expected indexes:
-- idx_users_clerk_id
-- idx_users_email
```

### 6. Test Insert (Simulated Clerk User)

```sql
-- Insert test user (simulating Clerk webhook)
INSERT INTO users (clerk_id, email, username)
VALUES ('user_2TestClerkId123', 'test@example.com', 'testuser')
RETURNING *;

-- Verify user created
SELECT * FROM users WHERE clerk_id = 'user_2TestClerkId123';

-- Clean up
DELETE FROM users WHERE clerk_id = 'user_2TestClerkId123';
```

---

## Key Differences from Password-Based Auth

| Aspect | Old (Password) | New (Clerk) |
|--------|---------------|-------------|
| **User Identifier** | `username` (unique) | `clerk_id` (unique) |
| **Password Storage** | `password_hash` (bcrypt) | ❌ None (handled by Clerk) |
| **Email** | Optional | Required |
| **User Creation** | Manual via `/auth/register` | Automatic via Clerk webhook |
| **Authentication** | Custom JWT | Clerk JWT |

---

## Notes

- **clerk_id format:** `user_2AbCdEfGhIjKlMnOpQrSt` (Clerk's stable user ID)
- **No initial user seeding:** Users created when they sign up via Clerk
- **Email is required:** Clerk always provides email
- **Username is optional:** Defaults to email prefix if not provided by Clerk
- **CASCADE DELETE:** Deleting user deletes all their data (conversations, messages, settings, tokens)

---

## Dependencies

None - This is the foundation for all other tickets
