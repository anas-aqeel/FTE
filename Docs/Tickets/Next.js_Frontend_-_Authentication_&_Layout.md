# Next.js Frontend - Authentication & Layout

## Objective

Set up Next.js project with authentication pages and main layout structure.

## Scope

**In Scope:**
- Next.js project setup with TypeScript and SSR
- Authentication pages:
  - Login page (username/password form)
  - Logout functionality
- Main layout structure:
  - Header with user info and logout
  - Sidebar for conversation list (placeholder)
  - Main content area (placeholder)
- API client setup for backend communication
- JWT token storage (localStorage or cookies)
- Authentication state management (React Context or Zustand)
- Protected routes (redirect to login if not authenticated)
- Responsive design foundation

**Out of Scope:**
- Chat interface
- Conversation management
- Sync status

## Spec References

- `spec:69889a16-05f6-4f4a-bf1a-8cf0daec03b5/4eb58cf0-bbfc-42e3-acd5-9fe60dd4f30b` (Tech Plan - Next.js Frontend)

## Acceptance Criteria

- [ ] Next.js app runs successfully
- [ ] Login page works (username/password)
- [ ] JWT token stored after login
- [ ] Protected routes redirect to login
- [ ] Main layout renders with header and sidebar
- [ ] Logout functionality works
- [ ] Responsive design on mobile
- [ ] API client configured

## Dependencies

- Backend authentication endpoints