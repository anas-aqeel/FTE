# Implement Filtered Data Tables (Emails, Events, Assignments, Announcements, WhatsApp Summaries)

## Objective

Create all filtered/processed data tables for storing AI-processed, labeled, and scored data.

## Scope

**In Scope:**
- Implement filtered data tables:
  - `emails` (id, user_id, raw_email_id, subject, summary, sender, importance_score, category, extracted_deadline, is_seen, source, received_at, processed_at)
  - `events` (id, user_id, raw_event_id, title, description, start_time, end_time, location, importance_score, event_type, is_seen, source, processed_at)
  - `assignments` (id, user_id, raw_assignment_id, course_name, title, description, due_date, importance_score, status, is_seen, source, processed_at)
  - `announcements` (id, user_id, source_type, source_id, title, content, announced_by, importance_score, category, is_seen, announced_at, processed_at)
  - `whatsapp_summaries` (id, user_id, raw_message_id, chat_name, is_group, summary, key_points, mentioned_deadlines, importance_score, is_seen, batch_start_time, batch_end_time, processed_at)
- Set up foreign keys to raw tables
- Create indexes on user_id, importance_score, is_seen, timestamps

**Out of Scope:**
- Application code

## Spec References

- `spec:69889a16-05f6-4f4a-bf1a-8cf0daec03b5/4eb58cf0-bbfc-42e3-acd5-9fe60dd4f30b` (Tech Plan - Filtered Data Tables)

## Acceptance Criteria

- [ ] 5 filtered data tables created with correct schema
- [ ] Foreign keys link to raw tables
- [ ] Indexes created for query performance
- [ ] Migration scripts committed
- [ ] Database schema complete

## Dependencies

- Previous ticket (raw data tables)