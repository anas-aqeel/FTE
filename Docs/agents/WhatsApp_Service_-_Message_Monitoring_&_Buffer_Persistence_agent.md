# Agent Specification: WhatsApp Service - Message Monitoring & Buffer Persistence

## 1. Purpose

Build a Node.js service that runs 24/7 on a VPS to monitor all WhatsApp messages (private chats and group chats) using whatsapp-web.js, accumulating messages in hourly batches and persisting them to local disk (SQLite or append-only files) to ensure no data loss during container restarts.

## 2. Scope

**In Scope:**
- Node.js project setup with whatsapp-web.js library
- WhatsApp Web connection and QR code authentication
- Session persistence to avoid repeated QR code scans
- 24/7 continuous message monitoring for all chats (private and group)
- In-memory message buffering with hourly batches per chat
- Hourly buffer persistence to local disk (SQLite database or append-only JSON files)
- Buffer recovery on service restart (resume from persisted data)
- Environment configuration for API keys and backend URL
- Dockerfile for VPS deployment (OCI/Writer Cloud)
- Logging and monitoring infrastructure
- Error handling and automatic reconnection logic

**Out of Scope:**
- Hourly summarization logic (handled by separate ticket)
- Gemini API integration (handled by separate ticket)
- Backend communication and data ingestion (handled by separate ticket)
- Data filtering or importance scoring
- Frontend components

## 3. Inputs

- **WhatsApp Web Session**:
  - QR code authentication (one-time setup, then session persists)
  - User's WhatsApp account credentials via QR scan
- **Incoming WhatsApp Messages**:
  - Message content (text, media metadata)
  - Sender information (name, phone number)
  - Chat information (chat name, is_group flag)
  - Timestamp (message received_at)
- **Environment Variables**:
  - BACKEND_API_URL: FastAPI backend base URL (for future use)
  - GEMINI_API_KEY: Google Gemini API key (for future use)
  - WHATSAPP_API_KEY: Shared secret for authenticating to backend (for future use)
  - DATA_DIR: Local directory for buffer persistence (default: ./data)
  - LOG_LEVEL: Logging verbosity (default: info)
- **Persisted Session Data**:
  - WhatsApp session tokens (in .wwebjs_auth directory)
  - Hourly message buffers (in DATA_DIR/buffers/)

## 4. Outputs

- **Persisted Message Buffers**:
  - Local SQLite database (messages.db) OR
  - Append-only JSON files (one file per chat per hour: {chat_name}_{timestamp}.json)
- **Accumulated Hourly Batches**:
  - Structure per chat:
    ```json
    {
      "chat_name": "Project Group",
      "is_group": true,
      "messages": [
        {"sender": "Alice", "content": "...", "timestamp": "..."},
        {"sender": "Bob", "content": "...", "timestamp": "..."}
      ],
      "batch_start_time": "2026-02-17T10:00:00Z",
      "batch_end_time": "2026-02-17T11:00:00Z"
    }
    ```
- **WhatsApp Session Persistence**:
  - Session tokens stored in .wwebjs_auth directory
  - Survives container restarts
- **Logs**:
  - Message monitoring status (connected, disconnected, reconnecting)
  - Messages received count per chat
  - Buffer persistence operations (writes, reads)
  - Errors and warnings (connection failures, disk write failures)
- **Health Check Endpoint** (optional):
  - HTTP endpoint on port 8080: GET /health
  - Returns: {status: "connected", uptime: 12345, message_count: 567}

## 5. Internal Responsibilities

1. **WhatsApp Web Connection**:
   - Initialize whatsapp-web.js client
   - Display QR code on first run (log to console or save as image)
   - Authenticate user session
   - Persist session tokens to .wwebjs_auth directory
   - Automatically restore session on restart (avoid QR re-scan)

2. **24/7 Message Monitoring**:
   - Listen to 'message' event from whatsapp-web.js
   - Capture all incoming messages from private chats
   - Capture all incoming messages from group chats
   - Extract message metadata (sender, content, timestamp, chat info)
   - Handle different message types (text, media, system messages)

3. **Hourly Buffer Management**:
   - Maintain in-memory buffer per chat (Map or object keyed by chat_name)
   - Append each incoming message to appropriate chat buffer
   - Track batch_start_time (first message in hour) and batch_end_time (last message)
   - Flush buffers at top of each hour (e.g., at :00 minutes)

