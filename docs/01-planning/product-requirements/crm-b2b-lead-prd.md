# B2B CRM (Lead-Centric) — Product Requirements

**Status:** Approved
**Author:** Waymaker
**Date:** 2026-02-26
**Last Updated:** 2026-02-26

## Problem Statement

Companies with structured sales teams — SDRs doing outbound, BDRs qualifying inbound, AEs closing deals — need a CRM that mirrors that workflow. Leads come in raw from campaigns, events, and outbound lists. They need to be qualified before they deserve a pipeline slot. Mixing unqualified leads into the deal pipeline pollutes forecasting and wastes AE time.

The Salesforce model solves this: Leads are a separate object. They get worked, scored, and qualified by SDRs/BDRs. When qualified, they "convert" — creating an Account, Contact, and Opportunity in one action. The AE picks up a clean, qualified opportunity. Pipeline stays honest.

The problem with Salesforce: it costs $150+/seat/month, takes months to configure, and the conversion process is a black box that confuses most users. The model is right. The implementation is overengineered.

This blueprint gives you the Salesforce lead model without the Salesforce price tag or complexity — built on WaymakerOS, integrated with Commander, deployable in a day.

## Goals

1. Clean separation between unqualified leads and qualified pipeline
2. Structured lead qualification workflow with scoring
3. One-click conversion: Lead → Account + Contact + Opportunity
4. Full audit trail — every lead knows where it came from and who touched it
5. Campaign tracking — know which campaigns generate revenue, not just leads
6. Commander integration — tasks, meetings, email all flow through existing tools

## Non-Goals

- This is NOT a marketing automation platform — no landing pages, forms, or ad tracking
- This is NOT a contact-centric CRM — leads ARE separate from contacts until conversion
- This is NOT a customer support tool — no ticketing or SLAs
- This does NOT replace Commander's Contacts — it creates them at conversion

## Data Model

### Core Objects

**Lead** — An unqualified person who has expressed interest or been targeted.
- Fields: first_name, last_name, email, phone, company_name, job_title, lead_source, lead_status, lead_score, owner_id, campaign_id
- Lead statuses: `new` → `contacted` → `working` → `qualified` → `converted` | `disqualified`
- Lead sources: website, referral, partner, event, outbound, cold_call, social, advertisement, other
- A Lead is NOT a Contact. It becomes a Contact only at conversion.
- Once converted, the Lead record is frozen (status = `converted`, `converted_at` timestamp, links to created Account/Contact/Opportunity)

**Account** — A company/organization (created at conversion or manually).
- Fields: name, domain, industry, size_band, annual_revenue_band, account_type, billing_address, owner_id
- Account types: `prospect` → `customer` → `partner` → `churned`
- Linked to Contacts (one-to-many) and Opportunities (one-to-many)

**Contact** — A qualified person at an Account (created at conversion or manually).
- Fields: first_name, last_name, email, phone, job_title, account_id, commander_contact_id, owner_id
- Links to Commander Contacts via `commander_contact_id`
- A Contact always belongs to an Account

**Opportunity** — A qualified sales deal with stages and value.
- Fields: name, amount, currency, stage, probability, expected_close_date, actual_close_date, won_lost_reason, account_id, primary_contact_id, owner_id, campaign_id
- Pipeline stages: `qualification` → `needs_analysis` → `proposal` → `negotiation` → `closed_won` | `closed_lost`
- Note: "Discovery" happens at the Lead stage. Opportunities start at Qualification.

**Campaign** — A marketing or sales initiative that generates leads.
- Fields: name, type, status, start_date, end_date, budget, expected_revenue, description
- Campaign types: email, event, webinar, content, referral, outbound, advertisement, partner
- Campaign statuses: `planned` → `active` → `completed`
- Leads and Opportunities link back to their originating Campaign

**Activity** — Anything that happened on a lead, contact, or opportunity.
- Types: `email`, `call`, `meeting`, `note`, `task`
- Polymorphic linking: can reference a lead_id, contact_id, or opportunity_id
- Commander integration: tasks → Taskboard, meetings → Calendar

### The Conversion Process

This is the key differentiator from contact-centric CRMs. When an SDR/BDR qualifies a lead:

