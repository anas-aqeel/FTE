# Next.js Frontend - Authentication & Layout (Clerk Auth)

## Objective

Set up Next.js project with **Clerk Auth** authentication and main layout structure.

## Scope

**In Scope:**
- Next.js project setup with TypeScript and App Router
- **Clerk Auth integration:**
  - Clerk Provider setup
  - Pre-built authentication UI (`<SignIn>`, `<SignUp>`, `<UserButton>`)
  - Protected routes using Clerk middleware
  - Automatic session management
- Main layout structure:
  - Header with user button and navigation
  - Sidebar for conversation list (placeholder)
  - Main content area (placeholder)
- API client setup for backend communication (with Clerk JWT)
- Responsive design foundation
- Google Account connection UI (for API access)

**Out of Scope:**
- ~~Custom login/signup forms~~ (Clerk provides UI)
- ~~Manual JWT storage~~ (Clerk handles automatically)
- Chat interface
- Conversation management
- Sync status

**Note:** All user authentication is handled by Clerk. This ticket focuses on integrating Clerk components and connecting to our backend.

## Spec References

- `spec:69889a16-05f6-4f4a-bf1a-8cf0daec03b5/4eb58cf0-bbfc-42e3-acd5-9fe60dd4f30b` (Tech Plan - Next.js Frontend)
- `docs/specs/OAuth_Setup_Guide.md` (Part 1: Clerk Auth Setup)

## Acceptance Criteria

- [ ] Next.js app runs successfully
- [ ] Clerk Provider wraps the application
- [ ] Sign-in page renders with Clerk component
- [ ] Sign-up page renders with Clerk component
- [ ] Users can sign in with Google (social login)
- [ ] Users can sign in with email/password
- [ ] Protected routes redirect unauthenticated users to sign-in
- [ ] Main layout renders with header and sidebar
- [ ] User button shows user profile and logout
- [ ] Session persists across page refreshes
- [ ] Clerk JWT automatically sent to backend
- [ ] "Connect Google Account" button appears for API access setup
- [ ] Responsive design on mobile
- [ ] API client configured with automatic Clerk token injection

## Dependencies

- Backend authentication endpoints (Clerk JWT verification)
- Clerk application created and configured

---

## Implementation Details

### 1. Create Next.js Project

```bash
npx create-next-app@latest academiq-frontend --typescript --tailwind --app --eslint
cd personal-assistant-frontend
```

### 2. Install Clerk

```bash
npm install @clerk/nextjs
```

### 3. Environment Variables

**File:** `.env.local`

```bash
# Clerk (from Clerk Dashboard)
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_xxxxxxxxxxxxxxxxxxxx
CLERK_SECRET_KEY=sk_test_xxxxxxxxxxxxxxxxxxxx

# Backend API
NEXT_PUBLIC_API_URL=http://localhost:8000
```

### 4. Wrap App with ClerkProvider

**File:** `app/layout.tsx`

```typescript
import { ClerkProvider } from '@clerk/nextjs'
import './globals.css'
import type { Metadata } from 'next'

export const metadata: Metadata = {
  title: 'AcademiQ',
  description: 'AI-powered personal assistant for students',
}

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

### 5. Create Authentication Pages

**Sign In Page:** `app/sign-in/[[...sign-in]]/page.tsx`

```typescript
import { SignIn } from '@clerk/nextjs'

export default function SignInPage() {
  return (
    <div className="flex items-center justify-center min-h-screen bg-gray-50">
      <SignIn
        appearance={{
          elements: {
            rootBox: "mx-auto",
            card: "shadow-xl",
            headerTitle: "text-2xl font-bold",
            headerSubtitle: "text-gray-600",
            socialButtonsBlockButton: "border-2 hover:bg-gray-50",
            formButtonPrimary: "bg-blue-600 hover:bg-blue-700",
          }
        }}
        routing="path"
        path="/sign-in"
        signUpUrl="/sign-up"
        afterSignInUrl="/dashboard"
      />
    </div>
  )
}
```

**Sign Up Page:** `app/sign-up/[[...sign-up]]/page.tsx`

```typescript
import { SignUp } from '@clerk/nextjs'

export default function SignUpPage() {
  return (
    <div className="flex items-center justify-center min-h-screen bg-gray-50">
      <SignUp
        appearance={{
          elements: {
            rootBox: "mx-auto",
            card: "shadow-xl",
          }
        }}
        routing="path"
        path="/sign-up"
        signInUrl="/sign-in"
        afterSignUpUrl="/dashboard"
      />
    </div>
  )
}
```

### 6. Configure Clerk Middleware (Protected Routes)

**File:** `middleware.ts` (in root directory)

```typescript
import { clerkMiddleware, createRouteMatcher } from '@clerk/nextjs/server'

// Define public routes (accessible without authentication)
const isPublicRoute = createRouteMatcher([
  '/',
  '/sign-in(.*)',
  '/sign-up(.*)',
  '/api/webhooks(.*)',  // Webhooks don't need user auth
])

