# OAuth Setup Guide - Google APIs with Clerk Auth

**Last Updated:** 2026-02-18
**Authentication Provider:** Clerk Auth

---

## Overview

This system uses **two separate authentication mechanisms**:

1. **User Authentication (Clerk Auth):**
   - Handles user login/signup/sessions
   - Manages user identity
   - Provides JWT tokens for API authorization
   - Supports social login (Google, GitHub, etc.) and email/password

2. **Google API Access (Google OAuth):**
   - Separate from user authentication
   - Required to access Gmail, Calendar, and Classroom APIs
   - Grants permission to read/send emails and access classroom data
   - Tokens stored per-user in database

---

## Architecture: Clerk Auth + Google OAuth

```
┌─────────────────────────────────────────────────────┐
│                   User Flow                          │
└─────────────────────────────────────────────────────┘

1. User signs up/logs in via Clerk (Google social login or email/password)
   ↓
2. Clerk provides JWT token for API authorization
   ↓
3. User connects Google account (OAuth) to grant API access
   ↓
4. Backend stores Google OAuth tokens in database
   ↓
5. Backend uses Google tokens to fetch emails/assignments
```

**Key Point:** Clerk handles WHO the user is. Google OAuth grants WHAT data we can access.

---

## Part 1: Clerk Auth Setup

### Prerequisites
- Clerk account created at https://clerk.com
- Application created in Clerk dashboard

### Step 1: Create Clerk Application

1. Go to https://dashboard.clerk.com
2. Click **"Create Application"**
3. Application name: **"Personal Assistant"**
4. Enable sign-in options:
   - ✅ **Google** (recommended for students)
   - ✅ **Email** (fallback option)
5. Click **"Create Application"**

### Step 2: Configure Clerk Settings

**Authentication:**
- **Social Login:** Enable Google OAuth
- **Email/Password:** Enable as fallback
- **Email Verification:** Required
- **Multi-factor:** Optional (recommended)

**Session Settings:**
- Session timeout: 7 days
- Multi-session: Allow (user can be logged in on multiple devices)

### Step 3: Get Clerk API Keys

From Clerk Dashboard:

```
API Keys (for backend):
- CLERK_SECRET_KEY=sk_live_... (or sk_test_... for development)

Frontend Keys:
- NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_live_... (or pk_test_...)
```

Add to `.env`:
```bash
# Clerk Authentication
CLERK_SECRET_KEY=sk_test_xxxxxxxxxxxxxxxxxxxx
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_xxxxxxxxxxxxxxxxxxxx
CLERK_WEBHOOK_SECRET=whsec_xxxxxxxxxxxx  # For user sync webhooks
```

### Step 4: Install Clerk SDKs

**Backend (FastAPI):**
```bash
pip install clerk-backend-api
```

**Frontend (Next.js):**
```bash
npm install @clerk/nextjs
```

### Step 5: Backend Integration

**File:** `backend/app/auth/clerk_middleware.py`

```python
from fastapi import HTTPException, Security
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from clerk_backend_api import Clerk
import os

clerk = Clerk(bearer_auth=os.getenv('CLERK_SECRET_KEY'))
security = HTTPBearer()


async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Security(security)
):
    """
    Verify Clerk JWT token and return user.
    Use this as a dependency in protected routes.
    """
    try:
        # Verify JWT token with Clerk
        token = credentials.credentials
        session = clerk.sessions.verify_token(token)

        # Get user from Clerk
        user_id = session['sub']  # Clerk user ID
        clerk_user = clerk.users.get(user_id)

        # Get or create user in our database
        db_user = await get_or_create_user_from_clerk(clerk_user)

        return db_user

    except Exception as e:
        raise HTTPException(
            status_code=401,
            detail=f"Invalid authentication credentials: {e}"
        )


async def get_or_create_user_from_clerk(clerk_user):
    """Sync Clerk user to our database"""
    from supabase import create_client

    supabase = create_client(
        os.getenv('SUPABASE_URL'),
        os.getenv('SUPABASE_KEY')
    )

    # Check if user exists
    result = supabase.table('users').select('*').eq('clerk_id', clerk_user.id).execute()

    if result.data:
        return result.data[0]

    # Create new user
    new_user = {
        'clerk_id': clerk_user.id,
        'email': clerk_user.email_addresses[0].email_address,
        'username': clerk_user.username or clerk_user.email_addresses[0].email_address,
        'created_at': clerk_user.created_at,
        'updated_at': clerk_user.updated_at
    }

    result = supabase.table('users').insert(new_user).execute()
    return result.data[0]
```

