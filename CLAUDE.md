# B2B CRM (Lead-Centric)

Salesforce-style lead-centric B2B CRM for WaymakerOS. Leads → Qualification → Conversion → Account + Contact + Opportunity → Pipeline. Campaigns track ROI end-to-end.

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React + TypeScript + Vite + Tailwind CSS |
| Auth | Clerk (via WaymakerOS) |
| Data | Commander Tables (Supabase PostgreSQL) |
| Drag & Drop | @dnd-kit/core + @dnd-kit/sortable |
| Charts | recharts |
| Hosting | Waymaker Host (EX app) |

## Documentation

| Folder | Contents |
|--------|----------|
| `docs/01-planning/product-requirements/` | PRD — data model, conversion logic, views |
| `docs/02-working/prompts/active/` | Build prompts — 4 phases with YAML front matter |
| `docs/02-working/sessions/` | Session briefs as you build |
| `docs/03-knowledge/` | Patterns discovered during the build |

## Build Phases

| Phase | Prompt | Status |
|-------|--------|--------|
| 1 — Leads & Conversion | `docs/02-working/prompts/active/phase-1-leads-and-conversion.md` | todo |
| 2 — Pipeline & Activities | `docs/02-working/prompts/active/phase-2-pipeline-and-activities.md` | todo |
| 3 — Campaigns & Dashboard | `docs/02-working/prompts/active/phase-3-campaigns-and-dashboard.md` | todo |
| 4 — Reporting, Import & Polish | `docs/02-working/prompts/active/phase-4-reporting-import-polish.md` | todo |

## Data Model (Quick Reference)

```
crm_campaigns ──< crm_leads ──CONVERT──> crm_accounts ──< crm_contacts
                     │                        │                │
                     │                        └──< crm_opportunities
                     │                                    │
                crm_activities <──────────────────────────┘
```

**The Conversion Flow:**
1. Lead is qualified by SDR/BDR
2. "Convert Lead" → creates Account + Contact + (optional) Opportunity
3. Duplicate account detection: matches by domain/name, offers "link to existing"
4. Lead frozen after conversion, links to created records
5. All lead activities backfilled to new Contact

**Key distinction:** Leads are SEPARATE from Contacts. A Lead becomes a Contact only at conversion. This keeps the pipeline clean — only qualified deals.

## Commander Integration

| CRM Action | Commander Tool | API |
|------------|---------------|-----|
| Create follow-up task | Tasks | `commander-task-operations` → `create_task` |
| Schedule meeting | Calendar | `commander-calendar-operations` → `create_event` |
| Lead nurture sequence | Journeys | `commander-journey-operations` → `enroll_contact` |
| Attach document | Docs | Link via activity note |
| Campaign ROI | Metrics | Dashboard queries |
| Contact record | Contacts | Created at conversion via `commander-contacts` |
| All CRM tables | Tables | `commander-table-operations` → CRUD |

## Critical Rules

- **Leads are NOT Contacts** — they exist in separate tables until conversion
- Conversion is atomic: Account + Contact + Opportunity created in one operation
- After conversion, the Lead record is frozen (read-only)
- Opportunities start at "Qualification" stage — discovery happens during lead qualification
- Auth via Clerk: `useAuth().getToken()` → Bearer token on all API calls
- API calls POST to `${SUPABASE_URL}/functions/v1/{function-name}` with `{ action, data }` body
- Creating a Contact at conversion also creates a Commander Contact (linked via `commander_contact_id`)
- Tasks and Meetings flow through Commander (not siloed in the CRM)
- Design tokens: Gold `#A49886`, Navy `#001126`, Blue `#3D5B6C`, Sand `#F5F5F0`
- Use Geist font family
- All tables scoped by `organization_id` with RLS policies

## How to Build

1. Read the PRD: `docs/01-planning/product-requirements/crm-b2b-lead-prd.md`
2. Work through each phase prompt in order
3. Update the YAML `status` field as you go: `todo` → `in-progress` → `review` → `done`
4. Write session briefs in `docs/02-working/sessions/completed/` between sessions

## Quick Reference

| What | How |
|------|-----|
| Dev server | `npm run dev` |
| Build | `npm run build` |
| Tables API | `commander-table-operations` → `query`, `insert`, `update`, `delete` |
| Contacts API | `commander-contacts` → `create_contact` (at conversion) |
| Tasks API | `commander-task-operations` → `create_task` |
| Calendar API | `commander-calendar-operations` → `create_event` |
| Journeys API | `commander-journey-operations` → `enroll_contact` |

## Lead Statuses

| Status | Color | Meaning |
|--------|-------|---------|
| `new` | Blue | Just created, not yet contacted |
| `contacted` | Yellow | First outreach made |
| `working` | Orange | Active conversation, multiple touches |
| `qualified` | Green | Ready to convert — meets qualification criteria |
| `converted` | Purple | Converted to Account + Contact + Opportunity |
| `disqualified` | Gray | Not a fit — reason required |

## Opportunity Stages

| Stage | Probability | Meaning |
|-------|------------|---------|
| Qualification | 20% | Confirming budget, authority, need, timeline |
| Needs Analysis | 40% | Deep dive on requirements |
| Proposal | 60% | Formal proposal delivered |
| Negotiation | 80% | Terms being finalized |
| Closed Won | 100% | Deal signed |
| Closed Lost | 0% | Deal lost — reason required |
