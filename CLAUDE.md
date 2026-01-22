# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a **documentation repository** containing comprehensive planning documents for building a Customer Success Management (CSM) platform (Vitally clone) with HubSpot, Gmail, Slack, and Microsoft Teams integrations. This is NOT an implementation repository - it contains only planning and specification documents.

## Document Structure

The repository follows a hierarchical documentation structure:

1. **README.md** - Entry point and navigation guide
2. **PROJECT_PLAN.md** - Master planning document (36KB)
3. **ROADMAP.md** - 60-week timeline and milestones
4. **INTEGRATIONS_SPEC.md** - Technical specs for all integrations
5. **DATABASE_SCHEMA.md** - Complete database design

### Document Dependencies

```
README.md (navigation)
    ├─→ PROJECT_PLAN.md (references features, architecture, phases)
    │       ├─→ ROADMAP.md (implements timeline from phases)
    │       ├─→ INTEGRATIONS_SPEC.md (details from Integration Requirements)
    │       └─→ DATABASE_SCHEMA.md (implements data models)
    └─→ All other documents (provides overview)
```

## Key Principles

### 1. Documentation Hierarchy
- **PROJECT_PLAN.md** is the source of truth for features and architecture
- Other documents elaborate on specific sections from PROJECT_PLAN.md
- README.md must stay in sync with all documents as a navigation layer

### 2. Cross-Document Consistency
When updating features or timelines, ensure consistency across:
- PROJECT_PLAN.md "Implementation Phases" → ROADMAP.md phase breakdown
- PROJECT_PLAN.md "Database Schema Overview" → DATABASE_SCHEMA.md full schema
- PROJECT_PLAN.md "Integration Requirements" → INTEGRATIONS_SPEC.md detailed specs
- README.md "Key Milestones" → ROADMAP.md milestone table

### 3. Document Metadata
All major documents include metadata footer:
```
---
**Document Version**: X.X
**Last Updated**: YYYY-MM-DD
**Status**: Draft/Ready for Review/Approved
```

Always update these when modifying documents.

## Working with Specific Documents

### PROJECT_PLAN.md
- **Sections**: 12 numbered sections from Executive Summary to Next Steps
- **Tech Stack**: Section 3.2 - only update if there are significant architectural changes
- **Phases**: Section 5 - 12 phases over 60 weeks, maps to ROADMAP.md
- **Sources**: Bottom of document - maintain research citation links

### ROADMAP.md
- **Timeline Format**: Quarterly breakdown (Q1-Q5) with weekly sprint detail
- **Milestone Table**: Week-based checkboxes (🔲) - keep in sync with README.md
- **Resource Allocation Table**: Section near end - shows team allocation per phase
- **Status Updates**: Milestone checkboxes should be updated as ✅ when completed

### INTEGRATIONS_SPEC.md
- **Structure**: One major section per integration (HubSpot, Gmail, Slack, Teams)
- **Code Examples**: TypeScript/JavaScript examples for OAuth, API calls, webhooks
- **Subsections**: Authentication → Data Sync → Features → Implementation for each
- **Integration Architecture**: Section 5 - unified architecture across all integrations

### DATABASE_SCHEMA.md
- **Schema Format**: SQL DDL statements with CREATE TABLE
- **Indexes**: Always include relevant indexes after table definitions
- **Relationships**: Maintain FK constraints and cascade rules
- **Sample Queries**: End of document - add new queries when new features require them

## Updating the Plan

### Adding a New Feature
1. Add feature description to PROJECT_PLAN.md § 2 (Core Features)
2. Determine which phase it belongs to in PROJECT_PLAN.md § 5
3. Add to appropriate phase in ROADMAP.md with week allocation
4. If database changes needed, update DATABASE_SCHEMA.md
5. If integration changes needed, update INTEGRATIONS_SPEC.md
6. Update README.md "Core Features" section