4. **Buffer Persistence to Disk**:
   - **Option A (SQLite)**:
     - Create SQLite database (messages.db)
     - Define schema: (id, chat_name, is_group, sender, content, timestamp, batch_id)
     - Insert messages as they arrive
     - Query batches at top of hour for processing
   - **Option B (Append-Only Files)**:
     - Create JSON file per chat per hour: data/buffers/{chat_name}_{YYYY-MM-DD_HH}.json
     - Append messages to file as JSON lines (one message per line)
     - Read files at top of hour for processing
   - **Recommendation**: Use SQLite for easier querying and atomic operations

5. **Buffer Recovery on Restart**:
   - On service startup, check DATA_DIR for persisted buffers
   - Load in-progress hourly buffer (current hour) back into memory
   - Resume monitoring without data loss
   - Log recovery status (X messages recovered from Y chats)

6. **Hourly Flush Logic**:
   - Set up interval timer (check every minute or use cron-like scheduler)
   - At top of hour (HH:00), for each chat buffer:
     - Persist batch to disk if not already done
     - Clear in-memory buffer for that chat
     - Start new batch for next hour
   - Log flush operations (X chats flushed, Y messages persisted)

7. **Session Monitoring and Reconnection**:
   - Listen to 'disconnected' event from whatsapp-web.js
   - Attempt automatic reconnection (exponential backoff)
   - Log connection status changes
   - Alert if connection fails for extended period (>5 minutes)

8. **Error Handling**:
   - Handle message parsing errors (malformed messages)
   - Handle disk write failures (disk full, permission issues)
   - Handle WhatsApp Web API errors (rate limits, authentication failures)
   - Log all errors with context (chat_name, timestamp, error message)
   - Continue monitoring despite individual message failures

9. **Logging and Monitoring**:
   - Structured logging (JSON format) with levels (info, warn, error)
   - Log key events: startup, QR code displayed, connected, message received, buffer flushed, error
   - Metrics tracking: messages_received_total, chats_monitored, buffer_size, connection_status
   - Optional: Expose Prometheus metrics endpoint

10. **Health Check Endpoint** (optional for production):
    - Simple HTTP server on port 8080
    - GET /health returns JSON with status, uptime, message_count
    - Enables external monitoring (Kubernetes liveness probe, uptime monitor)

## 6. Dependencies

