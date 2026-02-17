# Clerk Auth Migration Summary

**Date:** 2026-02-18
**Change:** Migrated from username/password JWT auth to Clerk Auth

---

## Overview

The system now uses **Clerk Auth** for user authentication instead of custom username/password authentication. Google OAuth is still used separately for API access to Gmail and Classroom.

---

## Key Changes

### 1. **Authentication Architecture**

**Before:**
- Username/password stored in database
- Custom JWT token generation
- Password hashing with bcrypt

**After:**
- Clerk handles all user authentication
- Clerk provides JWT tokens
- No password storage in our database
- Support for social login (Google, GitHub, email/password)

---

## Updated Files

### Specifications

✅ **OAuth_Setup_Guide.md**
- Now covers both Clerk Auth (Part 1) and Google OAuth for APIs (Part 2)
- Separated user authentication from API access
- Added Clerk SDK integration examples
- Updated environment variables

### Tickets Requiring Updates

The following tickets need to be updated to reflect Clerk Auth:

#### 1. **Setup_Supabase_Project_&_Core_Tables.md**
**Changes:**
```sql
-- OLD users table:
CREATE TABLE users (
    id UUID PRIMARY KEY,
    username VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,  -- REMOVE
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- NEW users table:
CREATE TABLE users (
    id UUID PRIMARY KEY,
    clerk_id VARCHAR(255) UNIQUE NOT NULL,  -- ADD
    email VARCHAR(255) NOT NULL,            -- ADD
    username VARCHAR(255),                  -- Make optional
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

CREATE INDEX idx_users_clerk_id ON users(clerk_id);
```

#### 2. **FastAPI_Project_Setup_&_Authentication.md**
**Changes:**
- Replace JWT token generation with Clerk JWT verification
- Remove password hashing logic
- Update authentication endpoints:
  - ~~POST /auth/login~~ → Handled by Clerk
  - ~~POST /auth/logout~~ → Handled by Clerk
  - `GET /auth/me` → Verify Clerk token and return user
- Add Clerk SDK integration
- Update middleware to use `get_current_user` from Clerk
- Remove user seeding (users created via Clerk signup)

**New Dependencies:**
```python
clerk-backend-api==0.1.0  # Clerk Python SDK
```

**New Environment Variables:**
```bash
CLERK_SECRET_KEY=sk_test_...
CLERK_WEBHOOK_SECRET=whsec_...  # For syncing user events
```

#### 3. **Next.js_Frontend_-_Authentication_&_Layout.md**
**Changes:**
- Replace custom login page with Clerk's `<SignIn />` component
- Replace custom signup with Clerk's `<SignUp />` component
- Use Clerk's `<UserButton />` for user menu
- Update authentication state management:
  - Use `useUser()` hook instead of custom context
  - Use `useAuth()` for session management
- Update protected routes to use Clerk's auth helpers
- Remove JWT token storage (handled by Clerk)

**New Dependencies:**
```bash
npm install @clerk/nextjs
```

**New Environment Variables:**
```bash
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
```

#### 4. **Celery_Maintenance_Tasks.md**
**Changes:**
- Update `refresh_oauth_tokens()` to only refresh Google OAuth tokens (not user auth tokens)
- Clerk handles user session refresh automatically
- Keep Google API token refresh logic unchanged

**No code changes needed** - just clarify in documentation that only Google tokens are refreshed.

---

## Database Migration Script

**File:** `migrations/002_add_clerk_auth.sql`

```sql
-- Add Clerk ID to users table
ALTER TABLE users
ADD COLUMN clerk_id VARCHAR(255) UNIQUE,
ADD COLUMN email VARCHAR(255);

-- Make username optional (was required)
ALTER TABLE users
ALTER COLUMN username DROP NOT NULL;

-- Remove password hash (no longer needed)
ALTER TABLE users
DROP COLUMN IF EXISTS password_hash;

-- Create index for Clerk lookups
CREATE INDEX IF NOT EXISTS idx_users_clerk_id ON users(clerk_id);

-- Add email index for lookups
CREATE INDEX IF NOT EXISTS idx_users_email ON users(email);

COMMENT ON COLUMN users.clerk_id IS 'Clerk user ID (e.g., user_2xxx...)';
COMMENT ON COLUMN users.email IS 'User email from Clerk';
```

---

## Environment Variables Update

### Add to `.env`:

```bash
# ==========================================
# Clerk Authentication (NEW)
# ==========================================
CLERK_SECRET_KEY=sk_test_xxxxxxxxxxxxxxxxxxxx
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_xxxxxxxxxxxxxxxxxxxx
CLERK_WEBHOOK_SECRET=whsec_xxxxxxxxxxxx

# ==========================================
# Google OAuth (for API access - UNCHANGED)
# ==========================================
GOOGLE_CLIENT_ID=your-client-id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=GOCSPX-xxxxxxxxxxxxx

# Other variables remain the same...
```

### Remove from `.env`:

```bash
# REMOVE these (no longer needed):
# JWT_SECRET_KEY=...
# INITIAL_USERNAME=...
# INITIAL_PASSWORD=...
```

---

## Updated User Flow

### Before (Custom Auth):
1. User enters username/password
2. Backend validates credentials
3. Backend generates JWT
4. Frontend stores JWT in localStorage
5. Frontend sends JWT in Authorization header

### After (Clerk Auth):
1. User clicks "Sign In" → Clerk modal opens
2. User signs in with Google or email/password via Clerk
3. Clerk handles authentication and session
4. Clerk provides JWT automatically
5. Frontend uses `useUser()` to get Clerk token
6. Frontend sends Clerk JWT to backend
7. Backend verifies Clerk JWT and syncs user to database

