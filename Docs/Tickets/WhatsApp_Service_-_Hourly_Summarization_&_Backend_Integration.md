# WhatsApp Service - Hourly Summarization & Backend Integration

## Objective

Implement hourly summarization logic using Gemini API and backend communication for WhatsApp service.

**Note:** This ticket requires changes to both the WhatsApp Node.js service AND the FastAPI backend (for the `/ingest/whatsapp` endpoint).

## Scope

**In Scope:**
- Hourly summarization process:
  - At top of each hour, select chats:
    - All private chats (always)
    - Allowlisted groups only (if allowlist is empty/null, no groups are summarized)
  - For each selected chat, call Gemini to extract: summary, key points, deadlines, importance score
  - Send payload to backend `POST /ingest/whatsapp`
- Gemini API client
- Backend API client with authentication
- Failure recovery:
  - Resume from persisted buffer on restart
  - Retry backend POST with exponential backoff
  - Persist unsent payloads until acknowledged
  - Fallback: send raw batch if Gemini fails
- Backend ingestion endpoint in FastAPI: `POST /ingest/whatsapp`

**Out of Scope:**
- Message monitoring (previous ticket)

## Spec References

- `spec:69889a16-05f6-4f4a-bf1a-8cf0daec03b5/4eb58cf0-bbfc-42e3-acd5-9fe60dd4f30b` (Tech Plan - WhatsApp Service, Hourly Summarization)

## Acceptance Criteria

- [ ] Hourly summarization runs at top of each hour
- [ ] Gemini API called for each selected chat
- [ ] Summary, key points, deadlines extracted
- [ ] Payload sent to backend successfully
- [ ] Backend endpoint saves raw + filtered data
- [ ] Retries on backend POST failure
- [ ] Fallback to raw batch if Gemini fails
- [ ] Unsent payloads persisted

## Dependencies

- Previous ticket (WhatsApp monitoring)
- FastAPI Auth ticket (for API key authentication middleware)
