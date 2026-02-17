# Setup Supabase Project & Core Tables

## Objective

Create Supabase project and implement core database tables (users, conversations, sync_state) that are foundational for all other components.

## Scope

**In Scope:**
- Create Supabase project
- Configure database connection
- Implement core tables:
  - `users` (id, username, password_hash, timestamps)
  - `conversations` (id, user_id, title, timestamps)
  - `conversation_messages` (id, conversation_id, role, content, timestamp)
  - `sync_state` (id, user_id, source, scope_key, last_synced_at, last_external_id, updated_at)
  - `user_settings` (id, user_id, important_senders, keyword_rules, whatsapp_group_allowlist, notification_preferences, timestamps)
  - `oauth_tokens` (id, user_id, provider, access_token, refresh_token, expires_at, timestamps)
- Set up foreign key constraints
- Create indexes on user_id and timestamps
- Set up migration framework (Alembic or Supabase migrations)

**Out of Scope:**
- Data source tables (emails, events, assignments, messages)
- Application code

## Spec References

- `spec:69889a16-05f6-4f4a-bf1a-8cf0daec03b5/4eb58cf0-bbfc-42e3-acd5-9fe60dd4f30b` (Tech Plan - Data Model)

## Acceptance Criteria

- [ ] Supabase project created and accessible
- [ ] 6 core tables created with correct schema
- [ ] Foreign keys and indexes in place
- [ ] Database connection string available
- [ ] Migration scripts committed

## Dependencies

None