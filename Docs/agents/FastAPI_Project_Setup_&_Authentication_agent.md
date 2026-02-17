# Agent Specification: FastAPI Project Setup & Authentication

## 1. Purpose

Initialize a production-ready FastAPI backend project with comprehensive authentication system supporting JWT-based user login and API key authentication for external services, serving as the central API gateway for the Personal Assistant Agent system.

## 2. Scope

**In Scope:**
- FastAPI project initialization with proper structure (routers, models, services, config)
- Environment configuration (.env support for DATABASE_URL, JWT_SECRET, GEMINI_API_KEY, WHATSAPP_API_KEY)
- Supabase PostgreSQL client setup with connection pooling
- Authentication endpoints:
  - `POST /auth/login` - username/password login, returns JWT
  - `POST /auth/logout` - logout endpoint
  - `GET /auth/me` - get current authenticated user
- JWT token generation and validation middleware
- API key authentication middleware (for WhatsApp service to call backend)
- Password hashing with bcrypt
- Error handling middleware with consistent error responses
- CORS configuration for Next.js frontend
- Automatic API documentation via FastAPI (Swagger UI at `/docs`)
- Initial user seeding (via migration or management command)
- Dependencies installation and requirements.txt

**Out of Scope:**
- Conversation endpoints (separate ticket)
- Chat/conversational agent logic (separate ticket)
- Data ingestion endpoints (separate tickets)
- Celery worker setup (separate ticket)
- Frontend implementation (separate tickets)

## 3. Inputs

