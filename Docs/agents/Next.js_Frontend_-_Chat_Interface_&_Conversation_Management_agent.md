# Agent Specification: Next.js Frontend - Chat Interface & Conversation Management

## 1. Purpose

Implement the complete chat interface with conversation thread management, real-time message display, sync status indicators, and "Refetch Data" functionality in the Next.js frontend for the Personal Assistant Agent system.

## 2. Scope

**In Scope:**
- Conversation thread list in sidebar:
  - Fetch and display all user's conversations (GET /conversations)
  - "New Conversation" button (POST /conversations)
  - Click conversation to switch/load
  - Delete conversation button (DELETE /conversations/{id})
  - Show conversation title and last updated timestamp
- Chat interface (main content area):
  - Display message history for selected conversation (GET /conversations/{id}/messages)
  - Message list with user/assistant message bubbles
  - Auto-scroll to latest message on load and new messages
  - Loading indicator while fetching messages
  - Pagination or infinite scroll for long conversations
- Message input and send:
  - Text input area (textarea, auto-resize)
  - Send button
  - Handle Enter key (send), Shift+Enter (new line)
  - Loading state during API call (POST /conversations/{id}/messages)
  - Optimistic UI update (show user message immediately, then add assistant response)
- Sync status indicator:
  - Display last sync time for Gmail, Classroom, WhatsApp (GET /sync/status)
  - "Refetch Data" button triggers ingestion (POST /ingest/trigger)
  - Show sync in progress state (polling GET /ingest/status)
  - Auto-refresh status every 60 seconds
- State management:
  - Global state for conversations list, selected conversation, messages
  - Use React Context, Zustand, or Redux
- Error handling and user feedback:
  - Show error messages (API failures, validation errors)
  - Toast notifications for success/error states
- Responsive design:
  - Mobile: Full-screen chat, sidebar toggleable
  - Desktop: Split view (sidebar + chat)

**Out of Scope:**
- Advanced features: message search, filters, conversation folders
- Settings page (user preferences)
- Notification preferences UI
- Real-time WebSocket updates (Phase 2)
- Voice input (Phase 2)
- File attachments (Phase 2)

## 3. Inputs

- **Backend API Endpoints**:
  - GET /conversations (paginated)
  - POST /conversations {title?}
  - GET /conversations/{id}/messages (paginated)
  - POST /conversations/{id}/messages {content}
  - DELETE /conversations/{id}
  - GET /sync/status
  - POST /ingest/trigger {sources?}
  - GET /ingest/status?task_ids=...
- **User Interactions**:
  - Click "New Conversation"
  - Click conversation in sidebar
  - Type message and click Send (or press Enter)
  - Click "Refetch Data"
  - Click Delete conversation
- **Environment Variables**:
  - NEXT_PUBLIC_API_URL (from previous ticket)

## 4. Outputs

- **Conversation Sidebar UI**:
  - List of conversations (titles, timestamps)
  - "New Conversation" button
  - Active conversation highlighted
  - Delete button (icon) per conversation
- **Chat Interface UI**:
  - Message history (user and assistant messages styled differently)
  - Message input textarea
  - Send button
  - Loading spinner for assistant response
  - Auto-scroll to bottom
- **Sync Status UI**:
  - Sync status badges (Gmail: synced, Classroom: synced, WhatsApp: synced)
  - Last sync timestamps
  - "Refetch Data" button
  - Progress indicator during ingestion
- **State Updates**:
  - Conversations list updated on create/delete
  - Messages list updated on send/receive
  - Selected conversation tracked
  - Sync status refreshed every 60 seconds
- **API Calls**:
  - GET /conversations on page load
  - GET /conversations/{id}/messages when conversation selected
  - POST /conversations/{id}/messages when user sends message
  - POST /conversations on "New Conversation"
  - DELETE /conversations/{id} on delete
  - GET /sync/status on page load and every 60 seconds
  - POST /ingest/trigger on "Refetch Data" click
  - GET /ingest/status (polling) when ingestion in progress

## 5. Internal Responsibilities

1. **Conversation List Component**:
   - Fetch conversations on mount (GET /conversations)
   - Render conversation list in sidebar
   - Handle "New Conversation" button click: POST /conversations, add to list, select new conversation
   - Handle conversation click: set selected conversation, fetch messages
   - Handle delete: DELETE /conversations/{id}, remove from list, select first remaining conversation

2. **Chat Message List Component**:
   - Fetch messages when conversation selected (GET /conversations/{id}/messages)
   - Render message bubbles (user messages on right, assistant on left)
   - Auto-scroll to bottom on mount and new messages (useEffect with scrollIntoView)
   - Implement pagination or infinite scroll (load older messages on scroll to top)

3. **Message Input Component**:
   - Controlled textarea (value from state, onChange updates state)
   - Handle Send button click: POST /conversations/{id}/messages, optimistically add user message to list, wait for response, add assistant message
   - Handle Enter key: submit form (Shift+Enter for new line)
   - Show loading spinner while waiting for response
   - Clear input after successful send

4. **Sync Status Component**:
   - Fetch sync status on mount (GET /sync/status)
   - Render status badges for each source (color-coded: green=synced, yellow=in_progress, red=error)
   - Display last sync timestamps (format: "5 minutes ago")
   - Auto-refresh every 60 seconds (setInterval)
   - Handle "Refetch Data" button: POST /ingest/trigger, poll GET /ingest/status until complete, update UI

