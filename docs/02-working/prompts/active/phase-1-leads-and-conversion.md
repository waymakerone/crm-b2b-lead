---
sync:
  type: doc
  layer: CRM B2B Lead
build:
  status: todo
  phase: 1
  priority: P0
  depends_on: []
  started_at: null
  completed_at: null
---

# Phase 1: Leads & Conversion

**Goal:** App scaffold with auth, full schema, Lead Queue with status workflow, Lead Detail page, and the critical Convert Lead dialog that atomically creates Account + Contact + Opportunity.

**PRD Reference:** `docs/01-planning/product-requirements/crm-b2b-lead-prd.md` — Phase 1

---

## What to Build

### 1. Project Setup

Create a React + Vite + TypeScript + Tailwind app.

**Files:**
- `src/main.tsx` — Clerk provider wrapper
- `src/App.tsx` — Router with routes: `/leads`, `/leads/:id`, `/accounts`, `/accounts/:id`, `/contacts/:id`, `/opportunities`, `/opportunities/:id`, `/campaigns`, `/dashboard`
- `src/lib/api.ts` — Authenticated fetch helper for Commander Tables
- `src/lib/types.ts` — TypeScript interfaces for Lead, Account, Contact, Opportunity, Campaign, Activity
- `src/index.css` — Tailwind base + Waymaker design tokens

