# Implement Raw Data Tables (Gmail, Classroom, WhatsApp)

## Objective

Create all raw data tables for storing unprocessed data from Gmail, Google Classroom, and WhatsApp.

## Scope

**In Scope:**
- Implement raw data tables:
  - `raw_emails` (id, user_id, gmail_message_id, subject, body, sender, recipients, received_at, labels, raw_metadata, created_at)
  - `raw_events` (id, user_id, google_event_id, title, description, start_time, end_time, location, attendees, raw_metadata, created_at)
  - `raw_assignments` (id, user_id, classroom_assignment_id, course_name, title, description, due_date, materials, raw_metadata, created_at)
  - `raw_classroom_announcements` (id, user_id, classroom_announcement_id, course_name, title, text, announced_by, announced_at, raw_metadata, created_at)
  - `raw_messages` (id, user_id, chat_name, is_group, sender, message_batch, batch_start_time, batch_end_time, created_at)
- Set up unique constraints (gmail_message_id, google_event_id, classroom_assignment_id, classroom_announcement_id)
- Set up composite unique constraint for WhatsApp batches (user_id, chat_name, batch_start_time, batch_end_time)
- Create indexes on user_id, timestamps, and external IDs
- Set up foreign key constraints with `ON DELETE CASCADE` for user_id references

**Out of Scope:**
- Filtered/processed data tables
- Application code

## Spec References

- `spec:69889a16-05f6-4f4a-bf1a-8cf0daec03b5/4eb58cf0-bbfc-42e3-acd5-9fe60dd4f30b` (Tech Plan - Raw Data Tables)

## Acceptance Criteria

- [ ] 5 raw data tables created with correct schema (including raw_classroom_announcements)
- [ ] Unique constraints prevent duplicate external IDs
- [ ] WhatsApp batch uniqueness enforced
- [ ] Indexes created for performance
- [ ] Migration scripts committed

## Dependencies

- Previous ticket (core tables)
