# Agent Specification: Docker Compose Setup & Deployment Documentation

## 1. Purpose

Create comprehensive Docker Compose configuration to orchestrate all services (backend, frontend, Celery worker, Celery Beat, Redis) and provide detailed deployment documentation for local development, production deployment, and VPS setup for the WhatsApp service in the AcademiQ system.

## 2. Scope

**In Scope:**
- Docker Compose file (docker-compose.yml) with services:
  - backend (FastAPI, port 8000)
  - frontend (Next.js, port 3000)
  - celery-worker (background task processor)
  - celery-beat (task scheduler)
  - redis (message broker, port 6379, with volume for persistence)
- Dockerfiles:
  - backend/Dockerfile (Python, FastAPI, Celery)
  - frontend/Dockerfile (Node.js, Next.js)
- Environment configuration:
  - .env.example with all required variables documented
  - .env.template for quick setup
- Docker volume configuration:
  - Redis data persistence (./redis-data:/data)
  - Database connection (Supabase, no volume needed)
- Network configuration:
  - All services on same Docker network
  - Port mappings for external access
- Health checks:
  - Backend: GET /health endpoint
  - Frontend: HTTP 200 on port 3000
  - Redis: redis-cli ping
  - Celery worker: Celery inspect ping
- README.md with comprehensive documentation:
  - Prerequisites (Docker, Docker Compose, Supabase account)
  - Environment setup instructions
  - OAuth token setup (manual process for MVP)
  - Running with docker-compose (up, down, logs)
  - VPS deployment for WhatsApp service (OCI/Writer Cloud)
  - Troubleshooting guide (common issues, logs, debugging)
  - Development workflow (hot reload, debugging)
- WhatsApp service VPS deployment documentation:
  - VPS setup (Ubuntu 22.04, Docker installation)
  - WhatsApp service Docker deployment
  - Session persistence (volume mounting)
  - Monitoring and logs

**Out of Scope:**
- Production Kubernetes deployment (Phase 2)
- CI/CD pipeline (GitHub Actions, GitLab CI)
- Monitoring and alerting setup (Prometheus, Grafana)
- Load balancing and horizontal scaling
- Database backup and disaster recovery
- SSL certificate management (handled by reverse proxy)

## 3. Inputs

- **Docker Compose Configuration**:
  - Service definitions (backend, frontend, celery-worker, celery-beat, redis)
  - Environment variables (SUPABASE_URL, SUPABASE_KEY, CLERK_SECRET_KEY, GEMINI_API_KEY, etc.)
  - Volume mounts (Redis persistence)
  - Network configuration
- **Dockerfiles**:
  - backend/Dockerfile: Base image (python:3.10-slim), install dependencies, copy code, CMD
  - frontend/Dockerfile: Base image (node:18-alpine), build Next.js app, CMD
- **Environment Variables** (from all previous tickets):
  - SUPABASE_URL, SUPABASE_KEY, CLERK_SECRET_KEY, CLERK_WEBHOOK_SECRET
  - GEMINI_API_KEY, WHATSAPP_API_KEY
  - GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET, ENCRYPTION_KEY
  - REDIS_URL, BACKEND_API_URL, NEXT_PUBLIC_API_URL, NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY, FRONTEND_URL
- **OAuth Tokens** (manual setup):
  - User must authenticate Gmail/Classroom once
  - Tokens stored in oauth_tokens table

## 4. Outputs

- **docker-compose.yml**:
  ```yaml
  version: '3.8'
  services:
    backend:
      build: ./backend
      ports: ["8000:8000"]
      environment: [SUPABASE_URL, SUPABASE_KEY, CLERK_SECRET_KEY, CLERK_WEBHOOK_SECRET, GEMINI_API_KEY, GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET, ENCRYPTION_KEY, REDIS_URL, WHATSAPP_API_KEY, FRONTEND_URL]
      depends_on: [redis]
      healthcheck: {test: ["CMD", "curl", "-f", "http://localhost:8000/health"]}

    frontend:
      build: ./frontend
      ports: ["3000:3000"]
      environment: [NEXT_PUBLIC_API_URL, NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY]
      depends_on: [backend]

    celery-worker:
      build: ./backend
      command: celery -A app.celery worker --loglevel=info
      environment: [same as backend]
      depends_on: [redis, backend]

    celery-beat:
      build: ./backend
      command: celery -A app.celery beat --loglevel=info
      environment: [same as backend]
      depends_on: [redis, backend]

    redis:
      image: redis:7-alpine
      ports: ["6379:6379"]
      volumes: ["./redis-data:/data"]
      command: redis-server --appendonly yes
  ```

- **Dockerfiles**:
  - backend/Dockerfile: Multi-stage build, install Python deps, copy code
  - frontend/Dockerfile: Multi-stage build, npm install, npm run build, npm start

