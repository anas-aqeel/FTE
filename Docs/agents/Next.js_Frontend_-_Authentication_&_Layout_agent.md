# Agent Specification: Next.js Frontend - Authentication & Layout

## 1. Purpose

Set up a production-ready Next.js frontend application with TypeScript, Clerk Auth integration for authentication, main application layout structure, and API client for backend communication in the AcademiQ system.

## 2. Scope

**In Scope:**
- Next.js project setup with TypeScript and App Router
- **Clerk Auth integration:**
  - ClerkProvider wrapping the application
  - Pre-built authentication UI (`<SignIn>`, `<SignUp>`, `<UserButton>`)
  - Protected routes using Clerk middleware
  - Automatic session management and token refresh
  - Social login (Google) and email/password via Clerk
- Authentication pages:
  - Sign-in page (/sign-in) with Clerk `<SignIn>` component
  - Sign-up page (/sign-up) with Clerk `<SignUp>` component
- Main layout structure (for authenticated pages):
  - Header component with Clerk `<UserButton>` and navigation
  - Sidebar component for conversation list (placeholder initially)
  - Main content area for chat interface (placeholder initially)
  - Responsive design foundation (mobile-first)
- API client setup for backend communication:
  - Server-side API client with automatic Clerk JWT injection
  - Client-side API client using Clerk `useUser` hook for token
  - Base URL configuration (http://localhost:8000 for development)
  - Error handling and retry logic
- Google Account connection UI:
  - "Connect Google Account" button for granting API access
  - Connection status display
- Protected routes:
  - Clerk middleware redirects unauthenticated users to /sign-in
  - Dashboard and all sub-routes protected
- Responsive design:
  - Mobile-friendly (sidebar collapsible)
  - Tailwind CSS for styling
- Environment configuration (.env.local for NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY, NEXT_PUBLIC_API_URL)

**Out of Scope:**
- ~~Custom login/signup forms~~ (Clerk provides UI)
- ~~Manual JWT storage~~ (Clerk handles automatically)
- ~~Custom auth state management~~ (Clerk provides hooks and context)
- Chat interface implementation (separate ticket)
- Conversation management UI (separate ticket)
- Sync status display (separate ticket)
- Settings page (Phase 2)

**Note:** All user authentication is handled by Clerk. This ticket focuses on integrating Clerk components, configuring protected routes, and connecting to the backend with Clerk JWTs.

## 3. Inputs

- **Environment Variables** (.env.local):
  - NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY: Clerk publishable key (from Clerk Dashboard)
  - CLERK_SECRET_KEY: Clerk secret key (for server-side operations)
  - NEXT_PUBLIC_API_URL: Backend API base URL (http://localhost:8000)
- **Clerk Dashboard Configuration**:
  - Clerk application created
  - Social login (Google) enabled
  - Email/password authentication enabled
  - Redirect URLs configured
- **Backend API Endpoints**:
  - GET /auth/me: (with Clerk JWT) → {id, clerk_id, email, username}
  - GET /google/auth-url: (with Clerk JWT) → {auth_url}
  - GET /google/callback: (OAuth callback)
- **Clerk SDK**:
  - `@clerk/nextjs` package
  - ClerkProvider, SignIn, SignUp, UserButton components
  - useUser, useAuth hooks
  - clerkMiddleware for route protection
  - auth() for server-side token access

## 4. Outputs

- **Next.js Application**:
  - Project structure:
    ```
    frontend/
    ├── app/
    │   ├── layout.tsx (root layout with ClerkProvider)
    │   ├── page.tsx (landing page)
    │   ├── sign-in/
    │   │   └── [[...sign-in]]/
    │   │       └── page.tsx (Clerk SignIn component)
    │   ├── sign-up/
    │   │   └── [[...sign-up]]/
    │   │       └── page.tsx (Clerk SignUp component)
    │   └── dashboard/
    │       ├── layout.tsx (authenticated layout with header/sidebar)
    │       ├── page.tsx (dashboard home with Google connection)
    │       └── settings/
    │           └── page.tsx (placeholder)
    ├── components/
    │   ├── GoogleConnectionButton.tsx
    │   └── (future components)
    ├── lib/
    │   └── api.ts (API client with Clerk token injection)
    ├── middleware.ts (Clerk route protection)
    ├── .env.local
    ├── next.config.js
    ├── tailwind.config.js
    └── package.json
    ```
- **Sign-In Page UI**: Clerk `<SignIn>` component with Google social login and email/password
- **Sign-Up Page UI**: Clerk `<SignUp>` component
- **Authenticated Layout UI**:
  - Header: App name, user email, Clerk `<UserButton>` (profile, sign out)
  - Sidebar: Navigation links, conversation list placeholder
  - Main content area: Dashboard home or sub-pages
- **API Client**:
  - Server-side: `serverFetch(endpoint, options)` - auto-injects Clerk JWT from `auth()`
  - Client-side: `clientFetch(endpoint, token, options)` - uses token from `useUser().getToken()`
- **Google Connection**: Button to initiate Google OAuth flow via backend

## 5. Internal Responsibilities

1. **Next.js Project Initialization**:
   - Create Next.js app with TypeScript: `npx create-next-app@latest --typescript --tailwind --app --eslint`
   - Install Clerk: `npm install @clerk/nextjs`
   - Configure App Router (app/ directory)

2. **Clerk Provider Setup**:
   - Wrap application with `<ClerkProvider>` in root layout.tsx
   - Configure Clerk appearance/theming if desired
   - Set up environment variables (NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY, CLERK_SECRET_KEY)

3. **Authentication Pages**:
   - Create /sign-in/[[...sign-in]]/page.tsx with Clerk `<SignIn>` component
   - Create /sign-up/[[...sign-up]]/page.tsx with Clerk `<SignUp>` component
   - Configure routing props (path, signUpUrl, signInUrl, afterSignInUrl, afterSignUpUrl)
   - Style with centered layout and consistent design

4. **Clerk Middleware (Protected Routes)**:
   - Create middleware.ts in project root
   - Use `clerkMiddleware` and `createRouteMatcher`
   - Define public routes: /, /sign-in(.*), /sign-up(.*), /api/webhooks(.*)
   - Protect all other routes (redirect to /sign-in if unauthenticated)
   - Configure matcher to skip Next.js internals and static files

5. **Dashboard Layout (Authenticated)**:
   - Create app/dashboard/layout.tsx
   - Use `currentUser()` server function to get authenticated user
   - Redirect to /sign-in if not authenticated
   - Render Header with user email and Clerk `<UserButton>`
   - Render Sidebar with navigation links and conversation list placeholder
   - Render main content area with children

6. **Dashboard Home Page**:
   - Create app/dashboard/page.tsx
   - Display welcome message with user's name
   - Show "Get Started" section with Google Account connection
   - Show "Start a Conversation" teaser

7. **Google Connection Button**:
   - Create components/GoogleConnectionButton.tsx
   - Use `useUser()` hook to get Clerk token
   - On click: Call `GET /google/auth-url` with Clerk JWT
   - Receive Google OAuth URL from backend
   - Redirect user to Google consent screen
   - Display connection status (connecting, connected, error)

8. **API Client Setup**:
   - Create lib/api.ts
   - Server-side `serverFetch()`: Uses `auth()` from @clerk/nextjs/server to get token
   - Client-side `clientFetch()`: Accepts token parameter (from `user.getToken()`)
   - Both inject Authorization: Bearer {token} header
   - Both set Content-Type: application/json
   - Handle errors (401 → redirect, 500 → display message)

9. **Landing Page**:
   - Create app/page.tsx (public route)
   - Display app name and description
   - "Get Started" link to /sign-up
   - "Sign In" link to /sign-in
   - Simple, clean design

10. **Responsive Design**:
    - Use Tailwind CSS utility classes
    - Mobile-first breakpoints (md:, lg:)
    - Sidebar collapsible on mobile
    - Test on mobile and desktop viewports

11. **Environment Configuration**:
    - Create .env.local with NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY, CLERK_SECRET_KEY, NEXT_PUBLIC_API_URL
    - Create .env.example for documentation
    - Never commit .env.local to Git

## 6. Dependencies

**External Services:**
- Clerk Auth (Clerk application configured in Clerk Dashboard)
- FastAPI Backend (authentication endpoints with Clerk JWT verification)

**Libraries/Tools:**
- Next.js 14+ with App Router
- TypeScript
- @clerk/nextjs (Clerk SDK for Next.js)
- Tailwind CSS

**Ticket Dependencies:**
- FastAPI_Project_Setup_&_Authentication (provides backend Clerk JWT verification and Google OAuth endpoints)

**Blocks:**
- Next.js_Frontend_-_Chat_Interface_&_Conversation_Management (needs layout and auth)

## 7. Execution Model

**Type**: Server-Side Rendered Web Application

**Lifecycle**:
- Development: npm run dev (runs on http://localhost:3000)
- Production: npm run build && npm start

**Rendering**:
- SSR for initial page load (faster first contentful paint)
- Client-side navigation (React Router behavior)
- Clerk middleware runs on edge for fast route protection

**Response Time**:
- Initial page load: <2 seconds
- Client-side navigation: <500ms
- Clerk auth check: <100ms (cached sessions)

## 8. Failure Handling

**Backend API Failures:**
- Network errors: Display error message, retry button
- 401 Unauthorized: Clerk auto-refreshes token; if persistent, redirect to /sign-in
- 500 Internal Server Error: Display generic error message

**Clerk Authentication Failures:**
- Clerk service down: Show error page with "Authentication service unavailable"
- Social login failure: Clerk displays error in its own UI
- Token refresh failure: Redirect to /sign-in

**Google OAuth Failures:**
- Backend returns error: Display "Failed to connect Google Account, please try again"
- User denies consent: Show appropriate message
- Callback error: Display error with retry option

**State Synchronization:**
- Clerk handles all session state (no localStorage management needed)
- If `auth()` returns null on server, redirect to /sign-in

## 9. Observability

**Logging** (client-side console logs for development):
- API requests (method, URL, status)
- Google OAuth flow events (initiated, success, error)
- Errors (API errors, component errors)

**Monitoring** (future):
- Frontend error tracking (Sentry, LogRocket)
- Performance monitoring (Vercel Analytics, Google Lighthouse)
- User analytics (page views, sign-in frequency)
- Clerk Dashboard provides built-in user analytics

## 10. Security Considerations

**Authentication Security (Clerk)**:
- Clerk handles all authentication security (password hashing, session management, token refresh)
- No JWT tokens stored in localStorage (Clerk manages token lifecycle)
- No custom auth code to maintain or audit
- Clerk provides CSRF protection, XSS protection, and secure session cookies
- All Clerk communication uses HTTPS

**XSS Prevention:**
- Sanitize all user input (React handles this by default)
- Never use dangerouslySetInnerHTML with user-provided content

**Google OAuth Security:**
- OAuth tokens never exposed to frontend (stored in backend database)
- Frontend only initiates OAuth flow via backend endpoint
- State parameter prevents CSRF in OAuth flow

**HTTPS:**
- Use HTTPS in production (Vercel/Netlify handle automatically)
- Clerk enforces HTTPS for all authentication flows

**Environment Variables:**
- Never expose CLERK_SECRET_KEY in NEXT_PUBLIC_* variables
- Only expose NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY (designed to be public)
- Only expose NEXT_PUBLIC_API_URL (public information)

## 11. Scaling Considerations

**Current MVP:**
- Single user
- Low traffic
- Static build deployment

**Performance:**
- Next.js SSR provides fast initial load
- Clerk middleware on edge for fast auth checks
- Code splitting reduces bundle size
- Tailwind CSS purges unused styles

**Future Scaling:**
- CDN deployment (Vercel, Netlify, Cloudflare)
- Image optimization (Next.js Image component)
- Static site generation (SSG) for public pages

## 12. Future Extensions

**Phase 2:**
- Dark mode toggle
- Settings page (user preferences, notification settings)
- Internationalization (i18n)
- Accessibility improvements (ARIA labels, keyboard navigation)
- Progressive Web App (PWA) support
- Offline mode (service workers)
- Additional Clerk features:
  - Organization support (multi-user teams)
  - Multi-factor authentication (configured in Clerk Dashboard)
  - Additional social login providers (GitHub, Microsoft)
  - Custom user metadata via Clerk