5. **State Management**:
   - Global state: {conversations: [], selectedConversation: null, messages: [], syncStatus: {}, isLoading: {}}
   - Actions: fetchConversations, createConversation, selectConversation, deleteConversation, fetchMessages, sendMessage, fetchSyncStatus, triggerIngestion
   - Use React Context or Zustand for state

6. **API Client Integration**:
   - Use API client from previous ticket (lib/api.ts)
   - Wrap API calls in try-catch, handle errors (show toast notifications)
   - Add retry logic for transient errors

7. **Optimistic UI Updates**:
   - When user sends message: immediately add to messages list (role=user), show loading spinner, wait for API response, add assistant message
   - If API fails: remove optimistic message, show error

8. **Error Handling**:
   - API errors: Show toast notification (e.g., "Failed to send message")
   - Network errors: Show retry button
   - Validation errors: Show inline error message (e.g., "Message cannot be empty")

9. **Responsive Design**:
   - Mobile: Sidebar toggleable (hamburger menu), full-screen chat when conversation selected
   - Desktop: Split view (sidebar 1/4, chat 3/4)
   - Use Tailwind CSS responsive breakpoints

10. **Toast Notifications**:
    - Install react-hot-toast or similar library
    - Show success toast: "Conversation created", "Data refetch triggered"
    - Show error toast: "Failed to load conversations", "Failed to send message"

## 6. Dependencies

**External Services:**
- FastAPI Backend (all conversation, chat, sync endpoints)

**Libraries/Tools:**
- Next.js 14+ (already set up)
- React Context or Zustand (state management)
- Axios (API client)
- Tailwind CSS (styling)
- react-hot-toast (notifications)
- date-fns or dayjs (timestamp formatting)

**Ticket Dependencies:**
- Next.js_Frontend_-_Authentication_&_Layout (provides layout and auth)
- Conversation_Management_API (provides conversation endpoints)
- Conversational_Agent_with_Gemini_Integration (provides chat endpoint)
- Sync_Status_&_Data_Management_Endpoints (provides sync endpoints)

**Blocks:**
- Docker_Compose_Setup_&_Deployment_Documentation (needs complete frontend)

## 7. Execution Model

**Type**: Client-Side React Application (CSR)

**Lifecycle**:
- Component mount: Fetch conversations, sync status
- User actions: API calls, state updates, re-render
- Auto-refresh: Sync status every 60 seconds

**Rendering**:
- Client-side rendering for dynamic content (messages, conversations)
- Optimistic UI updates for instant feedback

**Response Time**:
- Conversation list: <500ms
- Message send: 2-5 seconds (waiting for Gemini)
- Sync status: <200ms

## 8. Failure Handling

**API Failures:**
- Failed to load conversations: Show error message, retry button
- Failed to send message: Remove optimistic message, show error toast
- Failed to fetch sync status: Show "Unable to fetch status", retry after 60s

**Network Errors:**
- Timeout: Retry once, then show error
- Offline: Detect navigator.onLine, show "You are offline" banner

**Optimistic UI Failures:**
- If POST /conversations/{id}/messages fails: Remove optimistic user message from list, show error

**Sync Status Polling:**
- If GET /ingest/status fails: Stop polling, show error, retry on next manual trigger

## 9. Observability

**Logging** (client-side):
- Component mount (conversations, messages loaded)
- User actions (send message, create conversation, delete conversation, trigger sync)
- API calls (endpoint, status, duration)
- Errors (API failures, validation errors)

**Monitoring** (future):
- Frontend error tracking (Sentry)
- Performance monitoring (Core Web Vitals)
- User analytics (message send frequency, conversation creation)

## 10. Security Considerations

**Data Privacy:**
- All API calls use JWT authentication (handled by API client)
- Never store sensitive data in localStorage (only JWT token)

**XSS Prevention:**
- Sanitize assistant responses (React handles by default, but be cautious with markdown rendering)
- Never use dangerouslySetInnerHTML with user or API content

**Rate Limiting** (frontend side):
- Debounce "Refetch Data" button (prevent rapid clicks)
- Limit message send frequency (max 1 message per 2 seconds)

**Input Validation:**
- Validate message content (not empty, max 5000 chars)
- Trim whitespace

## 11. Scaling Considerations

**Current MVP:**
- Single user
- Low message frequency
- Simple state management

**Performance:**
- Optimize re-renders (React.memo, useMemo, useCallback)
- Lazy load old messages (pagination)
- Debounce auto-refresh (avoid excessive API calls)

**Future Scaling:**
- WebSocket for real-time updates (replace polling)
- Virtual scrolling for long message lists (react-window)
- Service worker for offline support

## 12. Future Extensions

**Phase 2:**
- Real-time updates via WebSocket (new messages, sync status changes)
- Message search functionality
- Conversation folders/tags
- Voice input (speech-to-text)
- File attachments (images, documents)
- Markdown rendering in messages
- Code syntax highlighting (for code snippets in responses)
- Export conversation to PDF/text
- Conversation sharing (public link)
- Message reactions (thumbs up/down)
- Dark mode toggle
- Keyboard shortcuts (Ctrl+K for new conversation, Ctrl+Enter for send)
- Accessibility improvements (screen reader support, ARIA labels)