export default clerkMiddleware(async (auth, request) => {
  // Protect all routes except public ones
  if (!isPublicRoute(request)) {
    await auth.protect()
  }
})

export const config = {
  matcher: [
    // Skip Next.js internals and static files
    '/((?!_next|[^?]*\\.(?:html?|css|js(?!on)|jpe?g|webp|png|gif|svg|ttf|woff2?|ico|csv|docx?|xlsx?|zip|webmanifest)).*)',
    // Always run for API routes
    '/(api|trpc)(.*)',
  ],
}
```

### 7. Create Main Layout (Dashboard)

**File:** `app/dashboard/layout.tsx`

```typescript
import { UserButton, currentUser } from '@clerk/nextjs'
import { redirect } from 'next/navigation'

export default async function DashboardLayout({
  children,
}: {
  children: React.ReactNode
}) {
  // Verify user is authenticated
  const user = await currentUser()

  if (!user) {
    redirect('/sign-in')
  }

  return (
    <div className="flex h-screen bg-gray-100">
      {/* Sidebar */}
      <aside className="w-64 bg-white shadow-lg">
        <div className="p-4 border-b">
          <h1 className="text-xl font-bold text-gray-800">AcademiQ</h1>
        </div>

        <nav className="p-4">
          <div className="space-y-2">
            <a href="/dashboard" className="block px-4 py-2 rounded hover:bg-gray-100">
              💬 Chat
            </a>
            <a href="/dashboard/settings" className="block px-4 py-2 rounded hover:bg-gray-100">
              ⚙️ Settings
            </a>
          </div>
        </nav>

        {/* Conversation List Placeholder */}
        <div className="p-4 mt-4">
          <h3 className="text-sm font-semibold text-gray-500 mb-2">Conversations</h3>
          <div className="space-y-1">
            <div className="text-sm text-gray-400 italic">No conversations yet</div>
          </div>
        </div>
      </aside>

      {/* Main Content */}
      <div className="flex-1 flex flex-col">
        {/* Header */}
        <header className="bg-white shadow-sm px-6 py-4 flex items-center justify-between">
          <h2 className="text-lg font-semibold text-gray-800">Dashboard</h2>

          <div className="flex items-center gap-4">
            <span className="text-sm text-gray-600">
              {user.emailAddresses[0]?.emailAddress}
            </span>
            <UserButton
              appearance={{
                elements: {
                  avatarBox: "w-10 h-10"
                }
              }}
              afterSignOutUrl="/"
            />
          </div>
        </header>

        {/* Page Content */}
        <main className="flex-1 overflow-auto p-6">
          {children}
        </main>
      </div>
    </div>
  )
}
```

**Dashboard Home:** `app/dashboard/page.tsx`

```typescript
'use client'

import { useUser } from '@clerk/nextjs'
import GoogleConnectionButton from '@/components/GoogleConnectionButton'

export default function DashboardPage() {
  const { user } = useUser()

  return (
    <div className="max-w-4xl">
      <h1 className="text-3xl font-bold mb-4">
        Welcome, {user?.firstName || user?.username}! 👋
      </h1>

      <div className="bg-white rounded-lg shadow p-6 mb-6">
        <h2 className="text-xl font-semibold mb-4">Get Started</h2>

        <div className="space-y-4">
          <div>
            <h3 className="font-medium mb-2">1. Connect Your Google Account</h3>
            <p className="text-gray-600 text-sm mb-3">
              Grant access to your Gmail, Calendar, and Google Classroom to start tracking your schedule.
            </p>
            <GoogleConnectionButton />
          </div>

          <div>
            <h3 className="font-medium mb-2">2. Start a Conversation</h3>
            <p className="text-gray-600 text-sm">
              Once connected, you can ask questions like "What's my schedule today?" or "What assignments are due this week?"
            </p>
          </div>
        </div>
      </div>
    </div>
  )
}
```

### 8. Google Connection Component

**File:** `components/GoogleConnectionButton.tsx`

```typescript
'use client'

import { useState } from 'react'
import { useUser } from '@clerk/nextjs'