**Protected Route Example:**
```python
from fastapi import APIRouter, Depends

router = APIRouter()

@router.get("/conversations")
async def list_conversations(current_user = Depends(get_current_user)):
    """Only authenticated users can access this"""
    conversations = supabase.table('conversations') \
        .select('*') \
        .eq('user_id', current_user['id']) \
        .execute()
    return conversations.data
```

### Step 6: Frontend Integration

**File:** `frontend/app/layout.tsx`

```typescript
import { ClerkProvider } from '@clerk/nextjs'

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <ClerkProvider>
      <html lang="en">
        <body>{children}</body>
      </html>
    </ClerkProvider>
  )
}
```

**Protected Page:**
```typescript
import { auth } from '@clerk/nextjs/server'
import { redirect } from 'next/navigation'

export default async function DashboardPage() {
  const { userId } = await auth()

  if (!userId) {
    redirect('/sign-in')
  }

  return <div>Dashboard content</div>
}
```

**Sign In Page:**
```typescript
import { SignIn } from '@clerk/nextjs'

export default function SignInPage() {
  return (
    <div className="flex items-center justify-center min-h-screen">
      <SignIn
        appearance={{
          elements: {
            rootBox: "mx-auto",
            card: "shadow-lg"
          }
        }}
        routing="path"
        path="/sign-in"
        signUpUrl="/sign-up"
      />
    </div>
  )
}
```

---

## Part 2: Google OAuth for API Access

**Important:** This is SEPARATE from Clerk authentication. Google OAuth here is only for accessing Gmail/Classroom APIs.

### Prerequisites
- User already authenticated via Clerk
- Google Cloud Console project created
- Billing enabled (required for API access)

### Step 1: Create OAuth 2.0 Credentials

1. Go to https://console.cloud.google.com
2. Select your project (or create: "Personal Assistant Backend")
3. Navigate to **"APIs & Services"** > **"Credentials"**
4. Click **"Create Credentials"** > **"OAuth 2.0 Client ID"**
5. Application type: **"Web application"**
6. Name: **"Personal Assistant API Access"**
7. **Authorized redirect URIs:**
   - `http://localhost:3000/api/auth/google/callback` (development)
   - `https://your-domain.com/api/auth/google/callback` (production)
8. Click **"Create"**
9. **Download** credentials JSON file

### Step 2: Enable Required APIs

1. Go to **"APIs & Services"** > **"Library"**
2. Search and enable:
   - ✅ **Gmail API**
   - ✅ **Google Calendar API**
   - ✅ **Google Classroom API**
3. Wait 1-2 minutes for propagation

### Step 3: Configure OAuth Consent Screen

1. Go to **"APIs & Services"** > **"OAuth consent screen"**
2. **User Type:** External
3. **App Information:**
   - App name: **"Personal Assistant"**
   - Support email: Your email
   - Developer contact: Your email
4. Click **"Save and Continue"**

**Scopes (Step 2):**

Add these scopes:

```
Gmail & Calendar:
✅ https://www.googleapis.com/auth/gmail.readonly
✅ https://www.googleapis.com/auth/gmail.send
✅ https://www.googleapis.com/auth/calendar.readonly

Google Classroom:
✅ https://www.googleapis.com/auth/classroom.courses.readonly
✅ https://www.googleapis.com/auth/classroom.coursework.me.readonly
✅ https://www.googleapis.com/auth/classroom.announcements.readonly
```

**Test Users:**
- Add your Gmail address for testing

### Step 4: Implement OAuth Connection Flow

**Backend Endpoint:** `POST /api/google/connect`

