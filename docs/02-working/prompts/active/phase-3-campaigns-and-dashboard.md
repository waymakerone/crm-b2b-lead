---
sync:
  type: doc
  layer: CRM B2B Lead
build:
  status: todo
  phase: 3
  priority: P1
  depends_on: ["phase-2-pipeline-and-activities"]
  started_at: null
  completed_at: null
---

# Phase 3: Campaigns & Dashboard

**Goal:** Campaign management with ROI tracking, two-part dashboard (lead metrics + pipeline metrics), lead scoring, and Journeys integration for automated sequences.

**PRD Reference:** `docs/01-planning/product-requirements/crm-b2b-lead-prd.md` — Phase 3

---

## What to Build

### 1. Campaign Management

**Campaign List (/campaigns):**
- Table: name, type (badge), status (badge), start/end dates, budget, leads count, revenue, ROI
- ROI column: `(actual_revenue - budget) / budget × 100%` — green if positive, red if negative
- Filter by type, status
- "New Campaign" button

**Campaign Form (modal):**
- Fields: name (required), type (dropdown), status (dropdown), start date, end date, budget, expected revenue, description

**Campaign Detail (/campaigns/:id):**
```
┌──────────────────────────────────────────────────┐
│ Campaign: Q1 SEO Content                         │
│ Type: Content | Status: Active | Jan-Mar 2026    │
│                                                  │
│ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────┐ │
│ │ Leads    │ │ Converted│ │ Revenue  │ │ ROI  │ │
│ │ Generated│ │          │ │ Won      │ │      │ │
│ │ 87       │ │ 23 (26%) │ │ $145,000 │ │ 483% │ │
│ └──────────┘ └──────────┘ └──────────┘ └──────┘ │
│                                                  │
│ ┌──────────────────────┐ ┌──────────────────────┐│
│ │ Leads from Campaign  │ │ Opportunities from   ││
│ │ Name | Status | Score│ │ Campaign             ││
│ │ Jane  | Converted    │ │ Acme Q2 — $50K — Won ││
│ │ Bob   | Working   45 │ │ BW Pilot — $8K — Open││
│ │ Sara  | New       10 │ │                      ││
│ └──────────────────────┘ └──────────────────────┘│
│                                                  │
│ Budget: $2,500 | Expected Revenue: $100,000      │
│ Cost per Lead: $28.74 | Cost per Opportunity: $108││
└──────────────────────────────────────────────────┘
```

**Campaign metrics (computed):**
- Leads generated: count of leads with this campaign_id
- Conversion rate: converted leads / total leads
- Revenue won: sum of closed_won opportunities linked via converted leads
- ROI: (revenue - budget) / budget
- Cost per lead: budget / lead count
- Cost per opportunity: budget / opportunity count

### 2. Dashboard (/dashboard)

Two-section layout: Lead metrics (SDR view) and Pipeline metrics (AE view).

**Lead Metrics Section:**
```
┌──────────────────────────────────────────────────┐
│ LEADS                                  [30d ▼]   │
│                                                  │
│ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────┐ │
│ │ New Leads│ │ Conversion│ │ Avg Days │ │ Qual.│ │
│ │ (period) │ │ Rate      │ │ to Conv. │ │ Rate │ │
│ │ 42       │ │ 28%       │ │ 8.5 days │ │ 45%  │ │
│ │ +15% ↑   │ │ +3% ↑    │ │ -2d ↓    │ │      │ │
│ └──────────┘ └──────────┘ └──────────┘ └──────┘ │
│                                                  │
│ ┌─────────────────────┐ ┌──────────────────────┐ │
│ │ Leads by Source      │ │ Leads by Status      │ │
│ │ ████ Website (35)    │ │ ██████ New (15)      │ │
│ │ ███ Event (28)       │ │ ████ Contacted (12)  │ │
│ │ ██ Referral (15)     │ │ ████ Working (10)    │ │
│ │ █ Outbound (8)       │ │ ██ Qualified (5)     │ │
│ └─────────────────────┘ └──────────────────────┘ │
└──────────────────────────────────────────────────┘
```

**Lead metrics:**
- New leads (period): count created in selected time range
- Conversion rate: leads converted / total leads (period)
- Average days to convert: mean of (converted_at - created_at) for converted leads
- Qualification rate: leads reaching "qualified" status / total leads
- Leads by source: horizontal bar chart
- Leads by status: horizontal bar chart