- **.env.example**:
  ```env
  # Supabase
  SUPABASE_URL=https://your-project.supabase.co
  SUPABASE_KEY=your-service-role-key

  # Clerk Auth
  CLERK_SECRET_KEY=sk_test_xxxxxxxxxxxxxxxxxxxx
  CLERK_WEBHOOK_SECRET=whsec_xxxxxxxxxxxxxxxxxxxx
  NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_xxxxxxxxxxxxxxxxxxxx

  # Google OAuth (for API access)
  GOOGLE_CLIENT_ID=your-client-id.apps.googleusercontent.com
  GOOGLE_CLIENT_SECRET=your-client-secret
  ENCRYPTION_KEY=your-32-byte-encryption-key

  # Services
  GEMINI_API_KEY=your-gemini-api-key
  WHATSAPP_API_KEY=your-whatsapp-api-key
  REDIS_URL=redis://redis:6379/0
  BACKEND_API_URL=http://backend:8000
  NEXT_PUBLIC_API_URL=http://localhost:8000
  FRONTEND_URL=http://localhost:3000
  ```

- **README.md** (comprehensive documentation):
  - Prerequisites section
  - Quick start guide (docker-compose up)
  - Environment setup (copy .env.example, fill values)
  - OAuth token setup instructions
  - Development workflow
  - Production deployment
  - WhatsApp VPS setup
  - Troubleshooting guide

- **WhatsApp VPS Deployment Guide** (separate file: WHATSAPP_DEPLOYMENT.md):
  - VPS provisioning (OCI/Writer Cloud)
  - Docker installation on Ubuntu
  - WhatsApp service deployment (docker run with volumes)
  - QR code authentication
  - Monitoring and logs

## 5. Internal Responsibilities

1. **Docker Compose File Creation**:
   - Define all 5 services (backend, frontend, celery-worker, celery-beat, redis)
   - Configure port mappings (8000, 3000, 6379)
   - Set up service dependencies (depends_on)
   - Configure environment variables (from .env file)
   - Set up Docker network (default bridge network)
   - Add health checks for critical services

2. **Backend Dockerfile**:
   - Use python:3.10-slim base image
   - Install system dependencies (curl for health check)
   - Copy requirements.txt and install Python packages
   - Copy application code
   - Expose port 8000
   - CMD: uvicorn app.main:app --host 0.0.0.0 --port 8000

3. **Frontend Dockerfile**:
   - Use node:18-alpine base image
   - Copy package.json and package-lock.json
   - Run npm install
   - Copy application code
   - Run npm run build (production build)
   - Expose port 3000
   - CMD: npm start

4. **Redis Configuration**:
   - Use official Redis Alpine image (lightweight)
   - Enable AOF persistence (--appendonly yes)
   - Mount volume for data persistence
   - Expose port 6379

5. **Environment Configuration**:
   - Create .env.example with all variables documented
   - Add comments explaining each variable
   - Never commit .env to Git (.gitignore)
   - Provide setup script (copy .env.example to .env)

6. **Volume Configuration**:
   - Redis data: ./redis-data:/data (persist across restarts)
   - WhatsApp service (VPS): ./.wwebjs_auth and ./data volumes

7. **Health Checks**:
   - Backend: curl http://localhost:8000/health
   - Frontend: curl http://localhost:3000 (Next.js health)
   - Redis: redis-cli ping
   - Celery: celery -A app.celery inspect ping (in worker container)

8. **README Documentation**:
   - Prerequisites: Docker 20+, Docker Compose 2+, Supabase account, Clerk account, Google Cloud project (OAuth for API access)
   - Quick start: git clone, cd project, cp .env.example .env, edit .env, docker-compose up
   - Clerk setup: Create Clerk application, configure social login (Google), get API keys
   - Google OAuth setup: Step-by-step guide (Google Cloud Console, enable APIs, create credentials for API access)
   - Commands: docker-compose up, down, logs, restart, ps
   - Troubleshooting: Common errors (database connection failed, Redis connection refused, Clerk JWT verification errors, Google OAuth errors)

9. **WhatsApp VPS Deployment Guide**:
   - VPS provisioning: Choose Ubuntu 22.04, 1GB RAM minimum
   - Install Docker: apt-get update, install docker.io, enable docker service
   - Clone repository or copy WhatsApp service code
   - Create .env file with BACKEND_API_URL, GEMINI_API_KEY, WHATSAPP_API_KEY
   - Run WhatsApp service: docker run -d --name whatsapp-service -v ./.wwebjs_auth:/app/.wwebjs_auth -v ./data:/app/data whatsapp-service
   - QR code authentication: docker logs whatsapp-service (scan QR code)
   - Monitoring: docker logs -f whatsapp-service, docker stats

