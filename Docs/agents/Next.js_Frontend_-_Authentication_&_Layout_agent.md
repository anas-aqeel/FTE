# Agent Specification: Next.js Frontend - Authentication & Layout

## 1. Purpose

Set up a production-ready Next.js frontend application with TypeScript, server-side rendering, authentication pages (login/logout), main application layout structure, and API client for backend communication in the Personal Assistant Agent system.

## 2. Scope

**In Scope:**
- Next.js project setup with TypeScript and SSR (App Router)
- Authentication pages:
  - Login page (/login) with username/password form
  - Logout functionality (clear JWT token, redirect to login)
- Main layout structure (for authenticated pages):
  - Header component with user info and logout button
  - Sidebar component for conversation list (placeholder initially)
  - Main content area for chat interface (placeholder initially)
  - Responsive design foundation (mobile-first)
- API client setup for backend communication:
  - Axios or fetch with automatic JWT token injection
  - Base URL configuration (http://localhost:8000 for development)
  - Error handling and retry logic
- JWT token storage:
  - Store in localStorage (simple for MVP) or httpOnly cookies (more secure)
  - Automatic token refresh logic (future enhancement)
- Authentication state management:
  - React Context or Zustand for global auth state
  - isAuthenticated, currentUser, login, logout functions
- Protected routes:
  - Redirect to /login if not authenticated
  - Middleware or route guards
- Responsive design:
  - Mobile-friendly (sidebar collapsible)
  - Tailwind CSS for styling
- Environment configuration (.env.local for NEXT_PUBLIC_API_URL)

**Out of Scope:**
- Chat interface implementation (separate ticket)
- Conversation management UI (separate ticket)
- Sync status display (separate ticket)
- Settings page (Phase 2)
- User registration (Phase 2)

## 3. Inputs

- **Environment Variables** (.env.local):
  - NEXT_PUBLIC_API_URL: Backend API base URL (http://localhost:8000)
- **Backend API Endpoints**:
  - POST /auth/login: {username, password} → {access_token, user}
  - GET /auth/me: (with JWT) → {id, username}
  - POST /auth/logout: (with JWT) → {message}
- **User Input**:
  - Login form: username, password
- **Local Storage** (or cookies):
  - JWT token (access_token)
  - User data (username, id)

## 4. Outputs

- **Next.js Application**:
  - Project structure:
    ```
    frontend/
    ├── app/
    │   ├── layout.tsx (root layout)
    │   ├── page.tsx (home/redirect)
    │   ├── login/
    │   │   └── page.tsx
    │   └── dashboard/
    │       ├── layout.tsx (authenticated layout)
    │       └── page.tsx (placeholder)
    ├── components/
    │   ├── Header.tsx
    │   ├── Sidebar.tsx
    │   └── ProtectedRoute.tsx
    ├── lib/
    │   ├── api.ts (API client)
    │   └── auth.ts (auth helpers)
    ├── context/
    │   └── AuthContext.tsx
    ├── .env.local
    ├── next.config.js
    ├── tailwind.config.js
    └── package.json
    ```
- **Login Page UI**:
  - Username and password input fields
  - Login button
  - Error message display (invalid credentials)
  - Loading state during API call
- **Authenticated Layout UI**:
  - Header: App logo, user info (username), logout button
  - Sidebar: Placeholder for conversation list
  - Main content area: Placeholder or redirect to chat
- **API Client**:
  - HTTP client with JWT token injection
  - Functions: login(username, password), logout(), getCurrentUser()
- **Auth State Management**:
  - Global state: {isAuthenticated, user, login, logout}
  - Context provider wrapping app

## 5. Internal Responsibilities

1. **Next.js Project Initialization**:
   - Create Next.js app with TypeScript: npx create-next-app@latest --typescript
   - Install dependencies: axios (or use fetch), tailwindcss, zustand (optional)
   - Configure App Router (app/ directory)

2. **API Client Setup**:
   - Create lib/api.ts with Axios instance
   - Set baseURL from NEXT_PUBLIC_API_URL
   - Add request interceptor to inject JWT token from localStorage
   - Add response interceptor for error handling (401 → redirect to login)

3. **Authentication Context**:
   - Create context/AuthContext.tsx
   - Implement AuthProvider with state: {isAuthenticated, user, login, logout}
   - login function: Call POST /auth/login, store token in localStorage, set user state
   - logout function: Clear token from localStorage, reset state, redirect to /login
   - Initialize state on mount: Check localStorage for token, validate with GET /auth/me

4. **Login Page**:
   - Create app/login/page.tsx
   - Form with username and password inputs (controlled components)
   - Handle form submission: call AuthContext.login()
   - Display loading spinner during API call
   - Show error message on failure (invalid credentials)
   - Redirect to /dashboard on successful login

5. **Protected Routes**:
   - Create ProtectedRoute component or middleware
   - Check AuthContext.isAuthenticated
   - Redirect to /login if not authenticated
   - Wrap all authenticated pages with ProtectedRoute

6. **Authenticated Layout**:
   - Create app/dashboard/layout.tsx
   - Render Header, Sidebar, and children (main content area)
   - Apply responsive design (sidebar collapsible on mobile)

7. **Header Component**:
   - Display app logo/name
   - Show current user's username
   - Logout button: calls AuthContext.logout()

8. **Sidebar Component**:
   - Placeholder for conversation list (implement in next ticket)
   - Collapsible on mobile (hamburger menu)

9. **Responsive Design**:
   - Use Tailwind CSS utility classes
   - Mobile-first breakpoints (md:, lg:)
   - Test on mobile and desktop viewports

10. **Environment Configuration**:
    - Create .env.local with NEXT_PUBLIC_API_URL
    - Create .env.example for documentation
    - Never commit .env.local to Git

## 6. Dependencies

**External Services:**
- FastAPI Backend (authentication endpoints)

**Libraries/Tools:**
- Next.js 14+ with App Router
- TypeScript
- Tailwind CSS
- Axios (or fetch)
- React Context or Zustand (state management)

**Ticket Dependencies:**
- FastAPI_Project_Setup_&_Authentication (provides backend auth endpoints)

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

**Response Time**:
- Initial page load: <2 seconds
- Client-side navigation: <500ms

## 8. Failure Handling

**Backend API Failures:**
- Network errors: Display error message, retry button
- 401 Unauthorized: Redirect to /login (token expired or invalid)
- 500 Internal Server Error: Display generic error message

**Login Failures:**
- Invalid credentials (400): Show "Invalid username or password"
- Network timeout: Show "Unable to connect to server, please try again"

**Token Expiration:**
- JWT expires: Detect 401 response, redirect to /login
- Automatic refresh (Phase 2): Implement refresh token flow

**State Synchronization:**
- If localStorage token exists but GET /auth/me fails: Clear token, redirect to /login

## 9. Observability

**Logging** (client-side console logs for development):
- API requests (method, URL, status)
- Authentication events (login success/failure, logout)
- Errors (API errors, validation errors)

**Monitoring** (future):
- Frontend error tracking (Sentry, LogRocket)
- Performance monitoring (Vercel Analytics, Google Lighthouse)
- User analytics (page views, login frequency)

## 10. Security Considerations

**JWT Token Storage:**
- localStorage: Simple but vulnerable to XSS attacks
- httpOnly cookies: More secure (not accessible via JavaScript)
- Recommendation: Use httpOnly cookies for production

**XSS Prevention:**
- Sanitize all user input (React handles this by default)
- Never use dangerouslySetInnerHTML with user-provided content

**CSRF Protection:**
- If using cookies: Implement CSRF tokens
- If using localStorage: No CSRF risk (but XSS risk)

**HTTPS:**
- Use HTTPS in production (Vercel/Netlify handle automatically)

**Environment Variables:**
- Never expose backend secrets in NEXT_PUBLIC_* variables
- Only expose API URL (public information)

## 11. Scaling Considerations

**Current MVP:**
- Single user
- Low traffic
- Static build deployment

**Performance:**
- Next.js SSR provides fast initial load
- Code splitting reduces bundle size
- Tailwind CSS purges unused styles

**Future Scaling:**
- CDN deployment (Vercel, Netlify, Cloudflare)
- Image optimization (Next.js Image component)
- Static site generation (SSG) for public pages

## 12. Future Extensions

**Phase 2:**
- User registration page
- Password reset flow
- Remember me (persistent sessions)
- OAuth login (Google, GitHub)
- Two-factor authentication
- Settings page (profile, preferences)
- Dark mode
- Internationalization (i18n)
- Accessibility improvements (ARIA labels, keyboard navigation)
- Progressive Web App (PWA) support
- Offline mode (service workers)