1. User clicks "Convert Lead" on a qualified lead
2. Conversion dialog shows:
   - **Account**: auto-filled from lead's `company_name`. If an account with matching name/domain exists, offer to link to it (don't create a duplicate). Otherwise, create new.
   - **Contact**: auto-filled from lead's name, email, phone, job title. Always created new.
   - **Opportunity**: optional. Pre-fill name as "{Company} — {date}". Amount, stage (defaults to Qualification), expected close date.
3. User confirms → one API call creates all three records atomically
4. Lead status changes to `converted`, stores `converted_account_id`, `converted_contact_id`, `converted_opportunity_id`, `converted_at`
5. All activities from the Lead are copied/linked to the new Contact
6. The Lead record is frozen — read-only from this point

**Duplicate Account detection:**
- Match on `domain` (exact) or `name` (fuzzy, case-insensitive)
- If match found: show dialog "An account named X already exists. Link to existing or create new?"
- This prevents the #1 Salesforce admin headache: duplicate accounts

### Tables Schema (Commander Tables)

```
crm_leads
├── id (uuid, PK)
├── organization_id (text, FK → Clerk org)
├── first_name (text)
├── last_name (text)
├── email (text, required)
├── phone (text)
├── company_name (text)
├── company_domain (text)
├── job_title (text)
├── lead_source (text: website, referral, partner, event, outbound, cold_call, social, advertisement, other)
├── lead_status (text: new, contacted, working, qualified, converted, disqualified)
├── lead_score (integer, 0-100, default 0)
├── disqualified_reason (text, nullable)
├── campaign_id (uuid, FK → crm_campaigns, nullable)
├── owner_id (text, Clerk user ID — the SDR/BDR)
├── converted_at (timestamptz, nullable)
├── converted_account_id (uuid, FK → crm_accounts, nullable)
├── converted_contact_id (uuid, FK → crm_contacts, nullable)
├── converted_opportunity_id (uuid, FK → crm_opportunities, nullable)
├── last_contacted_at (timestamptz)
├── created_at / updated_at
└── created_by (text)

crm_accounts
├── id (uuid, PK)
├── organization_id (text)
├── name (text, required)
├── domain (text)
├── industry (text)
├── size_band (text: 1-10, 11-50, 51-200, 201-1000, 1000+)
├── annual_revenue_band (text)
├── account_type (text: prospect, customer, partner, churned)
├── billing_address (jsonb)
├── notes (text)
├── owner_id (text, Clerk user ID)
├── created_at / updated_at
└── created_by (text)

crm_contacts
├── id (uuid, PK)
├── organization_id (text)
├── account_id (uuid, FK → crm_accounts, required)
├── commander_contact_id (uuid, FK → Commander Contacts, nullable)
├── first_name (text)
├── last_name (text)
├── email (text, required)
├── phone (text)
├── job_title (text)
├── is_primary (boolean, default false — primary contact on the account)
├── owner_id (text)
├── last_contacted_at (timestamptz)
├── created_at / updated_at
└── created_by (text)

crm_opportunities
├── id (uuid, PK)
├── organization_id (text)
├── account_id (uuid, FK → crm_accounts, required)
├── primary_contact_id (uuid, FK → crm_contacts)
├── campaign_id (uuid, FK → crm_campaigns, nullable)
├── name (text, required)
├── amount (numeric)
├── currency (text, default 'USD')
├── stage (text: qualification, needs_analysis, proposal, negotiation, closed_won, closed_lost)
├── probability (integer, 0-100)
├── expected_close_date (date)
├── actual_close_date (date)
├── won_lost_reason (text)
├── owner_id (text, Clerk user ID — the AE)
├── created_at / updated_at
└── created_by (text)

crm_campaigns
├── id (uuid, PK)
├── organization_id (text)
├── name (text, required)
├── type (text: email, event, webinar, content, referral, outbound, advertisement, partner)
├── status (text: planned, active, completed)
├── start_date (date)
├── end_date (date)
├── budget (numeric)
├── expected_revenue (numeric)
├── actual_revenue (numeric, computed from closed_won opportunities)
├── description (text)
├── created_at / updated_at
└── created_by (text)

crm_activities
├── id (uuid, PK)
├── organization_id (text)
├── lead_id (uuid, FK → crm_leads, nullable)
├── contact_id (uuid, FK → crm_contacts, nullable)
├── opportunity_id (uuid, FK → crm_opportunities, nullable)
├── activity_type (text: email, call, meeting, note, task)
├── subject (text)
├── body (text)
├── call_outcome (text: connected, voicemail, no_answer, wrong_number — for calls)
├── call_duration_minutes (integer — for calls)
├── commander_task_id (uuid, nullable)
├── commander_event_id (uuid, nullable)
├── due_date (timestamptz, nullable)
├── completed_at (timestamptz, nullable)
├── created_at
└── created_by (text)

crm_opportunity_stages
├── id (uuid, PK)
├── organization_id (text)
├── name (text)
├── display_order (integer)
├── probability_default (integer)
├── is_won (boolean, default false)
├── is_lost (boolean, default false)
└── created_at
```