- **Environment Variables** (from .env file):
  - DATABASE_URL: Supabase PostgreSQL connection string
  - JWT_SECRET: Secret key for signing JWT tokens (generate random 256-bit key)
  - JWT_ALGORITHM: Algorithm for JWT (default: HS256)
  - JWT_EXPIRATION_HOURS: Token expiration time (default: 24)
  - GEMINI_API_KEY: Google Gemini API key for AI processing
  - WHATSAPP_API_KEY: Shared secret for authenticating WhatsApp service requests
  - CORS_ORIGINS: Comma-separated list of allowed origins (default: http://localhost:3000)
  - BACKEND_PORT: Port for FastAPI server (default: 8000)
- **Login Request** (POST /auth/login):
  - username: String (required)
  - password: String (required, plaintext)
- **JWT Token** (for protected endpoints):
  - Authorization header: Bearer {token}
- **API Key** (for WhatsApp service endpoints):
  - X-API-Key header: {WHATSAPP_API_KEY}
- **Database Schema**: Existing users table from core tables ticket

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
    │   │   └── auth.py (authentication endpoints)
    │   ├── services/
    │   │   ├── __init__.py
    │   │   └── auth_service.py (authentication logic)
    │   └── middleware/
    │       ├── __init__.py
    │       └── error_handler.py
    ├── requirements.txt
    ├── .env.example
    └── Dockerfile (placeholder for Docker ticket)
    ```
- **Authentication Endpoints**:
  - POST /auth/login
    - Input: {username: string, password: string}
    - Output: {access_token: string, token_type: "bearer", user: {id, username}}
  - POST /auth/logout
    - Input: JWT token (header)
    - Output: {message: "Logged out successfully"}
  - GET /auth/me
    - Input: JWT token (header)
    - Output: {id: UUID, username: string, created_at: timestamp}
- **Authentication Middleware**:
  - JWT validation dependency (get_current_user)
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
- **Initial User**: Seeded user account for testing (username: admin, password: changeme)

## 5. Internal Responsibilities

1. **Project Structure Setup**:
   - Create FastAPI project directory structure
   - Initialize Python package with __init__.py files
   - Set up routers, models, services, middleware directories
   - Create main.py as application entry point

2. **Environment Configuration**:
   - Create config.py using Pydantic BaseSettings
   - Load environment variables from .env file
   - Validate required environment variables (DATABASE_URL, JWT_SECRET, WHATSAPP_API_KEY)
   - Provide default values for optional settings
   - Create .env.example with placeholder values

3. **Supabase Database Connection**:
   - Install supabase-py library
   - Create database.py with Supabase client initialization
   - Configure connection pooling and timeout settings
   - Implement connection health check function
   - Handle connection errors gracefully

4. **Password Hashing**:
   - Install bcrypt library
   - Implement password hashing function (hash_password)
   - Implement password verification function (verify_password)
   - Use bcrypt with appropriate cost factor (rounds=12)

5. **JWT Token Management**:
   - Install python-jose[cryptography] library
   - Implement JWT token creation function (create_access_token)
   - Implement JWT token validation function (verify_token)
   - Include user_id and username in JWT payload
   - Set token expiration using JWT_EXPIRATION_HOURS
   - Handle token expiration and invalid token errors

6. **Authentication Service**:
   - Implement authenticate_user function:
     - Query users table by username
     - Verify password hash
     - Return user object if valid, None otherwise
   - Implement get_user_by_id function (for JWT validation)
   - Handle database query errors

7. **Authentication Endpoints**:
   - POST /auth/login:
     - Validate username and password (required fields)
     - Call authenticate_user
     - Return 401 if credentials invalid
     - Generate JWT token if valid
     - Return access_token and user info
   - POST /auth/logout:
     - Validate JWT token (via dependency)
     - Return success message (stateless logout, token invalidated on client)
   - GET /auth/me:
     - Validate JWT token
     - Fetch current user from database
     - Return user info

8. **Authentication Dependencies**:
   - get_current_user dependency:
     - Extract JWT token from Authorization header
     - Validate token
     - Fetch user from database
     - Raise 401 HTTPException if invalid
     - Return User object for protected endpoints
   - verify_api_key dependency:
     - Extract API key from X-API-Key header
     - Compare with WHATSAPP_API_KEY environment variable
     - Raise 403 HTTPException if invalid
     - Used for WhatsApp service endpoints

9. **Error Handling Middleware**:
   - Global exception handler for HTTPException
   - Global exception handler for validation errors (422)
   - Global exception handler for uncaught exceptions (500)
   - Consistent error response format with timestamp
   - Log all errors with stack traces

10. **CORS Configuration**:
    - Install fastapi CORS middleware
    - Configure allowed origins from CORS_ORIGINS environment variable
    - Allow credentials (cookies, authorization headers)
    - Allow all HTTP methods (GET, POST, PUT, DELETE, PATCH, OPTIONS)
    - Allow all headers

11. **Initial User Seeding**:
    - Create management command or migration script
    - Check if users table is empty
    - Insert initial user: username=admin, password=changeme (hashed)
    - Log seeding status

12. **Dependencies Installation**:
    - Create requirements.txt:
      - fastapi
      - uvicorn[standard]
      - supabase-py
      - python-jose[cryptography]
      - bcrypt
      - python-dotenv
      - pydantic[email]
      - python-multipart

## 6. Dependencies

**External Services:**
- Supabase PostgreSQL database (DATABASE_URL from core tables ticket)

**Libraries/Tools:**
- Python 3.10+
- FastAPI web framework
- Uvicorn ASGI server
- Supabase Python client
- python-jose for JWT
- bcrypt for password hashing
- python-dotenv for environment variables

**Ticket Dependencies:**
- Setup_Supabase_Project_&_Core_Tables (MUST complete first - provides users table and DATABASE_URL)

**Blocks:**
- Conversation_Management_API (needs authentication middleware)
- Conversational_Agent_with_Gemini_Integration (needs authentication)
- Sync_Status_&_Data_Management_Endpoints (needs authentication)
- WhatsApp_Service_-_Hourly_Summarization_&_Backend_Integration (needs API key authentication)
- Next.js_Frontend_-_Authentication_&_Layout (needs login endpoints)

## 7. Execution Model

**Type**: HTTP Service (Long-Running Web Server)

**Lifecycle**:
1. **Startup**:
   - Load environment variables from .env
   - Validate required configuration
   - Initialize Supabase client and test connection
   - Mount routers (auth, placeholder routes)
   - Configure CORS middleware
   - Configure error handling middleware
   - Seed initial user if users table empty
   - Start Uvicorn server on BACKEND_PORT

2. **Runtime**:
   - Handle HTTP requests asynchronously
   - Authenticate requests using JWT or API key
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
  - Log detailed error message with DATABASE_URL (redacted password)
  - Retry connection 3 times with exponential backoff (1s, 2s, 4s)
  - Exit with error code 1 if all retries fail
  - Display user-friendly error message

**Invalid Environment Variables**:
- **Issue**: Missing required environment variables (DATABASE_URL, JWT_SECRET, WHATSAPP_API_KEY)
- **Handling**:
  - Validate on startup using Pydantic BaseSettings
  - Raise ValidationError with missing variable names
  - Exit with error code 1 and clear error message

**Authentication Failures**:
- **Invalid Credentials** (POST /auth/login):
  - Return 401 Unauthorized
  - Generic error message: "Invalid username or password" (avoid revealing which is wrong)
  - Rate limit login attempts (future enhancement)
- **Expired JWT Token**:
  - Return 401 Unauthorized with error: "Token expired"
  - Frontend should redirect to login
- **Invalid JWT Token**:
  - Return 401 Unauthorized with error: "Invalid token"
  - Log potential security issue
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
  - Initial user seeded (if applicable)
  - Configuration loaded (redact secrets)
- **Request Logging**:
  - HTTP method, path, status code, response time
  - Client IP address (for security monitoring)
  - User ID (for authenticated requests)
- **Authentication Events**:
  - Login success (username, IP)
  - Login failure (username, IP, reason)
  - Token validation errors (IP, token excerpt)
  - API key validation errors (IP, key excerpt)
- **Error Logging**:
  - All 4xx and 5xx responses with details
  - Database errors with query context
  - Uncaught exceptions with full stack trace

**Metrics** (if Prometheus integration added):
- http_requests_total (counter, labels: method, path, status)
- http_request_duration_seconds (histogram, labels: method, path)
- auth_login_attempts_total (counter, labels: success, failure)
- database_connections_active (gauge)
- jwt_tokens_issued_total (counter)
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

**Password Security**:
- Passwords hashed with bcrypt (cost factor 12)
- Never store plaintext passwords
- Never log passwords (even hashed)
- Password validation (minimum length 8 characters) enforced at application layer

**JWT Token Security**:
- JWT_SECRET must be cryptographically random (256-bit minimum)
- Never commit JWT_SECRET to Git (.env in .gitignore)
- Tokens expire after JWT_EXPIRATION_HOURS (default 24 hours)
- Include token issued timestamp (iat) and expiration (exp) claims
- Validate token signature on every request
- No token revocation in MVP (stateless design); logout only clears client-side token

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
- DATABASE_URL stored in environment variable
- Never log DATABASE_URL (contains credentials)
- Use connection pooling to prevent connection exhaustion attacks
- Parameterized queries prevent SQL injection (handled by Supabase client)

**Error Response Security**:
- Never expose internal error details (stack traces, database errors) to clients
- Use generic error messages for authentication failures ("Invalid credentials")
- Log detailed errors server-side for debugging

**Rate Limiting** (future enhancement for production):
- Implement rate limiting on /auth/login endpoint (prevent brute force)
- Consider using slowapi or fastapi-limiter

**HTTPS Enforcement** (production deployment):
- All production traffic must use HTTPS
- Set Secure flag on cookies (if using cookie-based auth)
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
- JWT validation is CPU-bound but fast (< 1ms per request)
- No caching layer required for MVP

**Vertical Scaling**:
- Increase CPU for handling more concurrent requests
- Increase memory for larger connection pool
- FastAPI scales well to 1000+ requests/second on single instance

**Horizontal Scaling** (future):
- Stateless design enables easy horizontal scaling
- Add multiple FastAPI instances behind load balancer (Nginx, Traefik)
- Supabase handles database connection pooling
- No session storage required (JWT tokens are stateless)

**Bottlenecks**:
- Database queries are primary bottleneck (mitigate with indexes and query optimization)
- Password hashing on login is CPU-intensive (acceptable for single-user MVP)
- No caching layer (add Redis cache for frequently accessed data if needed)

**Optimization Opportunities**:
- Add Redis cache for user lookups (reduce database queries)
- Implement connection pooling with PgBouncer if Supabase connection limit reached
- Add CDN for serving static API documentation
- Implement async database queries using asyncpg (Supabase client is synchronous)

## 12. Future Extensions

**Phase 2 Enhancements**:
- **Multi-User Support**:
  - User registration endpoint (POST /auth/register)
  - Email verification flow
  - Password reset functionality
  - User profile management
- **Advanced Authentication**:
  - OAuth 2.0 login (Google, GitHub)
  - Two-factor authentication (TOTP)
  - Session management with refresh tokens
  - Token revocation (blacklist)
- **Rate Limiting**:
  - Per-user rate limits
  - Per-IP rate limits
  - Adaptive rate limiting (increase limits for trusted users)
- **Audit Logging**:
  - Log all authentication events to audit table
  - Track user actions (create, update, delete)
  - Compliance reporting (GDPR, SOC2)

**Advanced Features**:
- **Role-Based Access Control (RBAC)**:
  - User roles (admin, user, readonly)
  - Permission system for endpoints
  - Fine-grained access control
- **API Versioning**:
  - Version endpoints (/v1/auth/login, /v2/auth/login)
  - Deprecation warnings for old API versions
- **WebSocket Support**:
  - Real-time updates for conversations
  - Live sync status notifications
- **GraphQL API**:
  - Alternative to REST API for complex queries
  - Reduce over-fetching of data

**Security Improvements**:
- **Advanced Threat Protection**:
  - IP blocking for repeated failed login attempts
  - Anomaly detection (unusual login patterns)
  - CAPTCHA for suspicious login attempts
- **Secrets Management**:
  - Integrate with HashiCorp Vault or AWS Secrets Manager
  - Rotate JWT_SECRET periodically
  - Encrypt sensitive environment variables
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
  - Full-text search across logs
  - Log retention policies

**Developer Experience**:
- **Testing**:
  - Unit tests for authentication logic
  - Integration tests for endpoints
  - Test fixtures for database setup
- **Documentation**:
  - Detailed API documentation with examples
  - Developer guide for adding new endpoints
  - Architecture decision records (ADRs)
- **Code Quality**:
  - Type hints for all functions
  - Linting with flake8 or ruff
  - Code formatting with black
