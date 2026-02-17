# Generate Documentation Files & Resolve Conflicts

## Objective

Create a Python script that generates all documentation files (specs, tickets, test cases) in a `docs/` folder structure, ready for Git commit and GitHub publishing.

## Scope

**In Scope:**
- Create `generate_docs.py` script that:
  - Creates `docs/` folder structure (specs/, tickets/)
  - Generates markdown files for all specs:
    - Epic Brief
    - Tech Plan
    - Test Case Scenarios
    - Data Ingestion Architecture explanation
  - Generates markdown files for all 15 tickets
  - Creates README.md with navigation
  - Creates TICKET_BREAKDOWN.md with dependency graph
  - Creates CONFLICTS_RESOLVED.md documenting all resolved conflicts
- Script should be idempotent (can run multiple times)
- All conflicts from `conflicts.md` resolved in generated docs

**Out of Scope:**
- Actual implementation of tickets
- Automated testing framework

## Conflicts Resolved

All 8 conflicts from the conflicts analysis have been resolved:

1. ✅ **Chat endpoint path** - Standardized to `POST /conversations/{id}/messages`
2. ✅ **Celery data flow** - Updated sequence diagram to show Celery calling APIs directly
3. ✅ **Missing raw_classroom_announcements table** - Added to schema
4. ✅ **Celery Gmail dependency** - Removed incorrect dependency on sync status endpoint
5. ✅ **WhatsApp endpoint ownership** - Clarified in ticket scope (both Node.js and FastAPI parts)
6. ✅ **API key auth for WhatsApp** - Added to FastAPI Auth ticket
7. ✅ **WhatsApp env vars** - Added WHATSAPP_API_KEY to deployment section
8. ✅ **Frontend /api/ prefix** - Clarified: no prefix, direct backend calls

## Spec References

- `spec:69889a16-05f6-4f4a-bf1a-8cf0daec03b5/381a7a45-1b56-45fb-a2eb-bc0e7799dd92` (Epic Brief)
- `spec:69889a16-05f6-4f4a-bf1a-8cf0daec03b5/4eb58cf0-bbfc-42e3-acd5-9fe60dd4f30b` (Tech Plan)
- All 15 tickets

## Acceptance Criteria

- [ ] `generate_docs.py` script created
- [ ] Running script creates `docs/` folder structure
- [ ] All spec files generated in `docs/specs/`
- [ ] All ticket files generated in `docs/tickets/`
- [ ] README.md with navigation created
- [ ] TICKET_BREAKDOWN.md with dependency graph created
- [ ] CONFLICTS_RESOLVED.md documenting all fixes created
- [ ] All mermaid diagrams included in generated files
- [ ] Script is idempotent (can run multiple times safely)
- [ ] Generated files are valid markdown
- [ ] All conflicts resolved in generated documentation

## Dependencies

None (documentation task)