---

## Google Account Connection (NEW FLOW)

After Clerk authentication, users must separately connect their Google account for API access:

1. User logs in via Clerk ✅ (User authenticated)
2. Dashboard shows "Connect Google Account" button
3. User clicks button → Redirected to Google OAuth
4. User grants Gmail, Calendar, Classroom permissions
5. Backend receives OAuth tokens and stores in `oauth_tokens` table
6. Data ingestion can now access user's Gmail/Classroom ✅

**Two separate tokens:**
- **Clerk JWT:** Identifies user to our backend
- **Google OAuth tokens:** Access user's Gmail/Classroom data

---

## Implementation Order

1. ✅ Update OAuth_Setup_Guide.md (DONE)
2. ⏳ Update database schema (add clerk_id, remove password_hash)
3. ⏳ Update FastAPI authentication (Clerk SDK)
4. ⏳ Update Next.js frontend (Clerk components)
5. ⏳ Test Clerk authentication flow
6. ⏳ Implement Google account connection UI
7. ⏳ Test end-to-end flow (Clerk login → Google connect → data ingestion)

---

## Testing Checklist

### Clerk Authentication:
- [ ] User can sign up with email/password
- [ ] User can sign in with Google (social login)
- [ ] User session persists across page refreshes
- [ ] User can sign out
- [ ] Protected routes redirect to sign-in
- [ ] Clerk JWT is verified by backend
- [ ] User data syncs to database on first login

### Google API Connection:
- [ ] "Connect Google Account" button appears after Clerk login
- [ ] OAuth flow completes successfully
- [ ] Google tokens saved to database
- [ ] Gmail ingestion works with saved tokens
- [ ] Classroom ingestion works with saved tokens
- [ ] Token refresh works for Google tokens

---

## Clerk Webhooks (User Sync)

Set up webhooks in Clerk Dashboard to sync user events:

**Webhook URL:** `https://your-backend.com/webhooks/clerk`

**Events to subscribe:**
- `user.created` - Create user in database
- `user.updated` - Update user email/username
- `user.deleted` - Delete user and all their data

**Backend Handler:**
```python
from fastapi import APIRouter, Request, HTTPException
from svix.webhooks import Webhook
import os

router = APIRouter()

@router.post("/webhooks/clerk")
async def handle_clerk_webhook(request: Request):
    """Sync Clerk user events to database"""

    # Verify webhook signature
    svix_id = request.headers.get("svix-id")
    svix_timestamp = request.headers.get("svix-timestamp")
    svix_signature = request.headers.get("svix-signature")

    webhook = Webhook(os.getenv('CLERK_WEBHOOK_SECRET'))

    try:
        payload = await request.body()
        event = webhook.verify(payload, {
            "svix-id": svix_id,
            "svix-timestamp": svix_timestamp,
            "svix-signature": svix_signature
        })
    except Exception as e:
        raise HTTPException(status_code=400, detail="Invalid signature")

    # Handle events
    event_type = event["type"]
    user_data = event["data"]

    if event_type == "user.created":
        # Create user in database
        supabase.table('users').insert({
            'clerk_id': user_data['id'],
            'email': user_data['email_addresses'][0]['email_address'],
            'username': user_data.get('username'),
            'created_at': user_data['created_at']
        }).execute()

    elif event_type == "user.updated":
        # Update user in database
        supabase.table('users').update({
            'email': user_data['email_addresses'][0]['email_address'],
            'username': user_data.get('username'),
            'updated_at': user_data['updated_at']
        }).eq('clerk_id', user_data['id']).execute()

    elif event_type == "user.deleted":
        # Delete user (CASCADE will delete all their data)
        supabase.table('users').delete().eq('clerk_id', user_data['id']).execute()

    return {"status": "success"}
```

---

## Benefits of Clerk Auth

✅ **No password management:** Clerk handles hashing, validation, security
✅ **Social login:** Google, GitHub, etc. out of the box
✅ **Email verification:** Built-in email verification flow
✅ **Multi-factor auth:** Optional MFA for extra security
✅ **Session management:** Automatic token refresh
✅ **Security best practices:** Clerk handles JWT signing, rotation, etc.
✅ **User management UI:** Clerk dashboard for managing users
✅ **Webhooks:** Real-time user event notifications

---

## Cost Considerations

**Clerk Pricing (as of 2024):**
- **Free tier:** 10,000 monthly active users
- **Pro tier:** $25/month for up to 10,000 MAUs
- **Enterprise:** Custom pricing

For a personal assistant MVP (single user or small group), **free tier is sufficient**.

---

## Rollback Plan (If Needed)

If Clerk Auth causes issues, reverting is straightforward:

1. Keep old authentication code in a separate branch
2. Restore `password_hash` column to users table
3. Re-enable custom JWT endpoints
4. Switch frontend back to custom login forms
5. Remove Clerk SDK dependencies

**Estimated rollback time:** 2-4 hours

---

## Summary

| Aspect | Before | After |
|--------|--------|-------|
| **User Auth** | Username/password + custom JWT | Clerk Auth (social login + email) |
| **User Table** | Stores password_hash | Stores clerk_id |
| **Login UI** | Custom form | Clerk components |
| **Session Management** | Manual JWT storage | Clerk handles automatically |
| **Google OAuth** | Mixed with user auth | Separate (for API access only) |
| **Security** | Manual implementation | Managed by Clerk |

**Status:** ✅ OAuth guide updated, tickets pending update

**Next Steps:**
1. Update affected tickets with Clerk Auth changes
2. Update database migration to add clerk_id
3. Implement Clerk SDK in backend and frontend
4. Test complete authentication flow

---

**Last Updated:** 2026-02-18