### Commander Integration Map

| CRM Action | Commander Tool | How |
|------------|---------------|-----|
| Create a follow-up task | Tasks | Activity with type `task` → creates Commander Task, stores `commander_task_id` |
| Schedule a meeting | Calendar | Activity with type `meeting` → creates Calendar event, stores `commander_event_id` |
| Lead nurture sequence | Journeys | Lead status change triggers email Journey |
| Customer onboarding | Journeys | Opportunity closed_won triggers onboarding Journey |
| Proposal document | Docs | Created in Docs, linked to opportunity via activity |
| Campaign ROI | Metrics | Revenue per campaign on Commander Metrics dashboard |
| Contact record | Contacts | Created at conversion, linked via `commander_contact_id` |

## Proposed Solution

### Overview

An internal (EX) app deployed to Waymaker Host that implements the Salesforce-style lead model: Leads are separate from Contacts, get qualified through a structured workflow, and convert into Account + Contact + Opportunity when ready. Campaigns track where leads originate. The pipeline shows only qualified opportunities.

### Key Views

1. **Lead Queue** — Table of leads with status columns, owner, score, and source. Filterable by status, source, owner, campaign. Bulk actions: assign, change status.

2. **Lead Detail** — Full lead profile with qualification fields, lead score, activity timeline, and the "Convert Lead" button.

3. **Convert Lead Dialog** — The critical workflow: maps lead data to Account + Contact + Opportunity. Detects duplicate accounts. Creates all three atomically.

4. **Opportunity Pipeline** — Kanban board of opportunities by stage. Only qualified deals — no leads polluting the forecast. Drag-and-drop between stages.

5. **Account Detail** — Company profile with all contacts, all opportunities, total revenue, and activity stream.

6. **Contact Detail** — Person profile with account link, opportunities, activities. Shows "Converted from Lead" badge with link to original lead.

7. **Campaign List & Detail** — Campaign management with ROI tracking: budget vs actual revenue, lead count, conversion rate.

8. **Dashboard** — Two sections: Lead metrics (new leads, conversion rate, avg time to convert) and Pipeline metrics (total value, win rate, forecast).

### User Flow (SDR → AE handoff)

1. Marketing runs a campaign → 100 leads imported into Lead Queue
2. SDR opens Lead Queue, filters to "New", starts working
3. SDR clicks a lead → opens Lead Detail → calls the contact
4. Logs the call (connected, 5 min, "interested in product demo")
5. Creates a follow-up task: "Send case study" → appears on their Commander Taskboard
6. Updates lead status to "Working", score increases
7. After 2-3 touches, SDR qualifies the lead → status = "Qualified"
8. SDR clicks "Convert Lead" → Account, Contact, Opportunity created
9. SDR assigns the Opportunity to an AE
10. AE opens Opportunity Pipeline → sees the new deal in "Qualification" stage
11. AE works the deal through stages, using activities for each interaction
12. Deal closes → Campaign ROI updates automatically

## Scope

### Phase 1 (MVP) — Leads & Conversion

- [ ] App scaffold: React + Vite + Tailwind + Clerk auth
- [ ] Tables schema: all tables above with seed data for stages
- [ ] API layer: CRUD for leads, accounts, contacts, opportunities
- [ ] Lead Queue: table view with status, source, score, owner, campaign
- [ ] Lead status workflow: new → contacted → working → qualified → converted/disqualified
- [ ] Lead Detail page with activity timeline (notes and calls only)
- [ ] Convert Lead dialog: auto-fill, duplicate account detection, atomic creation
- [ ] Lead → Account + Contact + Opportunity conversion logic
- [ ] Disqualify Lead dialog: reason required
- [ ] Account and Contact basic CRUD

### Phase 2 — Pipeline & Activities

