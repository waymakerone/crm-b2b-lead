# B2B CRM (Lead-Centric)

A Salesforce-style lead-centric B2B CRM built on WaymakerOS. Separate lead qualification workflow, one-click conversion to Account + Contact + Opportunity, campaign ROI tracking, and a clean pipeline that only shows qualified deals.

## What You Get

- **Lead Queue** — Work leads through a structured qualification workflow (new → contacted → working → qualified)
- **Lead Conversion** — One-click convert: Lead → Account + Contact + Opportunity, with duplicate account detection
- **Opportunity Pipeline** — Kanban board of qualified deals only — no unqualified leads polluting your forecast
- **Campaign Tracking** — Know which campaigns generate revenue, not just leads. Full ROI: budget vs closed revenue.
- **Activity Timeline** — Log calls, create tasks, schedule meetings, add notes — all integrated with Commander
- **Dashboard** — Two-part: lead metrics (SDR view) + pipeline metrics (AE view)
- **Lead Scoring** — Auto-scoring based on activity, manual override

## CRM Model

This blueprint uses the **lead-centric** model (like Salesforce):

- **Leads** are a separate object — unqualified people who need to be worked
- SDRs/BDRs qualify leads through a structured status workflow
- When qualified, leads **convert** into Account + Contact + Opportunity
- The **pipeline** only shows converted, qualified opportunities
- **Campaigns** track the full funnel: lead → opportunity → revenue

This model is best for teams with structured sales processes, SDR/AE handoffs, and campaign-driven inbound.

Looking for contact-centric (HubSpot-style)? See the `crm-b2b-contact` blueprint.
Looking for B2C? See the `crm-b2c` blueprint.

## The Conversion Flow

```
Lead (new) → contacted → working → qualified
                                      │
                                      ▼
                              [Convert Lead]
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                  ▼
               Account          Contact           Opportunity
            (or link to       (always new,      (optional, starts
             existing)        linked to acct)    at Qualification)
```

Key: duplicate account detection matches by domain and company name before creating a new one.

## Commander Tools Used

| Tool | How the CRM Uses It |
|------|---------------------|
| **Tables** | All CRM data (leads, accounts, contacts, opportunities, campaigns, activities, stages) |
| **Contacts** | Contact records created at lead conversion — linked via `commander_contact_id` |
| **Tasks** | Follow-up tasks from the CRM appear on your Commander Taskboard |
| **Calendar** | Meetings scheduled from the CRM appear in your Calendar |
| **Journeys** | Email sequences triggered by lead status changes and deal stage changes |
| **Docs** | Proposals and contracts attached to opportunities |
| **Metrics** | Pipeline and campaign metrics on your Commander dashboard |

## How to Build

1. Clone this blueprint into your project
2. Open `CLAUDE.md` — it's the router file for your AI coding tool
3. Read the PRD in `docs/01-planning/product-requirements/`
4. Work through the 4 phase prompts in `docs/02-working/prompts/active/`
5. Point Claude Code, Cursor, or Codex at each phase and build

**Estimated build time:** 6-8 hours across all 4 phases.

## Build Phases

| Phase | What You Get |
|-------|-------------|
| 1 — Leads & Conversion | Schema, Lead Queue, lead status workflow, Convert Lead dialog with duplicate detection |
| 2 — Pipeline & Activities | Opportunity Kanban, Account/Contact/Opportunity detail pages, Commander task/calendar integration |
| 3 — Campaigns & Dashboard | Campaign management + ROI, two-part dashboard, lead scoring, Journeys integration |
| 4 — Reporting & Polish | Funnels, campaign comparison, activity leaderboard, CSV import, global search, mobile |

## Prerequisites

- WaymakerOS organization with Commander access
- Commander Tables, Tasks, Calendar, and Contacts enabled
- (Optional) Commander Journeys for automated email sequences
- (Optional) Commander Docs for proposals/contracts