```python
from fastapi import APIRouter, Depends
from google_auth_oauthlib.flow import Flow
import os

router = APIRouter()

SCOPES = [
    'https://www.googleapis.com/auth/gmail.readonly',
    'https://www.googleapis.com/auth/gmail.send',
    'https://www.googleapis.com/auth/calendar.readonly',
    'https://www.googleapis.com/auth/classroom.courses.readonly',
    'https://www.googleapis.com/auth/classroom.coursework.me.readonly',
    'https://www.googleapis.com/auth/classroom.announcements.readonly'
]

@router.get("/google/auth-url")
async def get_google_auth_url(current_user = Depends(get_current_user)):
    """
    Generate Google OAuth URL for user to connect their Google account.
    Called from frontend when user clicks "Connect Google Account".
    """
    flow = Flow.from_client_secrets_file(
        'credentials.json',
        scopes=SCOPES,
        redirect_uri=f"{os.getenv('FRONTEND_URL')}/api/auth/google/callback"
    )

    authorization_url, state = flow.authorization_url(
        access_type='offline',
        include_granted_scopes='true',
        prompt='consent',  # Force consent to get refresh token
        state=current_user['id']  # Pass user ID via state
    )

    return {
        "auth_url": authorization_url,
        "state": state
    }


@router.get("/google/callback")
async def google_oauth_callback(code: str, state: str):
    """
    Handle OAuth callback from Google.
    Exchanges code for tokens and saves to database.
    """
    user_id = state  # User ID passed via state

    flow = Flow.from_client_secrets_file(
        'credentials.json',
        scopes=SCOPES,
        redirect_uri=f"{os.getenv('FRONTEND_URL')}/api/auth/google/callback"
    )

    # Exchange authorization code for tokens
    flow.fetch_token(code=code)
    credentials = flow.credentials

    # Save tokens to database
    from datetime import datetime, timedelta

    token_data = {
        'user_id': user_id,
        'provider': 'google',
        'access_token': encrypt_token(credentials.token),
        'refresh_token': encrypt_token(credentials.refresh_token),
        'expires_at': (datetime.now() + timedelta(hours=1)).isoformat(),
        'scopes': SCOPES,
        'created_at': datetime.now().isoformat()
    }

    supabase.table('oauth_tokens').upsert(token_data).execute()

    return {"status": "connected", "message": "Google account connected successfully"}


def encrypt_token(token: str) -> str:
    """Encrypt token using Fernet"""
    from cryptography.fernet import Fernet
    key = os.getenv('ENCRYPTION_KEY').encode()
    f = Fernet(key)
    return f.encrypt(token.encode()).decode()
```

### Step 5: Frontend Google Connection UI

**File:** `frontend/components/GoogleConnectionButton.tsx`

```typescript
'use client'

import { useState } from 'react'
import { useUser } from '@clerk/nextjs'

export default function GoogleConnectionButton() {
  const { user } = useUser()
  const [connecting, setConnecting] = useState(false)
  const [connected, setConnected] = useState(false)

  const connectGoogle = async () => {
    setConnecting(true)

    try {
      // Get OAuth URL from backend
      const res = await fetch('/api/google/auth-url', {
        headers: {
          'Authorization': `Bearer ${await user?.getToken()}`
        }
      })

      const { auth_url } = await res.json()

      // Redirect to Google OAuth
      window.location.href = auth_url

    } catch (error) {
      console.error('Failed to connect Google:', error)
      setConnecting(false)
    }
  }

  return (
    <button
      onClick={connectGoogle}
      disabled={connecting || connected}
      className="btn-primary"
    >
      {connecting ? 'Connecting...' : connected ? '✓ Connected' : 'Connect Google Account'}
    </button>
  )
}
```

**Callback Page:** `frontend/app/api/auth/google/callback/page.tsx`

```typescript
'use client'

import { useEffect } from 'use'
import { useSearchParams, useRouter } from 'next/navigation'

export default function GoogleCallbackPage() {
  const searchParams = useSearchParams()
  const router = useRouter()

  useEffect(() => {
    const code = searchParams.get('code')
    const state = searchParams.get('state')

    if (code && state) {
      // Backend handles token exchange
      fetch(`/api/google/callback?code=${code}&state=${state}`)
        .then(() => {
          router.push('/dashboard?google_connected=true')
        })
        .catch(error => {
          console.error('OAuth callback failed:', error)
          router.push('/dashboard?google_error=true')
        })
    }
  }, [searchParams, router])

  return (
    <div className="flex items-center justify-center min-h-screen">
      <div>Connecting your Google account...</div>
    </div>
  )
}
```

