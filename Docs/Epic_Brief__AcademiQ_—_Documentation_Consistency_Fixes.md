# Epic Brief: AcademiQ — Documentation Consistency Fixes

## Summary

AcademiQ is an AI-powered academic assistant for university students that aggregates Gmail, Google Classroom, and WhatsApp into a single conversational interface. The project has a complete set of planning documentation — specs, tickets, and agent specs — but a consistency audit revealed 11 gaps that, if left unresolved, would cause bugs, misdirect developers, and create confusion during implementation. This Epic fixes all 11 issues across the `docs/` folder before any code is written. The most impactful change is a full product rebrand from "Personal Assistant Agent" to **AcademiQ** across all 35 markdown files. The remaining fixes close three implementation-breaking gaps (missing OAuth scope, wrong API prefix, incorrect FK reference) and clean up five moderate/minor issues (stale migration status, outdated tech note, incomplete README structure, schema field mismatch, and a code typo).

## Context & Problem

### Who's Affected

Developers picking up any ticket to implement AcademiQ. Every ticket, agent spec, and spec document is a direct input to implementation — inconsistencies in these files translate directly into bugs or wasted effort.

### Current Pain Points

| # | Severity | File(s) | Problem |
|---|----------|---------|---------|
| 1 | 🔴 Critical | All 35 docs | Product called "Personal Assistant Agent" everywhere except `design-prompt-master.md` which uses "AcademiQ" — the intended brand |
| 2 | 🔴 Critical | `WhatsApp_Group_Identification_Strategy.md` | Frontend code snippets use `/api/whatsapp/...` prefix — contradicts the resolved architectural decision that the frontend calls the backend directly with no `/api/` prefix |
| 3 | 🔴 Critical | `FastAPI_Project_Setup_&_Authentication_agent.md` | `gmail.send` scope missing from Google OAuth scope list — breaks the urgent email notification feature (Architectural Decision #12) |
| 4 | 🟡 Moderate | `CLERK_AUTH_MIGRATION.md` | Implementation order still shows items 2–7 as ⏳ pending, but the migration is fully complete as of 2026-02-19 |
| 5 | 🟡 Moderate | `Tech_Plan__Personal_Assistant_Agent.md` | Supabase responsibilities section says "Handle authentication (future: when migrating to Clerk)" — Clerk migration is already done |
| 6 | 🟡 Moderate | `Implement_Filtered_Data_Tables_...md` | Scope incorrectly says "Set up foreign keys to conversations table" — no filtered data table has a FK to `conversations` in the schema |
| 7 | 🟡 Moderate | `README.md` | Project structure missing 3 spec files added on 2026-02-18 and the `docs/agents/` and `docs/prompts/` directories entirely |
| 8 | 🟡 Moderate | `WhatsApp_Group_Identification_Strategy.md` vs `Tech_Plan` | Allowlist group object has `updated_at` in one doc, absent in the other — schema must be consistent |
| 9 | 🟢 Minor | `OAuth_Setup_Guide.md` | Code snippet has `import { useEffect } from 'use'` — should be `from 'react'` |
| 10 | 🟢 Minor | `Celery_Maintenance_Tasks_...md` | Spec reference cites only Architectural Decisions 11 & 12, but Decision #14 (self-notification loop prevention) is directly implemented in this ticket |
| 11 | 🟢 Minor | `Conflicts_Resolution_&_Review_Response.md` | Internal Epic room IDs in cross-references point to a different Epic room — should be noted as stale references |

### The Gap

All 11 issues exist purely in the documentation layer. No code has been written yet, making this the lowest-cost moment to fix them. Leaving them unresolved means the first developer to implement any affected ticket will either build the wrong thing or spend time debugging a discrepancy between docs.