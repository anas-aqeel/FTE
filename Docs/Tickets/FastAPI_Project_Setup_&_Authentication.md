# FastAPI Project Setup & Authentication (Clerk Auth)

## Objective

Set up FastAPI project structure with **Clerk Auth** authentication integration and Supabase database connection.

## Scope

**In Scope:**
- FastAPI project initialization
- Project structure (routers, models, services, config)
- Environment configuration (.env support for DATABASE_URL, CLERK_SECRET_KEY, GEMINI_API_KEY, WHATSAPP_API_KEY)
- Supabase client setup
- **Clerk Auth integration:**
  - Clerk SDK setup
  - JWT token verification middleware (`get_current_user` dependency)
  - User sync from Clerk to database
  - Clerk webhook handler for user events (create/update/delete)
- Authentication endpoints:
  - `GET /auth/me` - get current user (verifies Clerk JWT)
  - `POST /webhooks/clerk` - sync user events from Clerk
- API key authentication middleware (for WhatsApp service)
- Error handling middleware
- CORS configuration
- API documentation (automatic via FastAPI)

**Out of Scope:**
- ~~Username/password login~~ (handled by Clerk)
- ~~JWT token generation~~ (handled by Clerk)
- ~~Password hashing~~ (handled by Clerk)
- Conversation endpoints
- Chat/conversational agent logic
- Data ingestion endpoints

**Note:** User authentication is entirely handled by Clerk. This ticket focuses on integrating Clerk's JWT verification into FastAPI.

## Spec References

- `spec:69889a16-05f6-4f4a-bf1a-8cf0daec03b5/4eb58cf0-bbfc-42e3-acd5-9fe60dd4f30b` (Tech Plan - FastAPI Backend)
- `docs/specs/OAuth_Setup_Guide.md` (Part 1: Clerk Auth Setup)

## Acceptance Criteria

- [ ] FastAPI app runs and serves docs at `/docs`
- [ ] Supabase connection works
- [ ] Clerk SDK installed and configured
- [ ] `get_current_user()` dependency verifies Clerk JWT tokens
- [ ] Protected endpoints require valid Clerk JWT
- [ ] `/auth/me` returns current user info from database
- [ ] Clerk webhook endpoint receives and processes user events
- [ ] New users automatically synced to database on first login
- [ ] User updates synced from Clerk webhooks
- [ ] User deletions cascade delete all user data
- [ ] Environment variables loaded from .env
- [ ] Error responses are consistent
- [ ] CORS configured for frontend domain

## Dependencies

- Database schema complete (all 3 database tickets) with `clerk_id` column
- Clerk account created and application configured

---

## Implementation Details

### 1. Install Dependencies

```bash
pip install fastapi uvicorn supabase clerk-backend-api svix python-dotenv
```

### 2. Project Structure

```
backend/
├── app/
│   ├── main.py
│   ├── config.py
│   ├── auth/
│   │   ├── __init__.py
│   │   ├── clerk_middleware.py      # Clerk JWT verification
│   │   └── whatsapp_auth.py         # API key auth for WhatsApp
│   ├── routers/
│   │   ├── __init__.py
│   │   ├── auth.py                  # /auth/me endpoint
│   │   └── webhooks.py              # /webhooks/clerk
│   ├── models/
│   │   └── __init__.py
│   └── services/
│       └── supabase_client.py
├── .env
└── requirements.txt
```

### 3. Environment Variables

```bash
# Clerk Authentication
CLERK_SECRET_KEY=sk_test_xxxxxxxxxxxxxxxxxxxx
CLERK_WEBHOOK_SECRET=whsec_xxxxxxxxxxxx

# Database
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_KEY=your-supabase-key

# Google OAuth (for API access - separate ticket)
GOOGLE_CLIENT_ID=your-client-id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=GOCSPX-xxxxxxxxxxxxx

# Application
FRONTEND_URL=http://localhost:3000
CORS_ORIGINS=http://localhost:3000,https://your-domain.com

# APIs
GEMINI_API_KEY=your-gemini-api-key
WHATSAPP_API_KEY=shared-secret-key

# Encryption
ENCRYPTION_KEY=<32-byte-base64-key>  # For encrypting OAuth tokens
```

### 4. Clerk Authentication Middleware

**File:** `backend/app/auth/clerk_middleware.py`