10. **Troubleshooting Guide**:
    - Database connection errors: Check SUPABASE_URL and SUPABASE_KEY, verify Supabase project active
    - Redis connection errors: Check redis service running (docker ps), check REDIS_URL
    - Clerk auth errors: Check CLERK_SECRET_KEY, verify Clerk application configured
    - Google OAuth errors: Re-authenticate via "Connect Google Account" button, check token expiration
    - Celery worker not processing tasks: Check Redis connection, check worker logs
    - Frontend not connecting to backend: Check NEXT_PUBLIC_API_URL, check CORS settings
    - WhatsApp QR code not displaying: Check logs, ensure session not already active

## 6. Dependencies

**External Services:**
- Supabase (database, already set up)
- Google Cloud (OAuth credentials)
- VPS provider (OCI/Writer Cloud for WhatsApp service)

**Tools:**
- Docker 20+
- Docker Compose 2+
- Git

**Ticket Dependencies:**
- All previous tickets (complete application required)

**Blocks:**
- None (final deployment ticket)

## 7. Execution Model

**Type**: Infrastructure Configuration and Documentation

**Execution**:
- Docker Compose: docker-compose up -d (starts all services)
- Services run continuously until stopped (docker-compose down)

**Development Workflow**:
- Edit code locally
- Rebuild service: docker-compose build backend
- Restart service: docker-compose restart backend
- View logs: docker-compose logs -f backend

**Production Deployment**:
- Deploy on single machine (Docker Compose)
- Or deploy on cloud platform (AWS ECS, Google Cloud Run, Fly.io)

## 8. Failure Handling

**Service Startup Failures**:
- Backend fails: Check SUPABASE_URL/SUPABASE_KEY, CLERK_SECRET_KEY, check logs (docker-compose logs backend)
- Celery worker fails: Check REDIS_URL, check backend connection
- Redis fails: Check volume permissions, check port conflicts

**Runtime Failures**:
- Container crashes: Docker Compose auto-restarts (unless explicitly stopped)
- Redis data loss: Persistent volume prevents data loss on restart

**Network Errors**:
- Services can't communicate: Check Docker network (docker network ls)
- Frontend can't reach backend: Check port mappings, check NEXT_PUBLIC_API_URL

**Resource Exhaustion**:
- Out of memory: Increase Docker memory limit, add resource limits in docker-compose.yml
- Disk full: Clean up old Docker images/volumes (docker system prune)

## 9. Observability

**Logs**:
- View all logs: docker-compose logs
- View specific service: docker-compose logs backend
- Follow logs: docker-compose logs -f backend
- Tail logs: docker-compose logs --tail=100 backend

**Container Status**:
- List containers: docker-compose ps
- Check health: docker-compose ps (shows health status)

**Resource Usage**:
- docker stats (CPU, memory, network)

**Health Checks**:
- Backend: curl http://localhost:8000/health
- Redis: docker exec -it redis redis-cli ping

## 10. Security Considerations

**Environment Variables:**
- Never commit .env to Git
- Use secrets management in production (Docker secrets, AWS Secrets Manager)
- Rotate secrets periodically

**Network Security:**
- Expose only necessary ports (8000, 3000, 6379)
- Use firewall rules in production (ufw, iptables)
- Restrict Redis access (bind to 127.0.0.1 or use authentication)

**Container Security:**
- Use official base images (python:3.10-slim, node:18-alpine, redis:7-alpine)
- Scan images for vulnerabilities (docker scan, Trivy)
- Run containers as non-root user

**Data Security:**
- Database credentials in .env (encrypted at rest by Supabase)
- Redis AOF persistence (encrypted volume in production)

## 11. Scaling Considerations

**Current MVP:**
- Single machine deployment
- No load balancing
- Sufficient for single user

**Vertical Scaling:**
- Increase Docker resource limits (memory, CPU)
- Use faster machine

**Horizontal Scaling** (future):
- Multiple backend instances behind load balancer (Nginx, Traefik)
- Multiple Celery workers (add replicas in docker-compose.yml)
- Redis Cluster for high availability

**Production Considerations:**
- Use managed services (AWS ECS, Google Cloud Run)
- Implement auto-scaling (scale based on CPU/memory)
- Use CDN for frontend (Vercel, Cloudflare)

## 12. Future Extensions

**Phase 2:**
- Kubernetes deployment (Helm charts)
- CI/CD pipeline (GitHub Actions, GitLab CI)
- Monitoring and alerting (Prometheus, Grafana, Alertmanager)
- Log aggregation (ELK stack, Loki)
- Distributed tracing (Jaeger, Zipkin)
- Database backup automation
- SSL certificate automation (Let's Encrypt, Cert-Manager)
- Blue-green deployments
- Canary releases
- Multi-region deployment (disaster recovery)
