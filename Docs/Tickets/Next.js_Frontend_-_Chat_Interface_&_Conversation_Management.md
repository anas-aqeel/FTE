# Next.js Frontend - Chat Interface & Conversation Management

## Objective

Implement chat interface with conversation thread management, message history, and sync status indicators.

## Scope

**In Scope:**
- Conversation thread list in sidebar:
  - List all conversations
  - Create new conversation button
  - Switch between conversations
  - Delete conversation
- Chat interface:
  - Message history display
  - Message input and send
  - Loading indicator for AI response
  - Auto-scroll to latest message
- Sync status indicator:
  - Display last sync time for Gmail, Classroom, WhatsApp
  - "Refetch Data" button
  - Sync in progress indicator
- State management for conversations and messages
- Real-time updates (polling or optimistic UI)
- Error handling and user feedback

**Out of Scope:**
- Advanced features (search, filters)
- Settings page

## Spec References

- `spec:69889a16-05f6-4f4a-bf1a-8cf0daec03b5/381a7a45-1b56-45fb-a2eb-bc0e7799dd92` (Epic Brief - Success Metrics)
- `spec:69889a16-05f6-4f4a-bf1a-8cf0daec03b5/4eb58cf0-bbfc-42e3-acd5-9fe60dd4f30b` (Tech Plan - Next.js Frontend)

## Acceptance Criteria

- [ ] User sees conversation list in sidebar
- [ ] User can create new conversation
- [ ] User can switch between conversations
- [ ] User can delete conversation
- [ ] Message history displays correctly
- [ ] User can send message and see AI response
- [ ] Sync status shows last sync times
- [ ] "Refetch Data" button triggers ingestion
- [ ] Sync in progress indicator works
- [ ] Responsive on mobile
- [ ] Error messages displayed

## Dependencies

- Previous ticket (frontend auth & layout)
- Backend conversation & chat endpoints