**Auth pattern:**
```typescript
import { useAuth } from '@clerk/clerk-react'

const { getToken } = useAuth()
const token = await getToken()

const res = await fetch(`${SUPABASE_URL}/functions/v1/commander-table-operations`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`,
  },
  body: JSON.stringify({ action: 'query', data: { table: 'crm_leads', filters: {} } }),
})
```

### 2. Database Schema

Create all tables from the PRD. Seed opportunity stages:

| display_order | name | probability_default | is_won | is_lost |
|---|---|---|---|---|
| 1 | Qualification | 20 | false | false |
| 2 | Needs Analysis | 40 | false | false |
| 3 | Proposal | 60 | false | false |
| 4 | Negotiation | 80 | false | false |
| 5 | Closed Won | 100 | true | false |
| 6 | Closed Lost | 0 | false | true |

Note: no "Discovery" stage — discovery happens during lead qualification, before conversion.

### 3. API Layer

```
src/services/
├── leads.ts          — list, get, create, update, changeStatus, convert, disqualify, bulkAssign
├── accounts.ts       — list, get, create, update, findByDomain, findByName
├── contacts.ts       — list, get, create, update
├── opportunities.ts  — list, get, create, update, moveStage
├── campaigns.ts      — list, get, create, update
├── activities.ts     — list, create (by entity type)
└── stages.ts         — listStages
```

### 4. Lead Queue (/leads)

The primary workspace for SDRs/BDRs.

**Table layout:**
| Column | Content |
|--------|---------|
| Name | First + Last name |
| Company | company_name |
| Email | email (clickable mailto) |
| Source | Lead source badge |
| Status | Colored status badge |
| Score | Numeric score with progress bar |
| Owner | User avatar/initials |
| Campaign | Campaign name (if linked) |
| Last Activity | Relative time ("2h ago", "yesterday") |
| Created | Date |

**Status badges:**
- `new` — blue
- `contacted` — yellow
- `working` — orange
- `qualified` — green
- `converted` — purple (with link icon)
- `disqualified` — gray (strikethrough)

**Filters bar:**
- Status dropdown (multi-select)
- Lead source dropdown (multi-select)
- Owner dropdown
- Campaign dropdown
- Date range (created)
- Search: name, email, company

**Bulk actions:**
- Select multiple leads → Assign to owner
- Select multiple leads → Change status
- "New Lead" button → modal form

**Lead Form (modal):**
- Fields: first name, last name, email (required), phone, company name, company domain, job title, lead source (dropdown), campaign (dropdown), owner (dropdown)

### 5. Lead Detail (/leads/:id)

**Layout:**
```
┌────────────────────────────────────────────────────┐
│ [← Back to Leads]              [Disqualify] [Convert] │
│                                                    │
│ ┌───────────────────┐  ┌────────────────────────┐  │
│ │ Lead Info          │  │ Activity Timeline      │  │
│ │ Name: Jane Smith   │  │ [Log Call] [Note]      │  │
│ │ Company: Acme Corp │  │                        │  │
│ │ Email: jane@acme   │  │ ── Today ──            │  │
│ │ Phone: +1 555...   │  │ 📞 Call: connected 5m  │  │
│ │ Title: VP Sales    │  │    "Interested in demo" │  │
│ │                    │  │                        │  │
│ │ Source: Website    │  │ ── Yesterday ──        │  │
│ │ Campaign: Q1 SEO   │  │ 📝 Note: "Downloaded   │  │
│ │ Status: Working    │  │    whitepaper"          │  │
│ │ Score: 45/100      │  │                        │  │
│ │ Owner: SDR Name    │  │ ── Feb 24 ──           │  │
│ │                    │  │ 🆕 Lead created         │  │
│ │ [Edit Lead]        │  │    Source: Website      │  │
│ └───────────────────┘  └────────────────────────┘  │
│                                                    │
│ ┌──────────────────────────────────────────────────┐│
│ │ Conversion Info (only shown if converted)        ││
│ │ Converted: Feb 26, 2026 by Stuart Leo            ││
│ │ Account: Acme Corp (link) | Contact: Jane (link) ││
│ │ Opportunity: Acme Q2 Deal (link)                 ││
│ └──────────────────────────────────────────────────┘│
└────────────────────────────────────────────────────┘
```

**Key behaviors:**
- Status is editable via dropdown (except `converted` — that's set by the Convert action)
- Score is editable manually (auto-scoring comes in Phase 3)
- If status = `converted`: show Conversion Info panel, hide Convert button, make lead read-only
- If status = `disqualified`: show reason, gray out card, but allow re-opening
- Activity timeline: notes and calls for now (tasks/meetings in Phase 2)

### 6. Convert Lead Dialog

The most important UX in this entire CRM. Must be clear, fast, and safe.

**Trigger:** Click "Convert Lead" button on a qualified lead (warn if status is not `qualified`).

**Dialog layout:**
```
┌─────────────────────────────────────────────┐
│ Convert Lead: Jane Smith                     │
│                                             │
│ ACCOUNT                                     │
│ ┌─────────────────────────────────────────┐ │
│ │ ⚠ Existing account found: "Acme Corp"  │ │
│ │   (acme.com) — 3 contacts, 2 opps      │ │
│ │   ○ Link to existing  ● Create new     │ │
│ └─────────────────────────────────────────┘ │
│ Name: [Acme Corp        ]                   │
│ Domain: [acme.com       ]                   │
│ Industry: [Technology ▼ ]                   │
│                                             │
│ CONTACT                                     │
│ Name: [Jane Smith       ] (from lead)       │
│ Email: [jane@acme.com   ] (from lead)       │
│ Phone: [+1 555-0123     ] (from lead)       │
│ Title: [VP Sales        ] (from lead)       │
│                                             │
│ OPPORTUNITY (optional)                      │
│ ☑ Create opportunity                        │
│ Name: [Acme Corp — Feb 2026]               │
│ Amount: [$               ]                  │
│ Stage: [Qualification ▼ ]                   │
│ Close Date: [2026-04-30  ]                  │
│ Owner: [AE Name ▼       ]                  │
│                                             │
│           [Cancel]  [Convert Lead]          │
└─────────────────────────────────────────────┘
```

**Duplicate Account Detection:**
1. On dialog open, query `crm_accounts` for matching `domain` (exact) or `name` (case-insensitive contains)
2. If match found: show yellow banner with account details, radio to "Link to existing" or "Create new"
3. "Link to existing" uses the matched account — no new account created
4. "Create new" creates a fresh account even if similar exists (user's choice)

**Conversion API call:**
```typescript
// Single atomic operation
const result = await leads.convert(leadId, {
  account: { name, domain, industry } | { existing_account_id },
  contact: { first_name, last_name, email, phone, job_title },
  opportunity: null | { name, amount, stage, expected_close_date, owner_id },
})
// Returns: { account_id, contact_id, opportunity_id? }
```

**Post-conversion:**
- Lead status = `converted`, fields frozen
- Commander Contact created (via `commander-contacts` API), `commander_contact_id` stored
- All lead activities get `contact_id` set to the new contact (backfill)
- Redirect to the new Contact detail page (or Opportunity if created)

### 7. Disqualify Lead Dialog

Simple modal:
- Reason dropdown: not a fit, no budget, wrong timing, competitor, duplicate, spam, other
- Optional notes textarea
- Sets status to `disqualified`, stores reason, logs as activity

### 8. Design Tokens

Same Waymaker brand as the contact-centric blueprint:
- Font: `Geist`
- Gold: `#A49886` — accents, Convert button, qualified badges
- Navy: `#001126` — primary text
- Blue: `#3D5B6C` — headers
- Sand: `#F5F5F0` — page backgrounds
- Cards: white, `border-radius: 12px`, subtle shadow

---

## Acceptance Criteria

- [ ] App loads with Clerk auth (sign-in required)
- [ ] Lead Queue shows all leads with status badges, scores, and source
- [ ] Leads can be created, edited, and status-changed through the workflow
- [ ] Lead Detail shows full profile with activity timeline (notes + calls)
- [ ] Convert Lead dialog auto-fills from lead data
- [ ] Duplicate account detection works by domain and name
- [ ] Conversion atomically creates Account + Contact + (optional) Opportunity
- [ ] Converted leads are frozen (read-only with conversion info panel)
- [ ] Disqualify dialog requires a reason
- [ ] Disqualified leads can be re-opened
- [ ] Account and Contact basic CRUD works
- [ ] Search and filters work on Lead Queue
- [ ] Empty state: "No leads yet — create your first lead or import a list"