### Modifying Timeline
1. Update phase breakdown in PROJECT_PLAN.md § 5
2. Update detailed sprint breakdown in ROADMAP.md
3. Update milestone table in both ROADMAP.md and README.md
4. Check "Resource Allocation by Phase" table in ROADMAP.md

### Adding an Integration
1. Add integration details to PROJECT_PLAN.md § 4
2. Create new major section in INTEGRATIONS_SPEC.md with subsections:
   - Authentication (OAuth flow)
   - Data Sync Strategy
   - API endpoints and mapping
   - Webhook configuration
   - Code examples
3. Add new phase to ROADMAP.md if needed
4. Update integration count in README.md

### Database Schema Changes
1. Add/modify tables in DATABASE_SCHEMA.md PostgreSQL Schema section
2. Update Entity Relationship Diagram ASCII art
3. Add sample queries if new query patterns are needed
4. Update PROJECT_PLAN.md § 3.3 "Database Schema Overview" if major changes
5. Consider impact on INTEGRATIONS_SPEC.md data mapping sections

## Document Formatting

### Markdown Conventions
- Use `###` for main sections, `####` for subsections
- Code blocks: Use triple backticks with language identifier (```typescript, ```sql)
- Tables: Use GitHub-flavored markdown tables with alignment
- Links: Use reference-style links for external sources, inline for internal docs
- Emojis: Use sparingly in headers for visual navigation (📚, 🎯, 🏗️, etc.)

### Technical Content
- **SQL**: Use uppercase for keywords, lowercase for identifiers
- **TypeScript/JavaScript**: Use async/await patterns, not callbacks
- **Architecture Diagrams**: Use ASCII art boxes and arrows (┌─┐│└┘├┤)
- **API Examples**: Show full request/response with HTTP method and URL

### Section Numbering
- PROJECT_PLAN.md: Use numbered sections (1., 2., 3.)
- ROADMAP.md: Use phase numbers (Phase 1, Phase 2) and Q1-Q5
- INTEGRATIONS_SPEC.md: Use numbered sections for navigation
- DATABASE_SCHEMA.md: Use numbered sections for table categories

## Research Sources

All planning is based on:
- Vitally platform research (primary CSM competitor)
- Official API documentation (HubSpot, Gmail, Slack, Microsoft Graph)
- Industry best practices for CSM platforms

When adding new features or integrations:
- Research actual platform capabilities (don't assume)
- Include source links in PROJECT_PLAN.md "Research Sources" section
- Verify API capabilities before documenting integration specs

## Version Control

### Commit Messages
When updating documentation, use descriptive commit messages:
```
Update Phase 7 timeline in roadmap

- Extend Playbook Engine phase by 2 weeks
- Add conditional logic implementation to week 32
- Update milestone completion criteria
```

### Document Versions
Increment version numbers:
- **Major version** (1.0 → 2.0): Significant restructuring or scope changes
- **Minor version** (1.0 → 1.1): Adding new features or phases
- **No version change**: Typo fixes, clarifications, minor updates

## Important Notes

### This is Planning, Not Implementation
- Do not create actual code files (src/, tests/, etc.)
- Do not add package.json or other build configuration
- Focus on specifications, architecture, and planning
- Implementation happens in separate repositories per the plan

### Maintaining Accuracy
- Technical specs must be implementable (verify API capabilities)
- Timeline estimates should be realistic (not overly optimistic)
- Budget estimates should account for full loaded costs
- Team size recommendations based on actual workload

### Target Audience
Documents serve multiple audiences:
- **Developers**: INTEGRATIONS_SPEC.md, DATABASE_SCHEMA.md
- **Project Managers**: ROADMAP.md, PROJECT_PLAN.md § 5
- **Executives/Stakeholders**: PROJECT_PLAN.md § 1-2, README.md
- **Architects**: PROJECT_PLAN.md § 3, DATABASE_SCHEMA.md

Keep language and detail level appropriate for each audience in respective documents.