- [ ] Opportunity Pipeline: Kanban board with drag-and-drop
- [ ] Opportunity Detail page with stage progression bar
- [ ] Account Detail page: contacts, opportunities, revenue
- [ ] Contact Detail page: "Converted from Lead" badge, activities
- [ ] Full activity system: calls, notes, tasks, meetings, emails
- [ ] Commander Task integration (create task → Taskboard)
- [ ] Commander Calendar integration (schedule meeting → Calendar)
- [ ] Activity timeline on Lead, Contact, and Opportunity detail pages
- [ ] Activities from converted Lead linked to new Contact

### Phase 3 — Campaigns & Dashboard

- [ ] Campaign CRUD: create, edit, list
- [ ] Campaign Detail: leads generated, opportunities created, revenue won
- [ ] Campaign ROI: budget vs actual revenue, cost per lead, cost per opportunity
- [ ] Lead assignment to campaigns (at creation or bulk update)
- [ ] Dashboard — Lead section: new leads (period), conversion rate, avg days to convert, leads by source
- [ ] Dashboard — Pipeline section: total value, opportunity count by stage, win rate, avg deal size
- [ ] Dashboard — Campaign section: top campaigns by ROI, leads by campaign
- [ ] Lead scoring: manual score + auto-increment on activity (call = +10, meeting = +20, email reply = +5)
- [ ] Journeys integration: trigger sequences on lead status change and opportunity stage change

### Phase 4 — Reporting, Import & Polish

- [ ] Lead conversion funnel: new → contacted → working → qualified → converted (with drop-off %)
- [ ] Opportunity pipeline chart: value over time
- [ ] Opportunity conversion funnel: qualification → needs_analysis → proposal → negotiation → won
- [ ] Campaign comparison report: side-by-side ROI for multiple campaigns
- [ ] Activity leaderboard: calls, emails, meetings per SDR/AE
- [ ] Lead aging: flag leads with no activity in 7+ days
- [ ] Opportunity aging: flag deals stale for 14+ days
- [ ] Bulk lead import: CSV upload with column mapping, duplicate detection
- [ ] Global search across leads, accounts, contacts, opportunities
- [ ] Customizable opportunity stages (add/remove/reorder)
- [ ] waymaker.config.ts manifest for Host Schema
- [ ] Mobile responsive layouts
- [ ] Deploy-ready configuration

### Out of Scope

- Contact-centric model (that's the `crm-b2b-contact` blueprint)
- B2C / person account model (that's the `crm-b2c` blueprint)
- Web-to-Lead forms (use Commander Journeys or external form tools)
- Territory management
- Forecasting / quota management
- CPQ (Configure, Price, Quote)
- Email inbox sync (manual logging for now)

## Success Criteria

| Metric | Target |
|--------|--------|
| Lead conversion | Convert Lead creates Account + Contact + Opportunity in < 3 seconds |
| Duplicate detection | Existing accounts detected by name/domain before creation |
| Pipeline accuracy | Only converted opportunities appear on pipeline board |
| Activity logging | < 30 seconds to log a call or create a follow-up task |
| Campaign ROI | Revenue traced back to originating campaign |
| Build time (with AI) | < 8 hours for all 4 phases |

## Dependencies

- WaymakerOS organization with Commander access
- Commander Tables for data storage
- Commander Tasks API for task creation
- Commander Calendar API for meeting scheduling
- (Optional) Commander Journeys for lead nurture sequences
- (Optional) Commander Docs for proposals/contracts

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Conversion logic complexity | Medium | High | Atomic transaction, rollback on partial failure |
| Duplicate accounts | High | Medium | Domain + name matching with user confirmation dialog |
| Lead scoring too simple | Medium | Low | Start manual, extend with auto-scoring in Phase 3 |
| SDR/AE handoff unclear | Low | Medium | Ownership transfers logged as activities |
| Large lead imports (10K+) | Low | Medium | Batch processing with progress indicator |

## Open Questions

- [x] Leads separate from Contacts? **Yes — that's the entire point of this model**
- [x] What happens to Lead activities after conversion? **Linked/copied to the new Contact record**
- [x] Can you convert without creating an Opportunity? **Yes — Opportunity is optional in the conversion dialog**
- [x] Can a Lead be re-opened after disqualification? **Yes — status can be changed back to "working" (with activity log)**
- [ ] Web-to-Lead? **Out of scope — future platform feature via Journeys or external forms**
