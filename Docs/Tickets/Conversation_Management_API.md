# Conversation Management API

## Objective

Implement conversation thread management endpoints (create, list, get messages, delete).

## Scope

**In Scope:**
- Conversation endpoints:
  - `GET /conversations` - list user's conversations
  - `POST /conversations` - create new conversation
  - `GET /conversations/{id}/messages` - get conversation history
  - `DELETE /conversations/{id}` - delete conversation
- Database operations for conversations and messages
- Auto-generate conversation titles (or allow user to set)
- Pagination for message history
- Authorization (users can only access their own conversations)

**Out of Scope:**
- Chat message endpoint (conversational agent)
- Gemini API integration

## Spec References

- `spec:69889a16-05f6-4f4a-bf1a-8cf0daec03b5/4eb58cf0-bbfc-42e3-acd5-9fe60dd4f30b` (Tech Plan - FastAPI Backend, Conversations section)

## Acceptance Criteria

- [ ] User can create a new conversation
- [ ] User can list all their conversations
- [ ] User can get message history for a conversation
- [ ] User can delete a conversation
- [ ] Users cannot access other users' conversations
- [ ] Message history is paginated
- [ ] All endpoints require authentication

## Dependencies

- Previous ticket (FastAPI auth setup)