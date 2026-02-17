# Personal Assistant AI Agent

> An intelligent conversational assistant that helps students stay on top of their academic commitments by aggregating information from Gmail, Google Classroom, and WhatsApp.

[![Status](https://img.shields.io/badge/status-in%20development-yellow)](https://github.com)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
- [Technology Stack](#technology-stack)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
- [How It Works](#how-it-works)
- [Data Flow](#data-flow)
- [Security](#security)
- [Documentation](#documentation)
- [Roadmap](#roadmap)
- [Contributing](#contributing)

---

## Overview

### The Problem

Students managing academic commitments face critical challenges:

- **Information Fragmentation**: Critical data scattered across Gmail (emails, calendar), Google Classroom (assignments, announcements), and WhatsApp (group discussions)
- **Missed Deadlines**: Important announcements lost in high-volume group chats with hundreds of messages
- **Time Waste**: Significant time spent searching through emails and messages to find specific information
- **Social Impact**: Embarrassment from being uninformed about assignments, having to ask classmates repeatedly

### The Solution

A unified AI-powered interface that:

✅ **Aggregates** all academic information from multiple sources automatically
✅ **Surfaces** important deadlines and announcements proactively
✅ **Answers** natural language queries about schedules, deadlines, and project details
✅ **Filters** signal from noise in high-volume group chats
✅ **Provides** a reliable, always-up-to-date view of commitments

---

## Features

### Core Capabilities

#### 🤖 Conversational Interface
- Natural language queries: "What's my schedule today?", "What are my upcoming deadlines?"
- Multi-conversation thread support (like ChatGPT)
- Context-aware responses using conversation history
- Powered by Google Gemini AI

#### 📧 Gmail Integration
- Automatic hourly email ingestion
- AI-based email classification and importance scoring
- Deadline extraction from email content
- Calendar event detection from meeting invites

#### 📚 Google Classroom Integration
- Automatic assignment and announcement ingestion
- Due date tracking
- Course-organized data structure
- Assignment status management

#### 💬 WhatsApp Integration
- 24/7 message monitoring (via VPS)
- Hourly batch summarization with Gemini AI
- Configurable group allowlist (cost management)
- Private chat monitoring
- Deadline and announcement extraction from group chats

#### 🎯 Intelligent Features
- **Dual Storage**: Raw data + AI-processed filtered data
- **Importance Scoring**: AI + rule-based scoring (0-1 scale)
- **Smart Filtering**: Pre-processed data for fast queries
- **Notification System**: Email alerts for urgent items
- **Idempotent Ingestion**: Prevents duplicates using cursors

#### 🔐 Authentication
- **Clerk Auth**: Modern authentication with social login (Google, GitHub) + email/password
- **Google OAuth**: Separate API access for Gmail, Calendar, Classroom
- **Secure Token Management**: Encrypted storage of OAuth refresh tokens

---

## Architecture

### System Overview

```
┌─────────────┐      ┌──────────────┐      ┌─────────────┐
│   Next.js   │─────▶│   FastAPI    │─────▶│  Supabase   │
│   Frontend  │      │   Backend    │      │ PostgreSQL  │
└─────────────┘      └──────────────┘      └─────────────┘
                            │
                     ┌──────┴──────┐
                     ▼              ▼
              ┌──────────┐   ┌──────────┐
              │  Celery  │   │  Gemini  │
              │  Worker  │   │   API    │
              └──────────┘   └──────────┘
                     ▲
                     │
              ┌──────────────┐
              │  WhatsApp    │
              │  Service     │
              │  (VPS)       │
              └──────────────┘
```

### Key Architectural Decisions

1. **Microservices Architecture**: FastAPI Backend, Celery Worker, WhatsApp Service, Next.js Frontend, Supabase Database, Redis Queue

2. **Dual-Data Storage**:
   - **Raw Data**: Unprocessed original data (14-day retention, fallback)
   - **Filtered Data**: AI-processed, labeled, scored (long-term retention, optimized for queries)

3. **Centralized Data Ingestion**: WhatsApp service communicates via Backend API (not directly to DB)

4. **Background Processing**: Celery handles scheduled ingestion tasks separately from API requests

5. **Importance Detection at Ingestion**: AI classification happens once per item (not at query time)

6. **OAuth Token Management**: One-time setup with refresh tokens for automated access

---

## Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Frontend** | Next.js 14 + React + TypeScript | Server-side rendered web application |
| **Backend** | FastAPI (Python) | Async REST API with automatic docs |
| **Database** | Supabase (PostgreSQL) | Managed database with real-time capabilities |
| **Task Queue** | Celery + Redis | Background job processing and scheduling |
| **AI/LLM** | Google Gemini API | Natural language processing and classification |
| **Authentication** | Clerk + Google OAuth | User auth + API access |
| **WhatsApp** | whatsapp-web.js (Node.js) | WhatsApp Web integration |
| **Deployment** | Docker + Docker Compose | Containerization and orchestration |

### APIs Used
- Gmail API
- Google Calendar API
- Google Classroom API
- Google Gemini API
- Clerk API

---

## Project Structure

```
Automation/
├── docs/
│   ├── specs/                          # Project specifications
│   │   ├── Epic_Brief__Personal_Assistant_Agent.md
│   │   ├── Tech_Plan__Personal_Assistant_Agent.md
│   │   ├── OAuth_Setup_Guide.md
│   │   ├── WhatsApp_Group_Identification_Strategy.md
│   │   ├── Test_Case_Scenarios_&_Edge_Cases.md
│   │   └── Gemini_Prompt_Templates.md
│   ├── tickets/                        # Implementation tickets
│   │   ├── Setup_Supabase_Project_&_Core_Tables.md
│   │   ├── FastAPI_Project_Setup_&_Authentication.md
│   │   ├── Celery_Setup_&_Gmail_Ingestion_Worker.md
│   │   ├── Google_Classroom_Ingestion_Worker.md
│   │   ├── WhatsApp_Service_-_Message_Monitoring_&_Buffer_Persistence.md
│   │   ├── WhatsApp_Service_-_Hourly_Summarization_&_Backend_Integration.md
│   │   ├── Conversational_Agent_with_Gemini_Integration.md
│   │   ├── Next.js_Frontend_-_Authentication_&_Layout.md
│   │   ├── Next.js_Frontend_-_Chat_Interface_&_Conversation_Management.md
│   │   └── ...
│   ├── agents/                         # Agent-specific documentation
│   └── CLERK_AUTH_MIGRATION.md         # Authentication migration guide
├── backend/                            # FastAPI application (to be created)
├── frontend/                           # Next.js application (to be created)
├── whatsapp-service/                   # WhatsApp monitoring service (to be created)
└── README.md
```

---

## Getting Started

### Prerequisites

- **Node.js** 18+ and npm
- **Python** 3.11+
- **Docker** and Docker Compose
- **Google Cloud Account** (for Gmail, Calendar, Classroom APIs)
- **Clerk Account** (for authentication)
- **Supabase Account** (for database)
- **Google Gemini API Key**
- **VPS** (for WhatsApp service - OCI/Writer Cloud recommended)

### Installation

#### 1. Clone the Repository

```bash
git clone <repository-url>
cd Automation
```

#### 2. Set Up Supabase

1. Create a new project at [supabase.com](https://supabase.com)
2. Run database migrations (see `docs/tickets/Setup_Supabase_Project_&_Core_Tables.md`)
3. Get your `SUPABASE_URL` and `SUPABASE_KEY`

#### 3. Set Up Clerk Authentication

1. Create account at [clerk.com](https://clerk.com)
2. Create new application
3. Enable Google social login and email authentication
4. Get `CLERK_SECRET_KEY` and `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`

See `docs/specs/OAuth_Setup_Guide.md` for detailed setup.

#### 4. Set Up Google OAuth

1. Create project in [Google Cloud Console](https://console.cloud.google.com)
2. Enable Gmail, Calendar, and Classroom APIs
3. Create OAuth 2.0 credentials
4. Configure consent screen and scopes

See `docs/specs/OAuth_Setup_Guide.md` for detailed setup.

#### 5. Get Gemini API Key

1. Visit [Google AI Studio](https://makersuite.google.com/app/apikey)
2. Create API key for Gemini API

#### 6. Configure Environment Variables

Create `.env` file in project root:

```bash
# Clerk Authentication
CLERK_SECRET_KEY=sk_test_xxxxxxxxxxxxxxxxxxxx
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_xxxxxxxxxxxxxxxxxxxx
CLERK_WEBHOOK_SECRET=whsec_xxxxxxxxxxxx

# Google OAuth (API Access)
GOOGLE_CLIENT_ID=your-client-id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=GOCSPX-xxxxxxxxxxxxx

# Supabase
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_KEY=your-supabase-anon-key

# Gemini API
GEMINI_API_KEY=your-gemini-api-key

# Application
FRONTEND_URL=http://localhost:3000
BACKEND_URL=http://localhost:8000

# Redis
REDIS_URL=redis://localhost:6379

# WhatsApp Service
WHATSAPP_API_KEY=shared-secret-key
WHATSAPP_SERVICE_URL=http://whatsapp-service:3001

# Encryption (for OAuth tokens)
ENCRYPTION_KEY=<generate-32-byte-base64-key>
```

#### 7. Run with Docker Compose

```bash
docker-compose up -d
```

Services will be available at:
- Frontend: http://localhost:3000
- Backend API: http://localhost:8000
- API Docs: http://localhost:8000/docs

#### 8. Set Up WhatsApp Service (VPS)

Deploy WhatsApp service to VPS (see `docs/tickets/WhatsApp_Service_-_Message_Monitoring_&_Buffer_Persistence.md`):

```bash
# On VPS
git clone <repository-url>
cd Automation/whatsapp-service
npm install
npm start
```

Scan QR code to authenticate WhatsApp Web session.

---

## Configuration

### User Settings

Users can configure:

- **Important Senders**: Email addresses/names to prioritize
- **Keyword Rules**: Keywords that indicate important messages
- **WhatsApp Group Allowlist**: Which groups to summarize (cost management)
- **Notification Preferences**: When to receive urgent notifications

### System Settings

- **Data Retention**: Raw data kept for 14 days, filtered data long-term
- **Ingestion Frequency**: Hourly for Gmail/Classroom, continuous for WhatsApp
- **Initial Sync Window**: Last 30 days on first setup
- **WhatsApp Summarization**: Hourly batches with Gemini AI

---

## How It Works

### Data Ingestion Flow

#### Gmail & Google Classroom (Hourly via Celery)

```
1. Celery scheduled task triggers
2. Fetch data from Gmail/Classroom APIs using OAuth tokens
3. Save raw data to database (deduplicated by external IDs)
4. For each item:
   - Call Gemini API for classification and importance scoring
   - Apply rule-based scoring (sender importance, keywords)
   - Combine AI + rule scores
5. Save filtered data to database
6. Create announcement entries for high-importance items
7. Send email notifications for urgent items
```

#### WhatsApp (Continuous on VPS)

```
1. whatsapp-web.js monitors all messages 24/7
2. Accumulate messages in hourly batches (persisted to disk)
3. At top of each hour:
   - Select chats to summarize (all private + allowlisted groups)
   - Call Gemini to extract: summary, key points, deadlines
   - Calculate importance score
4. POST to backend API: /ingest/whatsapp
5. Backend saves raw messages + filtered summary
```

### Conversational Query Flow

```
1. User asks question in chat interface
2. Frontend sends to backend: POST /conversations/{id}/messages
3. Backend:
   - Fetches conversation history
   - Queries filtered data tables (fast, optimized)
   - Falls back to raw data if needed
   - Constructs prompt with relevant data + conversation context
   - Calls Gemini API for natural language response
4. Backend saves conversation turn
5. Frontend displays response
```

### Importance Scoring

Each item receives a score (0-1) based on:

- **AI Classification**: Gemini evaluates content urgency and relevance
- **Sender Importance**: User-configured important senders get higher scores
- **Keyword Matching**: User-configured keywords boost scores
- **Deadline Proximity**: Items with near deadlines scored higher
- **Source Type**: Classroom assignments prioritized over general emails

---

## Data Flow

### Database Schema Overview

#### Raw Data Tables
- `raw_emails` - Original email data from Gmail
- `raw_events` - Calendar events
- `raw_assignments` - Google Classroom assignments
- `raw_classroom_announcements` - Classroom announcements
- `raw_messages` - WhatsApp message batches

#### Filtered/Processed Tables
- `emails` - Classified emails with importance scores
- `events` - Processed calendar events
- `assignments` - Classified assignments with status
- `announcements` - Important announcements (all sources)
- `whatsapp_summaries` - Hourly WhatsApp summaries

#### Core Tables
- `users` - User accounts (linked to Clerk)
- `oauth_tokens` - Encrypted Google OAuth tokens
- `conversations` - Chat conversation threads
- `conversation_messages` - Individual messages
- `sync_state` - Ingestion cursors and timestamps
- `user_settings` - User preferences and configuration

### Data Relationships

- Each user has multiple conversations (1:N)
- Each conversation has multiple messages (1:N)
- Each raw data entry has 0-1 filtered entry (1:0..1)
- Filtered → Raw: `ON DELETE SET NULL` (filtered data survives cleanup)
- User → Data: `ON DELETE CASCADE` (user deletion removes all data)

---

## Security

### Authentication & Authorization

- **User Authentication**: Clerk handles login/signup/sessions with JWT tokens
- **API Access**: Google OAuth 2.0 with refresh tokens for Gmail/Classroom APIs
- **Token Storage**: OAuth tokens encrypted at rest using Fernet encryption
- **API Security**: WhatsApp service authenticates to backend with shared API key

### Data Privacy

- User data isolated by `user_id` foreign keys
- No cross-user data leakage
- OAuth tokens encrypted in database
- No passwords stored (Clerk handles authentication)

### Network Security

- All external communication over HTTPS
- Database not exposed to public internet (managed by Supabase)
- Backend validates all incoming requests
- WhatsApp service uses API key authentication

---

## Documentation

Comprehensive documentation available in `docs/`:

### Specifications
- [Epic Brief](docs/specs/Epic_Brief__Personal_Assistant_Agent.md) - Project overview and problem statement
- [Tech Plan](docs/specs/Tech_Plan__Personal_Assistant_Agent.md) - Architecture and technical decisions
- [OAuth Setup Guide](docs/specs/OAuth_Setup_Guide.md) - Complete authentication setup
- [Test Cases](docs/specs/Test_Case_Scenarios_&_Edge_Cases.md) - Testing scenarios

### Implementation Tickets
- Database setup
- Backend API development
- Frontend implementation
- Celery worker configuration
- WhatsApp service setup
- See `docs/tickets/` for all tickets

### Migration Guides
- [Clerk Auth Migration](docs/CLERK_AUTH_MIGRATION.md) - Migration from custom JWT to Clerk

---

## Roadmap

### Phase 1 (MVP) - Current Focus

✅ Core data ingestion (Gmail, Classroom, WhatsApp)
✅ Conversational query interface
✅ Importance scoring and filtering
✅ Basic notification system
⏳ Implementation in progress

### Phase 2 (Enhancements)

- Push notifications (web + mobile)
- Advanced filtering and search
- Calendar integration (direct Google Calendar API)
- Task management features
- Multi-user support with team features

### Phase 3 (Scale)

- Mobile app (React Native)
- Kubernetes deployment
- Advanced analytics and insights
- Integration with more platforms (Slack, Discord, etc.)
- Voice interface

---

## Contributing

Contributions are welcome! Please follow these guidelines:

1. Read the documentation in `docs/` to understand the architecture
2. Follow existing code style and patterns
3. Write tests for new features
4. Update documentation as needed
5. Submit pull requests with clear descriptions

### Development Workflow

1. Create a feature branch
2. Implement changes
3. Test locally with Docker Compose
4. Update relevant documentation
5. Submit PR for review

---

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

## Acknowledgments

- **Google Gemini** for powerful AI capabilities
- **Clerk** for modern authentication
- **Supabase** for excellent database platform
- **whatsapp-web.js** for WhatsApp integration
- The open-source community for amazing tools

---

## Support

For questions or issues:

1. Check the [documentation](docs/)
2. Review [test cases](docs/specs/Test_Case_Scenarios_&_Edge_Cases.md)
3. Open an issue on GitHub
4. Contact the development team

---

**Built with ❤️ for students struggling with information overload**