---

## Database Schema Updates

Update `users` table to include Clerk ID:

```sql
ALTER TABLE users
ADD COLUMN clerk_id VARCHAR(255) UNIQUE NOT NULL,
DROP COLUMN password_hash;  -- No longer needed with Clerk

-- Index for fast Clerk lookups
CREATE INDEX idx_users_clerk_id ON users(clerk_id);
```

`oauth_tokens` table remains the same (stores Google API tokens).

---

## Environment Variables

**Complete .env file:**

```bash
# Clerk Authentication
CLERK_SECRET_KEY=sk_test_xxxxxxxxxxxxxxxxxxxx
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_xxxxxxxxxxxxxxxxxxxx
CLERK_WEBHOOK_SECRET=whsec_xxxxxxxxxxxx

# Google OAuth (for API access only)
GOOGLE_CLIENT_ID=your-client-id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=GOCSPX-xxxxxxxxxxxxx

# Database
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_KEY=your-supabase-anon-key

# Encryption
ENCRYPTION_KEY=<32-byte-base64-key>  # For encrypting Google OAuth tokens

# Application
FRONTEND_URL=http://localhost:3000
BACKEND_URL=http://localhost:8000

# Gemini API
GEMINI_API_KEY=your-gemini-api-key

# WhatsApp Service
WHATSAPP_API_KEY=shared-secret-key
WHATSAPP_SERVICE_URL=http://whatsapp-service:3001
```

---

## Token Refresh (Google OAuth Only)

Clerk handles session refresh automatically. Only Google API tokens need manual refresh:

```python
# Celery task (runs daily)
@celery_app.task
def refresh_google_oauth_tokens():
    """Refresh expiring Google OAuth tokens"""
    from google.oauth2.credentials import Credentials
    from google.auth.transport.requests import Request
    from datetime import datetime, timedelta

    # Query tokens expiring in next 6 hours
    expiring_tokens = supabase.table('oauth_tokens') \
        .select('*') \
        .eq('provider', 'google') \
        .lt('expires_at', (datetime.now() + timedelta(hours=6)).isoformat()) \
        .execute()

    for token_record in expiring_tokens.data:
        creds = Credentials(
            token=decrypt_token(token_record['access_token']),
            refresh_token=decrypt_token(token_record['refresh_token']),
            token_uri='https://oauth2.googleapis.com/token',
            client_id=os.getenv('GOOGLE_CLIENT_ID'),
            client_secret=os.getenv('GOOGLE_CLIENT_SECRET')
        )

        # Refresh
        creds.refresh(Request())

        # Update database
        supabase.table('oauth_tokens').update({
            'access_token': encrypt_token(creds.token),
            'expires_at': creds.expiry.isoformat()
        }).eq('id', token_record['id']).execute()
```

---

## Testing

### Test User Authentication (Clerk):

1. Go to `http://localhost:3000/sign-in`
2. Sign in with Google or email
3. Verify JWT token in browser developer tools
4. Try accessing protected route

### Test Google API Connection:

1. Log in via Clerk
2. Click "Connect Google Account"
3. Authorize scopes
4. Verify tokens saved in `oauth_tokens` table
5. Test API access (fetch emails, assignments)

---

## Security Best Practices

1. **Clerk Webhooks:** Sync user deletions/updates to your database
2. **Token Encryption:** Always encrypt Google OAuth tokens at rest
3. **Scope Minimization:** Only request needed Google scopes
4. **HTTPS Only:** Enforce HTTPS in production for OAuth callbacks
5. **Environment Secrets:** Never commit credentials or keys

---

## Summary

| Component | Provider | Purpose |
|-----------|----------|---------|
| **User Auth** | Clerk | Login, signup, sessions |
| **Google APIs** | Google OAuth | Access Gmail, Calendar, Classroom |
| **Session Tokens** | Clerk JWT | Authorize API requests |
| **API Tokens** | Google OAuth | Call Google APIs |

**Flow:**
1. User signs in with Clerk → Gets Clerk JWT
2. User connects Google → Gets Google OAuth tokens
3. Backend uses Clerk JWT to identify user
4. Backend uses Google tokens to fetch data

✅ **Ready to implement!**