**Pipeline Metrics Section:**
```
┌──────────────────────────────────────────────────┐
│ PIPELINE                               [90d ▼]   │
│                                                  │
│ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────┐ │
│ │ Pipeline │ │ Open     │ │ Win Rate │ │ Avg  │ │
│ │ Value    │ │ Opps     │ │          │ │ Deal │ │
│ │ $380,000 │ │ 18       │ │ 35%      │ │$22.5K│ │
│ └──────────┘ └──────────┘ └──────────┘ └──────┘ │
│                                                  │
│ ┌─────────────────────┐ ┌──────────────────────┐ │
│ │ Pipeline by Stage    │ │ My Tasks (Due Soon)  │ │
│ │ ████ Qualification   │ │ ☐ Follow up — Acme   │ │
│ │ ███ Needs Analysis   │ │ ☐ Send deck — BW     │ │
│ │ ██ Proposal          │ │ ☑ Call recap — Delta  │ │
│ │ █ Negotiation        │ │                      │ │
│ └─────────────────────┘ └──────────────────────┘ │
│                                                  │
│ ┌─────────────────────┐ ┌──────────────────────┐ │
│ │ Top Campaigns (ROI)  │ │ Opps Closing Soon    │ │
│ │ Q1 SEO — 483%        │ │ Acme Q2 — $50K — 3d │ │
│ │ Webinar — 210%       │ │ BW Pilot — $8K — 7d  │ │
│ │ Partner — 150%       │ │ ⚠ Delta — $30K — -2d │ │
│ └─────────────────────┘ └──────────────────────┘ │
└──────────────────────────────────────────────────┘
```

**Pipeline metrics:**
- Total pipeline value, open opportunity count, win rate, average deal size
- Pipeline by stage (bar chart)
- My Tasks from CRM activities (incomplete, sorted by due date)
- Top campaigns by ROI
- Opportunities closing soon (within 14 days, overdue highlighted red)

**Time range selector:** 30d, 60d, 90d, this quarter, this year

### 3. Lead Scoring

Simple scoring model to start — auto-incremented by activity, manually adjustable.

**Auto-scoring rules:**
| Action | Points |
|--------|--------|
| Lead created | +5 |
| Call logged (connected) | +10 |
| Call logged (voicemail) | +3 |
| Meeting scheduled | +20 |
| Email sent | +5 |
| Note added | +2 |
| Status → contacted | +5 |
| Status → working | +10 |
| Manual adjustment | User sets directly |

**Score display:**
- On Lead Queue: numeric score with color (0-25 gray, 26-50 blue, 51-75 orange, 76-100 green)
- On Lead Detail: score with breakdown tooltip showing how points were earned
- Score resets to 0 on disqualification

**Implementation:**
- Add scoring logic in `activities.ts` — after creating an activity, update the lead score
- Score is stored on `crm_leads.lead_score`
- Display score breakdown via computed query on activities for that lead

### 4. Journeys Integration

**Trigger points for automated email sequences:**

| Event | Journey to Trigger |
|-------|-------------------|
| New lead created | "New Lead Welcome" |
| Lead status → qualified | "Qualified Lead — SDR Handoff" |
| Lead converted → Opportunity created | "New Opportunity — AE Introduction" |
| Opportunity → closed_won | "Customer Onboarding Welcome" |
| Opportunity → closed_lost | "Lost Deal — Win-Back Sequence" |
| Lead stale (no activity 7+ days) | "Lead Re-engagement" |

**Implementation:**
```
POST /functions/v1/commander-journey-operations
Body: {
  "action": "enroll_contact",
  "data": {
    "journey_slug": "new-lead-welcome",
    "contact_email": "{lead.email}",
    "trigger_source": "crm",
    "metadata": {
      "lead_source": "website",
      "campaign": "Q1 SEO"
    }
  }
}
```

**Settings page (/settings):**
- Map each trigger event to a Journey slug (or "disabled")
- Toggle automations on/off per trigger
- Test mode: log instead of sending
- If Journeys not configured: info message explaining the integration

---

## Acceptance Criteria

- [ ] Campaign CRUD: create, edit, list with type/status badges
- [ ] Campaign Detail shows leads, opportunities, revenue, ROI, cost per lead
- [ ] Dashboard Lead section: new leads, conversion rate, avg days to convert, charts
- [ ] Dashboard Pipeline section: value, win rate, avg deal, pipeline by stage chart
- [ ] Dashboard My Tasks shows upcoming CRM tasks
- [ ] Dashboard Top Campaigns shows campaigns ranked by ROI
- [ ] Lead score auto-increments on activity creation
- [ ] Lead score displayed with color coding on Lead Queue and Detail
- [ ] Journeys triggers fire on lifecycle events (when configured)
- [ ] Settings page allows mapping events to Journey slugs
- [ ] Time range selector works across all dashboard metrics
