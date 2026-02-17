# Docker Compose Setup & Deployment Documentation

## Objective

Create Docker Compose configuration for all services and comprehensive deployment documentation.

## Scope

**In Scope:**
- Docker Compose file with services:
  - backend (FastAPI, port 8000)
  - frontend (Next.js, port 3000)
  - celery-worker
  - celery-beat
  - redis (port 6379, with volume)
- Dockerfiles for backend and frontend
- Environment configuration (.env.example with all required variables)
- Volume configuration for Redis persistence
- Network configuration
- Health checks for services
- README with:
  - Prerequisites
  - Environment setup
  - OAuth token setup (manual for MVP)
  - Running with docker-compose
  - VPS deployment for WhatsApp service
  - Troubleshooting guide

**Out of Scope:**
- Production deployment (Kubernetes, CI/CD)
- Monitoring setup

## Spec References

- `spec:69889a16-05f6-4f4a-bf1a-8cf0daec03b5/4eb58cf0-bbfc-42e3-acd5-9fe60dd4f30b` (Tech Plan - Deployment Architecture)

## Acceptance Criteria

- [ ] `docker-compose up` starts all services
- [ ] Backend accessible at localhost:8000
- [ ] Frontend accessible at localhost:3000
- [ ] Celery worker processes tasks
- [ ] Celery beat schedules tasks
- [ ] Redis data persists
- [ ] .env.example documents all variables
- [ ] README covers setup and deployment
- [ ] WhatsApp VPS deployment documented
- [ ] All services communicate correctly

## Dependencies

- All previous tickets (complete system)