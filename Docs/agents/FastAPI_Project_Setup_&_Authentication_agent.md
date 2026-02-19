# Agent Specification: FastAPI Project Setup & Authentication

## 1. Purpose

Initialize a production-ready FastAPI backend project with Clerk Auth JWT verification, Google OAuth for API access, and API key authentication for external services, serving as the central API gateway for the AcademiQ system.

## 2. Scope

**In Scope:**
- FastAPI project initialization with proper structure (routers, models, services, config)
- Environment configuration (.env support for SUPABASE_URL, SUPABASE_KEY, CLERK_SECRET_KEY, CLERK_WEBHOOK_SECRET, GEMINI_API_KEY, GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET, ENCRYPTION_KEY, WHATSAPP_API_KEY)
- Supabase PostgreSQL client setup with connection pooling
- **Clerk Auth integration:**
  - Clerk JWT verification middleware (verify tokens from frontend)
  - Clerk webhook handler (`POST /webhooks/clerk`) for user sync
  - `get_current_user` dependency that extracts clerk_id from JWT and looks up user
- **Google OAuth endpoints (for API access):**
  - `GET /google/auth-url` - Generate Google OAuth consent URL
  - `GET /google/callback` - Handle Google OAuth callback, store tokens
- Authentication endpoints:
  - `GET /auth/me` - Get current authenticated user (from Clerk JWT)
  - `POST /webhooks/clerk` - Handle Clerk webhook events (user.created, user.updated, user.deleted)
- API key authentication middleware (for WhatsApp service to call backend)
- Error handling middleware with consistent error responses
- CORS configuration for Next.js frontend
- Automatic API documentation via FastAPI (Swagger UI at `/docs`)
- Dependencies installation and requirements.txt

**Out of Scope:**
- ~~Custom login/signup forms~~ (Clerk provides UI)
- ~~Manual JWT generation~~ (Clerk handles JWT issuance)
- ~~Password hashing/storage~~ (Clerk handles authentication)
- ~~Initial user seeding~~ (Users created via Clerk webhook)
- Conversation endpoints (separate ticket)
- Chat/conversational agent logic (separate ticket)
- Data ingestion endpoints (separate tickets)
- Celery worker setup (separate ticket)
- Frontend implementation (separate tickets)

**Note:** All user authentication is handled by Clerk. This ticket focuses on verifying Clerk JWTs, syncing users via webhooks, and managing Google OAuth tokens for API access.

## 3. Inputs

