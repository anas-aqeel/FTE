# Agent Specification: Generate Documentation Files & Resolve Conflicts

## 1. Purpose

Create a Python script that generates all project documentation files (Epic Brief, Tech Plan, tickets, test cases, conflict resolutions) in a structured docs/ folder, ready for Git commit and GitHub publishing, serving as the final deliverable for the AcademiQ project planning phase.

## 2. Scope

**In Scope:**
- Create `generate_docs.py` script that:
  - Creates `docs/` folder structure (specs/, tickets/, agents/)
  - Generates markdown files for all specification documents:
    - Epic Brief (epic_brief.md)
    - Tech Plan (tech_plan.md)
    - Test Case Scenarios (test_scenarios.md)
    - Data Ingestion Architecture explanation (data_ingestion_architecture.md)
  - Generates markdown files for all 16 tickets (including this one)
  - Generates markdown files for all 16 agent specifications (including this one)
  - Creates README.md with project overview and navigation
  - Creates TICKET_BREAKDOWN.md with dependency graph (mermaid diagram)
  - Creates CONFLICTS_RESOLVED.md documenting all 8 resolved conflicts
- Script should be idempotent (can run multiple times safely)
- All conflicts from conflicts.md have been resolved and documented
- Include all mermaid diagrams from specifications
- Validate generated markdown (basic syntax checking)
- Git-friendly output (proper line endings, UTF-8 encoding)

**Out of Scope:**
- Actual implementation of tickets (code development)
- Automated testing framework (unit tests, integration tests)
- API documentation generation (handled by FastAPI automatically)
- Code scaffolding or boilerplate generation
- GitHub Actions workflow (CI/CD)

## 3. Inputs

- **Specification Documents** (already created):
  - Epic Brief content
  - Tech Plan content
  - Test Case Scenarios content
  - All 16 ticket descriptions
  - All 16 agent specifications
- **Conflicts Documentation**:
  - List of 8 conflicts identified and resolved:
    1. Chat endpoint path standardization
    2. Celery data flow sequence diagram
    3. Missing raw_classroom_announcements table
    4. Celery Gmail dependency on sync status
    5. WhatsApp endpoint ownership clarification
    6. API key auth for WhatsApp service
    7. WhatsApp environment variables
    8. Frontend /api/ prefix clarification
- **Folder Structure Template**:
  ```
  docs/
  ├── specs/
  │   ├── epic_brief.md
  │   ├── tech_plan.md
  │   ├── test_scenarios.md
  │   └── data_ingestion_architecture.md
  ├── tickets/
  │   ├── Setup_Supabase_Project_&_Core_Tables.md
  │   ├── Implement_Raw_Data_Tables.md
  │   └── ... (all 16 tickets)
  ├── agents/
  │   ├── Setup_Supabase_Project_&_Core_Tables_agent.md
  │   └── ... (all 16 agents)
  ├── README.md
  ├── TICKET_BREAKDOWN.md
  └── CONFLICTS_RESOLVED.md
  ```

## 4. Outputs

- **generate_docs.py Script**:
  - Python script with functions:
    - create_folder_structure()
    - generate_spec_files()
    - generate_ticket_files()
    - generate_agent_files()
    - generate_readme()
    - generate_ticket_breakdown()
    - generate_conflicts_resolved()
    - validate_markdown()
    - main()
- **Generated Documentation Files**:
  - All spec files in docs/specs/
  - All ticket files in docs/tickets/
  - All agent files in docs/agents/
  - README.md with navigation
  - TICKET_BREAKDOWN.md with dependency graph
  - CONFLICTS_RESOLVED.md with all resolutions
- **Script Output** (console logs):
  - Created folder: docs/specs/
  - Generated file: docs/specs/epic_brief.md
  - Generated file: docs/tickets/Setup_Supabase_Project_&_Core_Tables.md
  - Validated markdown: OK (or errors if any)
  - Documentation generation complete!
- **Git Status**:
  - All files ready to commit
  - No binary files (all text/markdown)

## 5. Internal Responsibilities

1. **Folder Structure Creation**:
   - Check if docs/ exists, create if not
   - Create subdirectories: specs/, tickets/, agents/
   - Use os.makedirs(exist_ok=True) for idempotency

2. **Specification Files Generation**:
   - Read Epic Brief content (from memory or file)
   - Write to docs/specs/epic_brief.md
   - Repeat for Tech Plan, Test Scenarios, Data Ingestion Architecture
   - Ensure UTF-8 encoding, LF line endings (not CRLF)

3. **Ticket Files Generation**:
   - For each of 16 tickets:
     - Read ticket content
     - Format as markdown with proper headings
     - Write to docs/tickets/{ticket_name}.md
     - Preserve special characters in filenames (&, -, (, ), commas)

4. **Agent Files Generation**:
   - For each of 16 agents:
     - Read agent specification content
     - Write to docs/agents/{ticket_name}_agent.md
     - Ensure all 12 sections present

