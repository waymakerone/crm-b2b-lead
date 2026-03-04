---
sync:
  type: doc
  layer: CRM B2B Lead
build:
  status: todo
  phase: 4
  priority: P2
  depends_on: ["phase-3-campaigns-and-dashboard"]
  started_at: null
  completed_at: null
---

# Phase 4: Reporting, Import & Polish

**Goal:** Visual reporting (lead funnel, pipeline funnel, campaign comparison), bulk lead import, global search, customizable stages, aging alerts, and mobile responsiveness. Production-ready.

**PRD Reference:** `docs/01-planning/product-requirements/crm-b2b-lead-prd.md` — Phase 4

---

## What to Build

### 1. Lead Conversion Funnel

**Chart:** Horizontal funnel showing lead progression through statuses.

```
New (100) ──→ 75% ──→ Contacted (75) ──→ 53% ──→ Working (40)
──→ 50% ──→ Qualified (20) ──→ 60% ──→ Converted (12)
                                        Disqualified: 8
```

- Each step shows: count + conversion % to next step
- Color gradient from light blue (New) to green (Converted)
- Disqualified shown as a branch off "Qualified"
- Time range selector: 30d, 60d, 90d, custom

### 2. Opportunity Pipeline Funnel

**Chart:** Separate funnel for opportunity stages (post-conversion).

```
Qualification (30) ──→ 67% ──→ Needs Analysis (20) ──→ 75% ──→ Proposal (15)
──→ 60% ──→ Negotiation (9) ──→ 56% ──→ Closed Won (5)
                                         Closed Lost: 4
```

- Same visual pattern as lead funnel
- Shows where deals drop off most
- Time range selector

### 3. Campaign Comparison Report

**Table + chart** comparing multiple campaigns side by side.

| Metric | Q1 SEO | Partner Event | Outbound |
|--------|--------|---------------|----------|
| Leads | 87 | 45 | 120 |
| Converted | 23 (26%) | 18 (40%) | 15 (13%) |
| Revenue | $145K | $92K | $45K |
| Budget | $2.5K | $8K | $12K |
| ROI | 483% | 150% | 275% |
| Cost/Lead | $29 | $178 | $100 |
| Cost/Opp | $109 | $444 | $800 |

- Multi-select campaigns to compare
- Bar chart overlay: leads generated vs revenue won per campaign
- Helps answer: "Where should we spend next quarter's marketing budget?"

### 4. Activity Leaderboard

**Table:** Activity counts per team member.

| Rep | Calls | Emails | Meetings | Tasks | Notes | Total |
|-----|-------|--------|----------|-------|-------|-------|
| Sarah (SDR) | 45 | 30 | 8 | 22 | 15 | 120 |
| Mike (SDR) | 38 | 25 | 12 | 18 | 10 | 103 |
| Jane (AE) | 22 | 18 | 15 | 30 | 20 | 105 |

- Time range: this week, this month, this quarter
- Sortable by any column
- Click a rep to filter to their activities

### 5. Aging Alerts

**Lead Aging:**
- Leads with no activity in 7+ days → "Stale" badge on Lead Queue
- Dashboard widget: stale lead count with list
- Auto-trigger re-engagement Journey (if configured in Phase 3)

**Opportunity Aging:**
- Opportunities with no activity in 14+ days → "Stale" badge on pipeline board
- Dashboard widget: stale opportunities
- Optional: auto-create reminder task

**Implementation:**
- Query `MAX(crm_activities.created_at)` per lead/opportunity
- Compare against configurable threshold (default 7d leads, 14d opps)
- Settings page: adjust thresholds

### 6. Bulk Lead Import

**Import page (/leads/import):**

Step 1 — Upload:
- Drag-and-drop or file picker for CSV
- Show file name, size, row count preview

Step 2 — Column Mapping:
- Left column: CSV headers (detected from first row)
- Right column: CRM field dropdowns (first_name, last_name, email, phone, company_name, company_domain, job_title, lead_source)
- Auto-map obvious matches (e.g., "Email" → email, "First Name" → first_name)
- Required fields highlighted: email is required