```python
from fastapi import HTTPException, Security, Depends
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from clerk_backend_api import Clerk
from typing import Optional
import os

from app.services.supabase_client import get_supabase

# Initialize Clerk client
clerk = Clerk(bearer_auth=os.getenv('CLERK_SECRET_KEY'))
security = HTTPBearer()


async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Security(security)
):
    """
    Verify Clerk JWT token and return user from database.
    Use this as a dependency in protected routes.

    Example:
        @router.get("/conversations")
        async def list_conversations(user = Depends(get_current_user)):
            return {"user_id": user["id"]}
    """
    try:
        # Extract and verify token
        token = credentials.credentials
        session = clerk.sessions.verify_token(token)

        # Get Clerk user ID from session
        clerk_user_id = session['sub']

        # Get or create user in our database
        supabase = get_supabase()
        user = await get_or_create_user_from_clerk(clerk_user_id, supabase)

        return user

    except Exception as e:
        raise HTTPException(
            status_code=401,
            detail=f"Invalid authentication credentials: {str(e)}"
        )


async def get_or_create_user_from_clerk(clerk_user_id: str, supabase):
    """
    Fetch user from database or create if doesn't exist.
    Syncs user data from Clerk on first access.
    """
    # Check if user exists in our database
    result = supabase.table('users') \
        .select('*') \
        .eq('clerk_id', clerk_user_id) \
        .execute()

    if result.data:
        return result.data[0]

    # User doesn't exist - fetch from Clerk and create
    try:
        clerk_user = clerk.users.get(clerk_user_id)
    except Exception as e:
        raise HTTPException(
            status_code=404,
            detail=f"Clerk user not found: {e}"
        )

    # Create user in database
    new_user = {
        'clerk_id': clerk_user.id,
        'email': clerk_user.email_addresses[0].email_address if clerk_user.email_addresses else None,
        'username': clerk_user.username or clerk_user.email_addresses[0].email_address.split('@')[0],
        'created_at': clerk_user.created_at,
        'updated_at': clerk_user.updated_at
    }

    result = supabase.table('users').insert(new_user).execute()

    if not result.data:
        raise HTTPException(status_code=500, detail="Failed to create user")

    return result.data[0]


def get_optional_user(
    credentials: Optional[HTTPAuthorizationCredentials] = Security(security, auto_error=False)
):
    """
    Optional authentication - returns None if no token provided.
    Use for endpoints that work both authenticated and unauthenticated.
    """
    if not credentials:
        return None

    return get_current_user(credentials)
```

### 5. Authentication Router

**File:** `backend/app/routers/auth.py`

```python
from fastapi import APIRouter, Depends

from app.auth.clerk_middleware import get_current_user

router = APIRouter(prefix="/auth", tags=["Authentication"])


@router.get("/me")
async def get_current_user_info(user = Depends(get_current_user)):
    """
    Get current authenticated user.
    Requires Clerk JWT token in Authorization header.
    """
    return {
        "id": user["id"],
        "clerk_id": user["clerk_id"],
        "email": user["email"],
        "username": user["username"],
        "created_at": user["created_at"]
    }
```

### 6. Clerk Webhook Handler

**File:** `backend/app/routers/webhooks.py`