5. **README.md Generation**:
   - Project title and description
   - Table of contents with links to specs, tickets, agents
   - Quick start guide
   - Links to mermaid diagrams
   - Contributing guidelines

6. **TICKET_BREAKDOWN.md Generation**:
   - Mermaid dependency graph (flowchart):
     ```mermaid
     graph TD
       A[Setup Supabase] --> B[Raw Tables]
       A --> C[Filtered Tables]
       A --> D[FastAPI Auth]
       B --> E[Gmail Worker]
       ...
     ```
   - Table: Ticket name, Status (Not Started), Dependencies, Estimated effort

7. **CONFLICTS_RESOLVED.md Generation**:
   - List all 8 conflicts with before/after states
   - Resolution details for each conflict
   - Cross-references to updated tickets/specs

8. **Markdown Validation**:
   - Check for common syntax errors (unmatched brackets, invalid headers)
   - Validate mermaid diagram syntax
   - Check for broken internal links
   - Log validation results

9. **Idempotency Handling**:
   - Check if file exists before writing
   - Overwrite files (or skip if --no-overwrite flag)
   - Log "File already exists: skipping" or "Overwriting file"

10. **UTF-8 and Line Endings**:
    - Explicitly set encoding='utf-8' when writing files
    - Use newline='\n' to force LF line endings (cross-platform compatibility)

## 6. Dependencies

**Python Libraries:**
- os (folder operations)
- pathlib (path handling)
- json (data serialization if needed)
- argparse (CLI arguments: --output-dir, --no-overwrite)

**Inputs:**
- Specification documents (Epic Brief, Tech Plan, tickets, agents)
- Conflicts list (8 conflicts with resolutions)

**Ticket Dependencies:**
- None (documentation task, can be done independently)

**Blocks:**
- None (documentation doesn't block development tickets)

## 7. Execution Model

**Type**: One-Time Script Execution

**Usage**:
```bash
python generate_docs.py
# or with arguments:
python generate_docs.py --output-dir ./documentation --no-overwrite
```

**Execution Steps**:
1. Parse command-line arguments
2. Create folder structure
3. Generate all spec files
4. Generate all ticket files
5. Generate all agent files
6. Generate README, TICKET_BREAKDOWN, CONFLICTS_RESOLVED
7. Validate generated markdown
8. Print summary (files created, validation status)

**Execution Time**: <5 seconds for all files

**Idempotency**: Safe to run multiple times (overwrites files or skips based on flag)

## 8. Failure Handling

**Folder Creation Errors:**
- Permission denied: Log error, suggest running with sudo (Linux/Mac) or as admin (Windows)
- Disk full: Log error, suggest freeing disk space

**File Write Errors:**
- Permission denied on file: Log error, skip file, continue with others
- Encoding errors: Log error, use UTF-8 with error handling (replace invalid chars)

**Validation Errors:**
- Invalid markdown syntax: Log warning (don't fail script), list problematic files
- Broken links: Log warning, suggest manual review

**Content Missing:**
- If ticket or agent content not found: Log error, create placeholder file with "TODO: Add content"

## 9. Observability

**Console Output**:
- Progress messages: "Creating folder: docs/specs/"
- File creation: "Generated: docs/tickets/Setup_Supabase_Project_&_Core_Tables.md"
- Validation results: "Validated: 45 files, 0 errors, 2 warnings"
- Summary: "Documentation generation complete! 45 files created."

**Logs** (optional):
- Write detailed log to generate_docs.log
- Include timestamps, errors, warnings

**Validation Report**:
- List files with warnings (invalid markdown, broken links)
- Suggest fixes for common issues

## 10. Security Considerations

**File Permissions:**
- Generated files should be world-readable (chmod 644)
- Script should not require elevated privileges

**Path Traversal Prevention:**
- Validate output directory path (prevent ../../../etc/passwd)
- Use pathlib for safe path joining

**Content Sanitization:**
- No user input in generated content (all from trusted sources)
- No code execution in generated markdown

## 11. Scaling Considerations

**Current MVP:**
- 16 tickets, 16 agents, 4 specs (total ~50 files)
- Small file sizes (< 50KB each)
- Fast execution (< 5 seconds)

**Future Scaling:**
- If project grows to 100+ tickets: optimize file I/O (batch writes)
- Large mermaid diagrams: consider generating SVG images instead of inline mermaid

## 12. Future Extensions

**Phase 2:**
- Automated documentation updates (watch file changes, regenerate)
- GitHub Pages deployment (auto-publish docs on push)
- Interactive dependency graph (HTML + JavaScript)
- Search functionality (full-text search across docs)
- Version control for docs (track changes over time)
- PDF export (generate PDF from markdown)
- Code scaffolding (generate boilerplate code from tickets)
- Test case generation (generate test files from test scenarios)
- API documentation integration (link tickets to FastAPI docs)

**Advanced Features:**
- Multi-language support (i18n for docs)
- Documentation linting (enforce style guide)
- Broken link checker (validate external links)
- Documentation metrics (word count, readability score)
- Automated changelog generation (from Git commits)
- Documentation versioning (multiple versions for different releases)