Step 3 — Preview:
- Table showing first 10 rows with mapped data
- Highlight validation issues (missing email, duplicate emails)
- Duplicate detection: check existing leads by email, flag matches

Step 4 — Import:
- "Import X leads" button
- Progress bar for large imports
- Campaign assignment: optionally assign all imported leads to a campaign
- Owner assignment: optionally assign all to an owner

Step 5 — Summary:
- X leads imported successfully
- Y duplicates skipped
- Z validation errors (downloadable CSV of failed rows with error reasons)

### 7. Global Search

**Search bar** in top navigation (all pages):
- Keyboard shortcut: `Cmd+K` or `/`
- Searches across: leads (name, email, company), accounts (name, domain), contacts (name, email), opportunities (name), campaigns (name)
- Results grouped by type with icons:
  - 👤 Leads
  - 🏢 Accounts
  - 📇 Contacts
  - 💰 Opportunities
  - 📣 Campaigns
- Click to navigate to detail page
- Recent searches saved locally

### 8. Customizable Opportunity Stages

**Settings → Pipeline Stages:**
- Sortable list of stages
- Add new stage: name, default probability, position
- Edit: rename, change probability
- Delete: only if no opportunities are in that stage
- "Closed Won" and "Closed Lost" cannot be deleted (system stages)
- Reorder via drag-and-drop

### 9. Mobile Responsiveness

**Lead Queue:**
- Switch to card layout below 768px
- Key fields visible (name, company, status, score), rest behind expand
- Filters collapse to a "Filters" button with slide-out panel

**Pipeline Board:**
- Horizontal scroll with snap points
- Deal cards full-width within column
- Stage headers sticky

**Detail Pages:**
- Single column stack below 768px
- Activity timeline below info card
- Quick action buttons horizontal scroll

**Dashboard:**
- Summary cards 2x2 grid on tablet, stacked on phone
- Charts full-width and scrollable

### 10. Host Schema Manifest

```typescript
export default {
  name: 'CRM B2B Lead',
  slug: 'crm-b2b-lead',
  tables: [
    { name: 'crm_leads', access: 'read-write' },
    { name: 'crm_accounts', access: 'read-write' },
    { name: 'crm_contacts', access: 'read-write' },
    { name: 'crm_opportunities', access: 'read-write' },
    { name: 'crm_campaigns', access: 'read-write' },
    { name: 'crm_activities', access: 'read-write' },
    { name: 'crm_opportunity_stages', access: 'read-write' },
  ],
  ambassadors: [],
  commander_apis: [
    'commander-table-operations',
    'commander-contacts',
    'commander-task-operations',
    'commander-calendar-operations',
    'commander-journey-operations',
  ],
}
```

### 11. Deploy Configuration

- Verify all env vars use `${VITE_*}` placeholders
- Test with production Clerk keys
- RLS policies on all CRM tables (org-scoped)
- Test with 2+ users (SDR + AE) to verify owner-based views
- Load test: 500 leads, 100 accounts, 50 opportunities

---

## Acceptance Criteria

- [ ] Lead conversion funnel chart renders with real data and drop-off percentages
- [ ] Opportunity pipeline funnel chart renders with stage-to-stage conversion
- [ ] Campaign comparison: multi-select campaigns, side-by-side metrics + chart
- [ ] Activity leaderboard ranks reps by activity count with time range selector
- [ ] Stale leads (7+ days) show badge on Lead Queue
- [ ] Stale opportunities (14+ days) show badge on pipeline board
- [ ] CSV lead import: upload → map → preview → import with duplicate detection
- [ ] Global search (Cmd+K) returns results across all entity types
- [ ] Opportunity stages are editable in settings (add, rename, reorder, delete)
- [ ] Mobile: Lead Queue switches to card layout below 768px
- [ ] Mobile: pipeline scrolls horizontally with snap
- [ ] Mobile: detail pages stack single-column
- [ ] waymaker.config.ts manifest complete and valid
- [ ] All tables have RLS policies scoped to organization_id
- [ ] Passes blueprints:validate — no secrets or internal refs
- [ ] Deploys successfully to Waymaker Host
