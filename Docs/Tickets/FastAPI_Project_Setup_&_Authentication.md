# FastAPI Project Setup & Authentication

## Objective

Set up FastAPI project structure with authentication (username/password login, JWT tokens) and Supabase database connection.

## Scope

**In Scope:**
- FastAPI project initialization
- Project structure (routers, models, services, config)
- Environment configuration (.env support for DATABASE_URL, JWT_SECRET, GEMINI_API_KEY, WHATSAPP_API_KEY)
- Supabase client setup
- Authentication endpoints:
  - `POST /auth/login` - username/password login, returns JWT
  - `POST /auth/logout` - logout
  - `GET /auth/me` - get current user
- JWT token generation and validation middleware
- API key authentication middleware (for WhatsApp service)
- Password hashing (bcrypt)
- Error handling middleware
- CORS configuration
- API documentation (automatic via FastAPI)

**Out of Scope:**
- Conversation endpoints
- Chat/conversational agent logic
- Data ingestion endpoints

## Spec References

- `spec:69889a16-05f6-4f4a-bf1a-8cf0daec03b5/4eb58cf0-bbfc-42e3-acd5-9fe60dd4f30b` (Tech Plan - FastAPI Backend)

## Acceptance Criteria

- [ ] FastAPI app runs and serves docs at `/docs`
- [ ] Supabase connection works
- [ ] Initial user seeded (via migration or management command)
- [ ] User can login with username/password
- [ ] JWT token is generated and returned
- [ ] Protected endpoints require valid JWT
- [ ] `/auth/me` returns current user info
- [ ] Password is hashed in database
- [ ] Environment variables loaded from .env
- [ ] Error responses are consistent

## Dependencies

- Database schema complete (all 3 database tickets)
