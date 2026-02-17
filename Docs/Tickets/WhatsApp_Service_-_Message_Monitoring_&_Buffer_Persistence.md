# WhatsApp Service - Message Monitoring & Buffer Persistence

## Objective

Build Node.js service for 24/7 WhatsApp monitoring with hourly buffer persistence to disk.

## Scope

**In Scope:**
- Node.js project setup with whatsapp-web.js
- WhatsApp Web connection and authentication (QR code)
- 24/7 message monitoring (all private + group chats)
- Hourly buffer persistence to local disk (SQLite or append-only files)
- Session persistence (avoid QR re-authentication)
- Environment configuration (BACKEND_API_URL, GEMINI_API_KEY, WHATSAPP_API_KEY)
- Dockerfile for VPS deployment
- Logging and monitoring

**Out of Scope:**
- Summarization logic
- Backend communication
- Gemini API integration

## Spec References

- `spec:69889a16-05f6-4f4a-bf1a-8cf0daec03b5/4eb58cf0-bbfc-42e3-acd5-9fe60dd4f30b` (Tech Plan - WhatsApp Service, Architectural Decision 8)

## Acceptance Criteria

- [ ] Service connects to WhatsApp Web
- [ ] All messages monitored 24/7
- [ ] Hourly buffer persisted to disk
- [ ] Buffer survives container restarts
- [ ] WhatsApp session persists (no QR re-scan)
- [ ] Dockerfile ready for VPS deployment
- [ ] Logging configured

## Dependencies

- None (can be developed in parallel)