**External Services:**
- WhatsApp Web (user's WhatsApp account)
- VPS instance for deployment (OCI/Writer Cloud)

**Libraries and Tools:**
- Node.js 18+ runtime
- whatsapp-web.js library (npm package)
- SQLite3 (if using SQLite for buffer persistence) OR fs module (if using JSON files)
- Docker and Docker Compose (for containerization)
- Optional: Puppeteer (required by whatsapp-web.js for browser automation)

**Infrastructure:**
- VPS with persistent disk storage (for .wwebjs_auth and DATA_DIR)
- Docker volume for session and buffer persistence
- Network access to WhatsApp Web servers (ensure firewall allows outbound HTTPS)

**Ticket Dependencies:**
- None - This ticket can be developed independently in parallel with database setup

**Blocks:**
- WhatsApp_Service_-_Hourly_Summarization_&_Backend_Integration ticket (requires buffers created by this ticket)

## 7. Execution Model

**Type**: Long-Running Background Service (24/7 Daemon)

**Lifecycle**:
1. **Startup**:
   - Load environment variables
   - Initialize whatsapp-web.js client
   - Check for existing session in .wwebjs_auth
   - If session exists: restore and connect
   - If no session: display QR code and wait for user scan
   - Load persisted buffers from disk (recover in-progress batches)
   - Start message monitoring loop
   - Start hourly flush timer

2. **Runtime (Continuous)**:
   - Listen to WhatsApp message events
   - Buffer messages in memory per chat
   - Persist messages to disk as they arrive (or batch persist)
   - Flush buffers at top of each hour
   - Monitor connection health and reconnect if needed

3. **Shutdown**:
   - Gracefully close WhatsApp connection
   - Flush all in-memory buffers to disk
   - Close SQLite database connection (if used)
   - Log shutdown completion

**Concurrency**: Single-threaded Node.js event loop (no parallelism needed for MVP)

**Restart Policy**: Always restart (Docker restart: always or Kubernetes restartPolicy: Always)

**State Persistence**: Critical for avoiding data loss:
- .wwebjs_auth directory (session tokens) must be mounted as Docker volume
- DATA_DIR (message buffers) must be mounted as Docker volume

**Deployment Environment**: VPS (OCI/Writer Cloud) with Docker containerization

## 8. Failure Handling

**WhatsApp Disconnection**:
- **Cause**: Network instability, WhatsApp server issues, session expiration
- **Handling**:
  - Automatic reconnection with exponential backoff (1s, 2s, 4s, 8s, max 60s)
  - Preserve in-memory buffers during reconnection
  - Log disconnection and reconnection attempts
  - Alert if reconnection fails after 5 minutes

**Session Expiration**:
- **Cause**: WhatsApp session invalidated (user logged out on phone, session timeout)
- **Handling**:
  - Detect session invalidation error
  - Log error and alert (requires manual intervention)
  - Generate new QR code for re-authentication
  - Preserve buffered data during re-authentication

**Disk Write Failures**:
- **Cause**: Disk full, permission issues, I/O errors
- **Handling**:
  - Catch disk write exceptions
  - Log error with details (available disk space, error message)
  - Attempt retry with exponential backoff
  - If persistent failure: alert and continue buffering in memory (risk of data loss on crash)

**Message Parsing Errors**:
- **Cause**: Malformed messages, unsupported message types, API changes
- **Handling**:
  - Catch parsing exceptions
  - Log error with message metadata (chat, timestamp, raw message)
  - Skip problematic message and continue monitoring
  - Increment error counter for monitoring

**Container Restarts**:
- **Cause**: OOM, crash, manual restart, VPS reboot
- **Handling**:
  - On startup, load persisted buffers from disk
  - Resume monitoring without data loss
  - Log recovery status (messages recovered, chats resumed)
  - Continue from last persisted state

**Memory Overflow**:
- **Cause**: Extremely high message volume in a single hour (thousands of messages in busy groups)
- **Handling**:
  - Monitor memory usage (process.memoryUsage())
  - If memory exceeds threshold (e.g., 80% of available), flush buffers immediately
  - Log warning and trigger early flush
  - Consider implementing buffer size limits (max messages per chat per hour)

## 9. Observability

**Logging**:
- **Startup Events**:
  - Service started (timestamp, version)
  - Environment loaded (BACKEND_API_URL redacted)
  - Session status (existing session restored or new QR code required)
  - Buffer recovery (X messages loaded from Y chats)
- **Runtime Events**:
  - WhatsApp connected (timestamp, session age)
  - Message received (chat_name, sender, is_group, message_length)
  - Buffer flushed (chat_name, message_count, batch_start, batch_end)
  - Disk write operations (file created, database updated)
- **Error Events**:
  - Connection lost (timestamp, reason)
  - Reconnection attempt (attempt_number, backoff_delay)
  - Disk write failure (error message, available_space)
  - Message parsing error (chat_name, error_message)

**Metrics** (if Prometheus integration added):
- messages_received_total (counter, labels: chat_name, is_group)
- chats_monitored_count (gauge)
- buffer_size_bytes (gauge, labels: chat_name)
- connection_status (gauge, values: 0=disconnected, 1=connected, 2=reconnecting)
- buffer_flush_count (counter)
- disk_write_errors (counter)
- session_uptime_seconds (gauge)

**Health Check** (HTTP endpoint on port 8080):
```json
{
  "status": "connected",
  "uptime_seconds": 123456,
  "messages_received_total": 5432,
  "chats_monitored": 15,
  "buffer_size_mb": 12.5,
  "last_message_timestamp": "2026-02-17T14:35:22Z"
}
```

**Debugging**:
- Detailed message logs (enable with LOG_LEVEL=debug)
- Raw message dumps on parsing errors (saved to debug/ directory)
- Connection state logs (QR code displayed, authentication successful, session restored)
- Buffer state inspection (list all active buffers with message counts)

## 10. Security Considerations

**WhatsApp Session Security**:
- Session tokens stored in .wwebjs_auth directory are highly sensitive (equivalent to account credentials)
- Docker volume for .wwebjs_auth must have restricted permissions (chmod 600)
- Never commit .wwebjs_auth to Git (add to .gitignore)
- Session tokens grant full access to user's WhatsApp account

**Data Privacy**:
- Buffered messages contain sensitive personal information
- DATA_DIR (message buffers) must be on encrypted disk (VPS provider encryption at rest)
- Consider encrypting buffer files at application level if VPS lacks disk encryption
- Implement data retention policy (delete buffers after successful processing)

**Secrets Management**:
- WHATSAPP_API_KEY stored in environment variable (never hardcode)
- GEMINI_API_KEY stored in environment variable
- Use Docker secrets or Kubernetes secrets for production deployment
- Never log API keys or session tokens

**Network Security**:
- WhatsApp Web connections use TLS encryption (handled by whatsapp-web.js)
- Backend API calls (future) must use HTTPS
- VPS firewall should only allow outbound HTTPS (ports 443, 80)
- No inbound ports required except optional health check endpoint (8080)

**Container Security**:
- Run container as non-root user
- Use minimal base image (node:18-alpine)
- Scan Docker image for vulnerabilities (Trivy, Snyk)
- Keep dependencies updated (npm audit, Dependabot)

**Access Control**:
- VPS SSH access limited to authorized users
- Docker volumes not world-readable
- Consider using read-only root filesystem (except for .wwebjs_auth and DATA_DIR)

## 11. Scaling Considerations

**Current MVP Constraints**:
- Single user (one WhatsApp account)
- Single VPS instance
- No horizontal scaling required
- Designed for personal use (dozens of chats, hundreds of messages per hour)

**Performance Characteristics**:
- Memory usage: ~100-500 MB depending on message volume
- Disk usage: ~1-10 MB per hour of messages (varies with chat activity)
- CPU usage: Minimal (event-driven architecture)
- Network usage: Minimal (WhatsApp Web polling + message download)

**Vertical Scaling**:
- Increase VPS RAM if monitoring very high-volume group chats (thousands of messages/hour)
- Increase disk size if long-term buffer retention required
- Upgrade to VPS with SSD for faster disk I/O (important for high-frequency writes)

**Future Multi-User Scaling** (Phase 2):
- Run separate container instance per user
- Use orchestration (Kubernetes, Docker Swarm) to manage multiple instances
- Implement centralized logging and monitoring (ELK stack, Grafana)
- Use shared volume or object storage for buffer persistence

**Bottlenecks**:
- Disk I/O: High message volume can cause disk write bottleneck (mitigate with SSD or batch writes)
- Memory: Large buffers (thousands of messages) can cause OOM (mitigate with buffer size limits or early flush)
- WhatsApp API Rate Limits: Unofficial API may have rate limits (monitor for errors)

**Optimization Opportunities**:
- Batch disk writes (write every 100 messages instead of every message)
- Compress message buffers (gzip JSON files)
- Use database connection pooling if using SQLite
- Implement circular buffer (drop oldest messages if buffer exceeds limit)

## 12. Future Extensions

**Phase 2 Enhancements**:
- **Multi-User Support**: Run separate service instances per user with centralized orchestration
- **Advanced Filtering**: Pre-filter messages based on sender, keywords, or chat type before buffering
- **Media Handling**: Download and store media attachments (images, videos, documents)
- **Real-Time Processing**: Stream messages to backend immediately instead of hourly batches
- **Encryption**: Encrypt message buffers at application level (AES-256) for enhanced privacy
- **Backup and Archival**: Automatically backup buffers to cloud storage (S3, Google Cloud Storage)

**Advanced Features**:
- **Message Search**: Full-text search across historical messages
- **Conversation Threading**: Group messages by conversation thread for better context
- **Sentiment Analysis**: Real-time sentiment detection on messages (flag urgent/angry messages)
- **Spam Detection**: Filter out spam messages before buffering
- **Group Chat Analytics**: Track message frequency, top senders, peak activity times
- **Custom Webhooks**: Send real-time webhooks on specific keywords or senders
- **WhatsApp Business API Migration**: Migrate to official WhatsApp Business API for production deployments

**Monitoring and Alerting**:
- **Proactive Alerting**: Alert on connection failures, disk space low, buffer overflow
- **Performance Dashboards**: Grafana dashboards for real-time metrics (message rate, buffer size, connection status)
- **Log Aggregation**: Centralized logging with Elasticsearch or Loki
- **Distributed Tracing**: OpenTelemetry integration for end-to-end tracing

**Reliability Improvements**:
- **High Availability**: Run multiple instances with failover (requires session sharing or stateless design)
- **Data Replication**: Replicate buffers to multiple storage backends for disaster recovery
- **Circuit Breaker**: Implement circuit breaker pattern for external dependencies (WhatsApp Web, disk I/O)
- **Rate Limiting**: Implement rate limiting for disk writes to prevent I/O saturation