export default function GoogleConnectionButton() {
  const { user } = useUser()
  const [connecting, setConnecting] = useState(false)
  const [connected, setConnected] = useState(false)
  const [error, setError] = useState<string | null>(null)

  const connectGoogle = async () => {
    setConnecting(true)
    setError(null)

    try {
      // Get Clerk JWT token
      const token = await user?.getToken()

      if (!token) {
        throw new Error('Not authenticated')
      }

      // Request Google OAuth URL from backend
      const response = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/google/auth-url`, {
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        }
      })

      if (!response.ok) {
        throw new Error('Failed to get Google auth URL')
      }

      const { auth_url } = await response.json()

      // Redirect to Google OAuth
      window.location.href = auth_url

    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to connect')
      setConnecting(false)
    }
  }

  return (
    <div>
      <button
        onClick={connectGoogle}
        disabled={connecting || connected}
        className={`
          px-6 py-2 rounded-lg font-medium transition-colors
          ${connected
            ? 'bg-green-100 text-green-700 cursor-default'
            : connecting
            ? 'bg-gray-100 text-gray-500 cursor-wait'
            : 'bg-blue-600 text-white hover:bg-blue-700'
          }
        `}
      >
        {connecting && (
          <span className="inline-block mr-2">⏳</span>
        )}
        {connected && (
          <span className="inline-block mr-2">✓</span>
        )}
        {connecting
          ? 'Connecting...'
          : connected
          ? 'Google Connected'
          : 'Connect Google Account'}
      </button>

      {error && (
        <p className="mt-2 text-sm text-red-600">{error}</p>
      )}
    </div>
  )
}
```

### 9. API Client with Clerk Token

**File:** `lib/api.ts`

```typescript
import { auth } from '@clerk/nextjs/server'

const API_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000'

/**
 * Server-side API client (for Server Components)
 */
export async function serverFetch(
  endpoint: string,
  options: RequestInit = {}
) {
  const { getToken } = await auth()
  const token = await getToken()

  const response = await fetch(`${API_URL}${endpoint}`, {
    ...options,
    headers: {
      ...options.headers,
      'Authorization': token ? `Bearer ${token}` : '',
      'Content-Type': 'application/json',
    },
  })

  if (!response.ok) {
    throw new Error(`API error: ${response.statusText}`)
  }

  return response.json()
}

/**
 * Client-side API client (for Client Components)
 * Use with useUser hook to get token
 */
export async function clientFetch(
  endpoint: string,
  token: string,
  options: RequestInit = {}
) {
  const response = await fetch(`${API_URL}${endpoint}`, {
    ...options,
    headers: {
      ...options.headers,
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
  })

  if (!response.ok) {
    throw new Error(`API error: ${response.statusText}`)
  }

  return response.json()
}
```

**Example Usage (Server Component):**
```typescript
import { serverFetch } from '@/lib/api'

export default async function ConversationsPage() {
  const conversations = await serverFetch('/conversations')

  return <div>{/* Render conversations */}</div>
}
```

**Example Usage (Client Component):**
```typescript
'use client'

import { useUser } from '@clerk/nextjs'
import { clientFetch } from '@/lib/api'

export default function MyComponent() {
  const { user } = useUser()

  const fetchData = async () => {
    const token = await user?.getToken()
    if (token) {
      const data = await clientFetch('/conversations', token)
      console.log(data)
    }
  }

  return <button onClick={fetchData}>Fetch</button>
}
```

### 10. Landing Page

**File:** `app/page.tsx`

```typescript
import Link from 'next/link'

export default function LandingPage() {
  return (
    <div className="min-h-screen flex items-center justify-center bg-gradient-to-br from-blue-50 to-indigo-100">
      <div className="text-center">
        <h1 className="text-5xl font-bold text-gray-900 mb-4">
          AcademiQ
        </h1>
        <p className="text-xl text-gray-600 mb-8">
          Your AI-powered academic companion
        </p>

        <div className="flex gap-4 justify-center">
          <Link
            href="/sign-up"
            className="px-8 py-3 bg-blue-600 text-white rounded-lg font-semibold hover:bg-blue-700 transition"
          >
            Get Started
          </Link>
          <Link
            href="/sign-in"
            className="px-8 py-3 bg-white text-blue-600 rounded-lg font-semibold border-2 border-blue-600 hover:bg-blue-50 transition"
          >
            Sign In
          </Link>
        </div>
      </div>
    </div>
  )
}
```

---

## Testing

### 1. Test Sign Up Flow

1. Start app: `npm run dev`
2. Navigate to http://localhost:3000
3. Click "Get Started"
4. Sign up with Google or email
5. Verify redirect to `/dashboard`
6. Check Clerk Dashboard → Users tab (user should appear)

### 2. Test Sign In Flow

1. Sign out
2. Navigate to `/sign-in`
3. Sign in with created account
4. Verify session persists on refresh

### 3. Test Protected Routes

1. Sign out
2. Try to access `/dashboard` directly
3. Verify redirect to `/sign-in`
4. Sign in → Verify redirect back to `/dashboard`

### 4. Test API Integration

1. Sign in
2. Open browser console
3. Check Network tab for API requests
4. Verify `Authorization: Bearer <clerk-jwt>` header present

---

## Key Benefits of Clerk

✅ **No custom auth code:** Clerk components handle everything
✅ **Social login:** Google, GitHub, etc. built-in
✅ **Session management:** Automatic token refresh
✅ **Security:** Clerk handles JWT signing, encryption
✅ **User management:** Clerk Dashboard for managing users
✅ **Customizable UI:** Theming via `appearance` prop

---

## Dependencies

- Backend with Clerk JWT verification (FastAPI ticket)
- Clerk application configured in Clerk Dashboard
