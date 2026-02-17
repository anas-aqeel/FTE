# Conversational Agent with Gemini Integration

## Objective

Implement the core conversational agent that queries Supabase (filtered → raw fallback) and uses Gemini API to generate natural language responses.

## Scope

**In Scope:**
- Gemini API client setup
- Chat endpoint: `POST /conversations/{id}/messages`
- Conversational agent logic:
  1. Retrieve conversation history
  2. Query filtered data tables (emails, events, assignments, announcements, whatsapp_summaries)
  3. If insufficient data, fallback to raw tables
  4. Construct prompt with: system instructions + conversation history + retrieved data + user query
  5. Call Gemini API
  6. Save user message and assistant response to database
  7. Return response to user
- Query logic for filtered data (filter by user_id, order by importance_score, recent first)
- Raw data fallback logic
- Prompt engineering for Gemini
- Error handling for Gemini API failures

**Out of Scope:**
- Data ingestion
- Sync status endpoints

## Spec References

- `spec:69889a16-05f6-4f4a-bf1a-8cf0daec03b5/381a7a45-1b56-45fb-a2eb-bc0e7799dd92` (Epic Brief - Success Metrics)
- `spec:69889a16-05f6-4f4a-bf1a-8cf0daec03b5/4eb58cf0-bbfc-42e3-acd5-9fe60dd4f30b` (Tech Plan - Conversational Agent)

## Acceptance Criteria

- [ ] User can send a message to a conversation
- [ ] Agent queries filtered data tables first
- [ ] Agent falls back to raw data if needed
- [ ] Agent constructs prompt with conversation history + data
- [ ] Gemini API is called successfully
- [ ] Response is saved to database
- [ ] Response is returned to user
- [ ] Conversation history is maintained
- [ ] Handles Gemini API errors gracefully

## Dependencies

- Previous ticket (conversation management)