- **Environment Variables** (from .env file):
  - SUPABASE_URL: Supabase project URL
  - SUPABASE_KEY: Supabase service role key
  - CLERK_SECRET_KEY: Clerk secret key (for JWT verification and webhook validation)
  - CLERK_WEBHOOK_SECRET: Clerk webhook signing secret (for verifying webhook payloads)
  - GEMINI_API_KEY: Google Gemini API key for AI processing
  - GOOGLE_CLIENT_ID: Google OAuth client ID (for Gmail/Calendar/Classroom API access)
  - GOOGLE_CLIENT_SECRET: Google OAuth client secret
  - ENCRYPTION_KEY: Key for encrypting/decrypting OAuth tokens at rest
  - WHATSAPP_API_KEY: Shared secret for authenticating WhatsApp service requests
  - CORS_ORIGINS: Comma-separated list of allowed origins (default: http://localhost:3000)
  - FRONTEND_URL: Frontend URL for OAuth callback redirects
  - BACKEND_PORT: Port for FastAPI server (default: 8000)
- **Clerk JWT Token** (for protected endpoints):
  - Authorization header: Bearer {clerk-jwt-token}
  - JWT issued by Clerk, verified using Clerk SDK or JWKS
- **API Key** (for WhatsApp service endpoints):
  - X-API-Key header: {WHATSAPP_API_KEY}
- **Clerk Webhook Events**:
  - user.created: New user signed up
  - user.updated: User profile changed
  - user.deleted: User account deleted
- **Database Schema**: Existing users table from core tables ticket (with clerk_id)

## 4. Outputs

- **FastAPI Application**:
  - Project structure:
    ```
    backend/
    ├── app/
    │   ├── __init__.py
    │   ├── main.py (FastAPI app initialization)
    │   ├── config.py (environment variable loading)
    │   ├── database.py (Supabase client setup)
    │   ├── dependencies.py (authentication dependencies)
    │   ├── models/
    │   │   ├── __init__.py
    │   │   └── user.py (Pydantic models)
    │   ├── routers/
    │   │   ├── __init__.py
    │   │   ├── auth.py (auth endpoints - /auth/me)
    │   │   ├── webhooks.py (Clerk webhook handler)
    │   │   └── google_oauth.py (Google OAuth flow)
    │   ├── services/
    │   │   ├── __init__.py
    │   │   ├── auth_service.py (Clerk JWT verification, user lookup)
    │   │   └── google_oauth_service.py (Google OAuth token management)
    │   └── middleware/
    │       ├── __init__.py
    │       └── error_handler.py
    ├── requirements.txt
    ├── .env.example
    └── Dockerfile (placeholder for Docker ticket)
    ```
- **Authentication Endpoints**:
  - GET /auth/me
    - Input: Clerk JWT token (Authorization header)
    - Output: {id: UUID, clerk_id: string, email: string, username: string, created_at: timestamp}
  - POST /webhooks/clerk
    - Input: Clerk webhook payload (signed with CLERK_WEBHOOK_SECRET)
    - Output: {status: "ok"} (200) or error
  - GET /google/auth-url
    - Input: Clerk JWT token (Authorization header)
    - Output: {auth_url: string} (Google OAuth consent URL)
  - GET /google/callback
    - Input: Query params (code, state)
    - Output: Redirect to frontend with success/error status
- **Authentication Middleware**:
  - Clerk JWT validation dependency (get_current_user)
  - API key validation dependency (verify_api_key)
- **Error Responses** (consistent format):
  ```json
  {
    "detail": "Error message",
    "status_code": 401,
    "timestamp": "2026-02-17T14:30:00Z"
  }
  ```
- **API Documentation**: Automatic Swagger UI at http://localhost:8000/docs

## 5. Internal Responsibilities

1. **Project Structure Setup**:
   - Create FastAPI project directory structure
   - Initialize Python package with __init__.py files
   - Set up routers, models, services, middleware directories
   - Create main.py as application entry point

2. **Environment Configuration**:
   - Create config.py using Pydantic BaseSettings
   - Load environment variables from .env file
   - Validate required environment variables (SUPABASE_URL, SUPABASE_KEY, CLERK_SECRET_KEY, WHATSAPP_API_KEY)
   - Provide default values for optional settings
   - Create .env.example with placeholder values

3. **Supabase Database Connection**:
   - Install supabase-py library
   - Create database.py with Supabase client initialization
   - Configure connection pooling and timeout settings
   - Implement connection health check function
   - Handle connection errors gracefully

4. **Clerk JWT Verification**:
   - Install clerk-backend-api or use JWKS-based verification
   - Implement verify_clerk_jwt function:
     - Fetch Clerk JWKS (JSON Web Key Set) from Clerk's well-known endpoint
     - Verify JWT signature using Clerk's public keys
     - Validate token claims (exp, iss, azp)
     - Extract clerk_id (sub claim) from verified JWT
   - Cache JWKS keys for performance (refresh periodically)

5. **User Lookup Service**:
   - Implement get_or_create_user function:
     - Query users table by clerk_id
     - Return user if found
     - Return None if not found (user must be created via webhook first)
   - Implement get_user_by_clerk_id function (for JWT validation)
   - Handle database query errors

6. **Clerk Webhook Handler**:
   - POST /webhooks/clerk:
     - Verify webhook signature using CLERK_WEBHOOK_SECRET (svix library)
     - Parse webhook event type
     - Handle user.created: Insert new user into users table (clerk_id, email, username)
     - Handle user.updated: Update user email/username in users table
     - Handle user.deleted: Delete user from users table (CASCADE deletes all data)
     - Return 200 OK for successful processing
     - Return 400 for invalid signature
     - Log all webhook events

7. **Authentication Endpoints**:
   - GET /auth/me:
     - Validate Clerk JWT token (via dependency)
     - Fetch current user from database by clerk_id
     - Return user info (id, clerk_id, email, username, created_at)
     - Return 401 if token invalid
     - Return 404 if user not found in database

8. **Google OAuth Endpoints**:
   - GET /google/auth-url:
     - Validate Clerk JWT (user must be authenticated)
     - Generate Google OAuth consent URL with required scopes:
       - gmail.readonly, gmail.send, calendar.readonly, classroom.courses.readonly, classroom.coursework.me.readonly, classroom.announcements.readonly
     - Include state parameter (user_id for callback identification)
     - Return auth_url
   - GET /google/callback:
     - Exchange authorization code for access_token and refresh_token
     - Encrypt tokens using ENCRYPTION_KEY
     - Store in oauth_tokens table (provider='google', user_id from state)
     - Redirect to frontend with success/error status

9. **Authentication Dependencies**:
   - get_current_user dependency:
     - Extract Clerk JWT from Authorization header
     - Verify JWT using Clerk JWKS
     - Extract clerk_id from token
     - Fetch user from database by clerk_id
     - Raise 401 HTTPException if token invalid or user not found
     - Return User object for protected endpoints
   - verify_api_key dependency:
     - Extract API key from X-API-Key header
     - Compare with WHATSAPP_API_KEY environment variable
     - Raise 403 HTTPException if invalid
     - Used for WhatsApp service endpoints

10. **Error Handling Middleware**:
    - Global exception handler for HTTPException
    - Global exception handler for validation errors (422)
    - Global exception handler for uncaught exceptions (500)
    - Consistent error response format with timestamp
    - Log all errors with stack traces

11. **CORS Configuration**:
    - Install fastapi CORS middleware
    - Configure allowed origins from CORS_ORIGINS environment variable
    - Allow credentials (cookies, authorization headers)
    - Allow all HTTP methods (GET, POST, PUT, DELETE, PATCH, OPTIONS)
    - Allow all headers

12. **Dependencies Installation**:
    - Create requirements.txt:
      - fastapi
      - uvicorn[standard]
      - supabase-py
      - PyJWT or python-jose[cryptography] (for Clerk JWT verification)
      - svix (for Clerk webhook signature verification)
      - cryptography (for OAuth token encryption)
      - google-auth, google-auth-oauthlib (for Google OAuth)
      - python-dotenv
      - pydantic[email]
      - python-multipart
      - httpx (for async HTTP requests to Clerk JWKS)

## 6. Dependencies

**External Services:**
- Supabase PostgreSQL database (SUPABASE_URL and SUPABASE_KEY from core tables ticket)
- Clerk Auth (CLERK_SECRET_KEY from Clerk Dashboard)
- Google Cloud (GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET for OAuth)

**Libraries/Tools:**
- Python 3.10+
- FastAPI web framework
- Uvicorn ASGI server
- Supabase Python client
- PyJWT or python-jose for Clerk JWT verification
- svix for Clerk webhook verification
- cryptography for token encryption
- google-auth-oauthlib for Google OAuth
- python-dotenv for environment variables

**Ticket Dependencies:**
- Setup_Supabase_Project_&_Core_Tables (MUST complete first - provides users table and SUPABASE_URL/KEY)

**Blocks:**
- Conversation_Management_API (needs authentication middleware)
- Conversational_Agent_with_Gemini_Integration (needs authentication)
- Sync_Status_&_Data_Management_Endpoints (needs authentication)
- WhatsApp_Service_-_Hourly_Summarization_&_Backend_Integration (needs API key authentication)
- Next.js_Frontend_-_Authentication_&_Layout (needs Clerk JWT verification on backend)

## 7. Execution Model

**Type**: HTTP Service (Long-Running Web Server)

**Lifecycle**:
1. **Startup**:
   - Load environment variables from .env
   - Validate required configuration
   - Initialize Supabase client and test connection
   - Fetch Clerk JWKS for JWT verification
   - Mount routers (auth, webhooks, google_oauth, placeholder routes)
   - Configure CORS middleware
   - Configure error handling middleware
   - Start Uvicorn server on BACKEND_PORT

2. **Runtime**:
   - Handle HTTP requests asynchronously
   - Authenticate requests using Clerk JWT or API key
   - Route requests to appropriate handlers
   - Return JSON responses
   - Log all requests and errors

3. **Shutdown**:
   - Graceful shutdown on SIGTERM/SIGINT
   - Close database connections
   - Complete in-flight requests

**Concurrency**: Asynchronous (ASGI) with event loop, supports multiple concurrent requests

**Port**: 8000 (configurable via BACKEND_PORT)

**Deployment**: Runs in Docker container (Dockerfile created in Docker Compose ticket)

**Health Check**: GET /health endpoint returning {status: "ok", database: "connected"}

## 8. Failure Handling

**Database Connection Failures**:
- **Issue**: Unable to connect to Supabase on startup
- **Handling**:
  - Log detailed error message with SUPABASE_URL (redacted key)
  - Retry connection 3 times with exponential backoff (1s, 2s, 4s)
  - Exit with error code 1 if all retries fail
  - Display user-friendly error message

**Invalid Environment Variables**:
- **Issue**: Missing required environment variables (SUPABASE_URL, SUPABASE_KEY, CLERK_SECRET_KEY, WHATSAPP_API_KEY)
- **Handling**:
  - Validate on startup using Pydantic BaseSettings
  - Raise ValidationError with missing variable names
  - Exit with error code 1 and clear error message

**Clerk JWT Verification Failures**:
- **Expired Clerk JWT Token**:
  - Return 401 Unauthorized with error: "Token expired"
  - Frontend Clerk SDK should auto-refresh tokens
- **Invalid Clerk JWT Token**:
  - Return 401 Unauthorized with error: "Invalid token"
  - Log potential security issue
- **Clerk JWKS Fetch Failure**:
  - Use cached JWKS if available
  - Retry fetch with exponential backoff
  - Return 503 if unable to verify tokens

**Clerk Webhook Failures**:
- **Invalid Webhook Signature**:
  - Return 400 Bad Request
  - Log unauthorized webhook attempt with source IP
- **Database Error During User Sync**:
  - Return 500 Internal Server Error
  - Clerk will retry the webhook

**Google OAuth Failures**:
- **Invalid Authorization Code**:
  - Redirect to frontend with error status
  - Log error details
- **Token Encryption Failure**:
  - Log error, do not store unencrypted tokens
  - Return error to user

**API Key Authentication Failures**:
- **Invalid API Key** (WhatsApp service):
  - Return 403 Forbidden
  - Log unauthorized API key attempt with source IP

**Database Query Errors**:
- **Issue**: Query fails due to network issue or database error
- **Handling**:
  - Catch exception and log error
  - Return 500 Internal Server Error
  - Avoid exposing database error details to client
  - Retry transient errors (connection timeout)

**Uncaught Exceptions**:
- **Issue**: Unexpected error in application code
- **Handling**:
  - Global exception handler catches all uncaught exceptions
  - Log full stack trace for debugging
  - Return 500 Internal Server Error with generic message
  - Never expose stack traces to client in production

## 9. Observability

**Logging**:
- **Startup Events**:
  - Server started (timestamp, port, environment)
  - Database connection established
  - Clerk JWKS fetched successfully
  - Configuration loaded (redact secrets)
- **Request Logging**:
  - HTTP method, path, status code, response time
  - Client IP address (for security monitoring)
  - User clerk_id (for authenticated requests)
- **Authentication Events**:
  - Clerk JWT verification success (clerk_id, IP)
  - Clerk JWT verification failure (IP, reason)
  - Clerk webhook received (event type, clerk_id)
  - Google OAuth flow initiated (user_id)
  - Google OAuth callback success/failure
  - API key validation errors (IP, key excerpt)
- **Error Logging**:
  - All 4xx and 5xx responses with details
  - Database errors with query context
  - Uncaught exceptions with full stack trace

**Metrics** (if Prometheus integration added):
- http_requests_total (counter, labels: method, path, status)
- http_request_duration_seconds (histogram, labels: method, path)
- clerk_jwt_verifications_total (counter, labels: success, failure)
- clerk_webhooks_received_total (counter, labels: event_type)
- google_oauth_flows_total (counter, labels: success, failure)
- database_connections_active (gauge)
- api_key_requests_total (counter, labels: valid, invalid)

**Health Check**:
```json
GET /health
{
  "status": "ok",
  "database": "connected",
  "uptime_seconds": 12345,
  "version": "1.0.0"
}
```

**API Documentation**:
- Swagger UI at http://localhost:8000/docs
- ReDoc at http://localhost:8000/redoc
- OpenAPI schema at http://localhost:8000/openapi.json

## 10. Security Considerations

**Clerk Auth Security**:
- Clerk handles all password storage, hashing, and user authentication
- Backend only verifies Clerk-issued JWTs using public JWKS keys
- Never store passwords or authentication credentials in our database
- Clerk JWT verification uses RS256 algorithm with Clerk's public keys
- Webhook payloads verified using svix signature validation

**Google OAuth Token Security**:
- OAuth tokens (access_token, refresh_token) encrypted at rest using ENCRYPTION_KEY
- ENCRYPTION_KEY must be cryptographically random (256-bit minimum)
- Never store unencrypted OAuth tokens in database
- Never log OAuth tokens
- Tokens scoped to minimum required permissions (readonly access)

**API Key Security**:
- WHATSAPP_API_KEY stored in environment variable (never hardcoded)
- Use cryptographically random key (generate with: openssl rand -hex 32)
- Different API key per service if multiple external services added
- Log all API key validation failures for security monitoring

**CORS Security**:
- Restrict CORS_ORIGINS to known frontend domain (not wildcard * in production)
- Allow credentials only for trusted origins
- Reject requests from untrusted origins

**Database Security**:
- SUPABASE_URL and SUPABASE_KEY stored in environment variables
- Never log SUPABASE_KEY (contains credentials)
- Use connection pooling to prevent connection exhaustion attacks
- Parameterized queries prevent SQL injection (handled by Supabase client)

**Error Response Security**:
- Never expose internal error details (stack traces, database errors) to clients
- Use generic error messages for authentication failures
- Log detailed errors server-side for debugging

**Webhook Security**:
- Always verify Clerk webhook signatures before processing
- Never trust unverified webhook payloads
- Rate limit webhook endpoint if needed

**HTTPS Enforcement** (production deployment):
- All production traffic must use HTTPS
- Enable HSTS (HTTP Strict Transport Security) header

## 11. Scaling Considerations

**Current MVP Constraints**:
- Single user (no multi-tenancy)
- Single FastAPI instance (no load balancing required)
- Modest traffic (< 100 requests/minute)
- Synchronous database queries acceptable

**Performance Characteristics**:
- FastAPI async support enables high concurrency
- Database connection pooling reduces connection overhead
- Clerk JWT verification is fast (cached JWKS keys, < 1ms per request)
- No caching layer required for MVP

**Vertical Scaling**:
- Increase CPU for handling more concurrent requests
- Increase memory for larger connection pool
- FastAPI scales well to 1000+ requests/second on single instance

**Horizontal Scaling** (future):
- Stateless design enables easy horizontal scaling
- Add multiple FastAPI instances behind load balancer (Nginx, Traefik)
- Supabase handles database connection pooling
- Clerk JWT verification is stateless (JWKS cached per instance)

**Bottlenecks**:
- Database queries are primary bottleneck (mitigate with indexes and query optimization)
- Clerk JWKS fetch on cold start (mitigate with caching)
- No caching layer (add Redis cache for frequently accessed data if needed)

**Optimization Opportunities**:
- Add Redis cache for user lookups (reduce database queries)
- Implement connection pooling with PgBouncer if Supabase connection limit reached
- Add CDN for serving static API documentation
- Implement async database queries using asyncpg

## 12. Future Extensions

**Phase 2 Enhancements**:
- **Multi-User Support**:
  - Clerk handles multi-user authentication out of the box
  - Add organization support via Clerk Organizations
  - User profile management via Clerk webhooks
- **Advanced Authentication**:
  - Two-factor authentication (configured in Clerk Dashboard)
  - Additional social login providers (GitHub, Microsoft)
  - Clerk session management and device tracking
- **Rate Limiting**:
  - Per-user rate limits
  - Per-IP rate limits
  - Adaptive rate limiting (increase limits for trusted users)
- **Audit Logging**:
  - Log all authentication events to audit table
  - Track user actions (create, update, delete)

**Advanced Features**:
- **Role-Based Access Control (RBAC)**:
  - Clerk metadata for user roles (admin, user, readonly)
  - Permission system for endpoints
- **API Versioning**:
  - Version endpoints (/v1/auth/me, /v2/auth/me)
  - Deprecation warnings for old API versions
- **WebSocket Support**:
  - Real-time updates for conversations
  - Live sync status notifications

**Security Improvements**:
- **Secrets Management**:
  - Integrate with HashiCorp Vault or AWS Secrets Manager
  - Rotate ENCRYPTION_KEY periodically
  - Rotate WHATSAPP_API_KEY periodically
- **Security Headers**:
  - Content Security Policy (CSP)
  - X-Frame-Options (prevent clickjacking)
  - X-Content-Type-Options (prevent MIME sniffing)

**Monitoring and Alerting**:
- **Observability Platform**:
  - Integrate with Datadog, New Relic, or Prometheus
  - Real-time dashboards for request metrics
  - Alerting on high error rates, slow responses
- **Distributed Tracing**:
  - OpenTelemetry integration
  - Trace requests across services (FastAPI → Supabase → Celery)
- **Log Aggregation**:
  - Centralized logging with ELK stack or Loki

**Developer Experience**:
- **Testing**:
  - Unit tests for Clerk JWT verification logic
  - Integration tests for endpoints with mock Clerk tokens
  - Test fixtures for database setup
- **Documentation**:
  - Detailed API documentation with examples
  - Developer guide for adding new endpoints
  - Architecture decision records (ADRs)
- **Code Quality**:
  - Type hints for all functions
  - Linting with flake8 or ruff
  - Code formatting with black