```python
from fastapi import APIRouter, Request, HTTPException
from svix.webhooks import Webhook
import os

from app.services.supabase_client import get_supabase

router = APIRouter(prefix="/webhooks", tags=["Webhooks"])


@router.post("/clerk")
async def handle_clerk_webhook(request: Request):
    """
    Handle Clerk user events (create, update, delete).
    Syncs Clerk users to our database in real-time.

    Setup in Clerk Dashboard:
    1. Go to Webhooks section
    2. Add endpoint: https://your-backend.com/webhooks/clerk
    3. Subscribe to: user.created, user.updated, user.deleted
    """
    # Get webhook headers
    svix_id = request.headers.get("svix-id")
    svix_timestamp = request.headers.get("svix-timestamp")
    svix_signature = request.headers.get("svix-signature")

    if not all([svix_id, svix_timestamp, svix_signature]):
        raise HTTPException(status_code=400, detail="Missing webhook headers")

    # Verify webhook signature
    webhook = Webhook(os.getenv('CLERK_WEBHOOK_SECRET'))

    try:
        payload = await request.body()
        event = webhook.verify(payload.decode(), {
            "svix-id": svix_id,
            "svix-timestamp": svix_timestamp,
            "svix-signature": svix_signature
        })
    except Exception as e:
        raise HTTPException(status_code=400, detail=f"Invalid webhook signature: {e}")

    # Process event
    event_type = event["type"]
    user_data = event["data"]

    supabase = get_supabase()

    if event_type == "user.created":
        # Create user in database
        await handle_user_created(user_data, supabase)

    elif event_type == "user.updated":
        # Update user in database
        await handle_user_updated(user_data, supabase)

    elif event_type == "user.deleted":
        # Delete user (CASCADE deletes all their data)
        await handle_user_deleted(user_data, supabase)

    return {"status": "success", "event": event_type}


async def handle_user_created(user_data: dict, supabase):
    """Create user in database when created in Clerk"""
    supabase.table('users').insert({
        'clerk_id': user_data['id'],
        'email': user_data['email_addresses'][0]['email_address'] if user_data['email_addresses'] else None,
        'username': user_data.get('username') or user_data['email_addresses'][0]['email_address'].split('@')[0],
        'created_at': user_data['created_at'],
        'updated_at': user_data['updated_at']
    }).execute()


async def handle_user_updated(user_data: dict, supabase):
    """Update user when updated in Clerk"""
    supabase.table('users').update({
        'email': user_data['email_addresses'][0]['email_address'] if user_data['email_addresses'] else None,
        'username': user_data.get('username'),
        'updated_at': user_data['updated_at']
    }).eq('clerk_id', user_data['id']).execute()


async def handle_user_deleted(user_data: dict, supabase):
    """Delete user when deleted in Clerk (CASCADE deletes all data)"""
    supabase.table('users').delete().eq('clerk_id', user_data['id']).execute()
```

### 7. Supabase Client

**File:** `backend/app/services/supabase_client.py`

```python
from supabase import create_client, Client
import os

_supabase_client: Client = None


def get_supabase() -> Client:
    """Get singleton Supabase client"""
    global _supabase_client

    if _supabase_client is None:
        _supabase_client = create_client(
            os.getenv('SUPABASE_URL'),
            os.getenv('SUPABASE_KEY')
        )

    return _supabase_client
```

### 8. Main Application

**File:** `backend/app/main.py`

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
import os

from app.routers import auth, webhooks

app = FastAPI(
    title="AcademiQ API",
    version="1.0.0",
    description="Backend API for AcademiQ with Clerk Auth"
)

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=os.getenv('CORS_ORIGINS', 'http://localhost:3000').split(','),
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Routers
app.include_router(auth.router)
app.include_router(webhooks.router)

@app.get("/")
async def root():
    return {"message": "AcademiQ API with Clerk Auth"}


@app.get("/health")
async def health_check():
    return {"status": "healthy"}
```

### 9. Run Application

```bash
# Development
uvicorn app.main:app --reload --port 8000

# Production
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

---

## Testing

### 1. Test Clerk JWT Verification

```bash
# Get JWT token from Clerk (via frontend or Clerk API)
CLERK_JWT="eyJhbGc..."

# Test /auth/me endpoint
curl -H "Authorization: Bearer $CLERK_JWT" http://localhost:8000/auth/me
```

### 2. Test Webhook Handler (Local Testing)

```bash
# Use Clerk Dashboard to send test webhook
# Or use svix CLI for local testing

# Expected response:
{"status": "success", "event": "user.created"}
```

### 3. Test Protected Endpoint

```python
# Example protected route
from fastapi import APIRouter, Depends
from app.auth.clerk_middleware import get_current_user

router = APIRouter()

@router.get("/test-protected")
async def test_protected(user = Depends(get_current_user)):
    return {"message": f"Hello {user['username']}!"}
```

---

## Notes

- **No user seeding needed:** Users are created when they sign up via Clerk
- **JWT verification:** Clerk SDKhandles all token validation
- **User sync:** Webhooks keep our database in sync with Clerk
- **Security:** Clerk manages password hashing, MFA, session security
- **Scalability:** Clerk handles authentication infrastructure

---

## Dependencies

- Database schema complete with `clerk_id` column (instead of `password_hash`)
- Clerk application created in Clerk Dashboard
