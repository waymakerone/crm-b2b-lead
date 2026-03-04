---
sync:
  type: doc
  layer: CRM B2B Lead
build:
  status: todo
  phase: 2
  priority: P0
  depends_on: ["phase-1-leads-and-conversion"]
  started_at: null
  completed_at: null
---

# Phase 2: Pipeline & Activities

**Goal:** Opportunity pipeline board, full detail pages for Accounts/Contacts/Opportunities, complete activity system with Commander Task and Calendar integration. The AE workspace.

**PRD Reference:** `docs/01-planning/product-requirements/crm-b2b-lead-prd.md` — Phase 2

---

## What to Build

### 1. Opportunity Pipeline (/opportunities)

Kanban board — the AE's primary workspace. Only shows converted, qualified opportunities. No leads.

**Layout:**
- Columns from `crm_opportunity_stages`, ordered by `display_order`
- Column headers: stage name + opportunity count + total value
- "Closed Won" and "Closed Lost" collapsed on the right (totals visible, expandable)

**Opportunity Cards:**
- Account name (bold)
- Opportunity name (subtitle)
- Amount (formatted currency)
- Expected close date
- Owner avatar/initials
- Primary contact name (small text)
- Campaign badge (if linked, small text)
- Color-coded borders: overdue (red), closing this week (gold), normal (default)

**Drag and Drop:**
- Use `@dnd-kit/core` and `@dnd-kit/sortable`
- Drag to new column → calls `opportunities.moveStage(id, newStage)`
- Moving to Closed Won/Lost → prompt for reason (modal)
- Probability auto-updates from stage defaults

### 2. Account Detail (/accounts/:id)

**Layout:**
```
┌──────────────────────────────────────────────────┐
│ [← Back to Accounts]                   [Edit]    │
│                                                  │
│ ┌──────────────────┐  ┌────────────────────────┐ │
│ │ Account Info      │  │ Opportunities          │ │
│ │ Name: Acme Corp   │  │ ● Acme Q2 — $50K      │ │
│ │ Domain: acme.com  │  │   Proposal stage       │ │
│ │ Industry: Tech    │  │ ● Acme Pilot — $5K     │ │
│ │ Size: 51-200      │  │   Closed Won ✓         │ │
│ │ Type: Customer    │  │                        │ │
│ │ Owner: AE Name    │  │ [+ New Opportunity]    │ │
│ │                   │  ├────────────────────────┤ │
│ │ Revenue Summary   │  │ Contacts               │ │
│ │ Won: $55,000      │  │ ★ Jane Smith (primary) │ │
│ │ Pipeline: $50,000 │  │   VP Sales             │ │
│ │ Total Opps: 3     │  │   Bob Jones            │ │
│ │ Win Rate: 50%     │  │   Engineer             │ │
│ │                   │  │                        │ │
│ │ [Edit Account]    │  │ [+ Add Contact]        │ │
│ └──────────────────┘  └────────────────────────┘ │
│                                                  │
│ ┌──────────────────────────────────────────────┐ │
│ │ Activity Timeline (all contacts + opps)      │ │
│ │ Aggregated from all contacts and             │ │
│ │ opportunities at this account                │ │
│ └──────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

**Revenue Summary:**
- Won revenue: sum of Closed Won amounts
- Pipeline: sum of open opportunity amounts
- Total opportunities: count
- Win rate: won / (won + lost)

**Activity timeline:**
- Aggregates activities from ALL contacts and opportunities at this account
- Sorted reverse chronological

### 3. Contact Detail (/contacts/:id)

**Layout:**
```
┌──────────────────────────────────────────────────┐
│ [← Back to Account: Acme Corp]        [Edit]    │
│                                                  │
│ ┌──────────────────┐  ┌────────────────────────┐ │
│ │ Contact Info      │  │ Activity Timeline      │ │
│ │ Name: Jane Smith  │  │ [Log Call] [Task]      │ │
│ │ Email: jane@acme  │  │ [Meeting] [Note]       │ │
│ │ Phone: +1 555...  │  │ [Email]                │ │
│ │ Title: VP Sales   │  │                        │ │
│ │ Account: Acme Corp│  │ ── Today ──            │ │
│ │ ★ Primary Contact │  │ 📞 Call: connected 5m  │ │
│ │                   │  │ ✅ Task: Send proposal  │ │
│ │ ┌──────────────┐  │  │                        │ │
│ │ │ Converted from│  │  │ ── Yesterday ──       │ │
│ │ │ Lead #142     │  │  │ 📝 Note: "Interested" │ │
│ │ │ Feb 24, 2026  │  │  │                        │ │
│ │ │ Source: Event  │  │  │                        │ │
│ │ └──────────────┘  │  │                        │ │
│ │                   │  │                        │ │
│ │ Opportunities     │  │                        │ │
│ │ • Acme Q2 ($50K)  │  │                        │ │
│ │   Proposal stage  │  │                        │ │
│ └──────────────────┘  └────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

**"Converted from Lead" panel:**
- Shows if this contact was created via lead conversion
- Links back to the original Lead record
- Shows original lead source and campaign

### 4. Opportunity Detail (/opportunities/:id)

**Layout:**
- **Stage progression bar** at top — connected steps, current highlighted in gold
- Click a stage to advance (with confirmation for Closed)
- Opportunity info: name, amount, probability, expected close, owner
- Account link + primary contact link
- Campaign link (if originated from a campaign lead)
- Activity timeline with quick actions
- Edit opportunity button

### 5. Activity System

Same concept as the contact-centric CRM but with Lead support:

| Type | What Happens | Commander API |
|------|-------------|---------------|
| `note` | Freeform text saved to timeline | None |
| `call` | Log outcome + notes + duration | None |
| `task` | Creates Commander Task, stores `commander_task_id` | `commander-task-operations` → `create_task` |
| `meeting` | Creates Commander Calendar event, stores `commander_event_id` | `commander-calendar-operations` → `create_event` |
| `email` | Logs email sent (manual) | None |

**Activity can be linked to:**
- A Lead (pre-conversion)
- A Contact (post-conversion)
- An Opportunity (deal-specific)
- Or both a Contact AND an Opportunity

**Creating a Task from CRM:**
```
POST /functions/v1/commander-task-operations
Body: {
  "action": "create_task",
  "data": {
    "title": "Send proposal to Jane Smith — Acme Q2",
    "description": "Proposal for Q2 expansion, $50K deal",
    "due_date": "2026-03-05",
    "priority": "high",
    "tags": ["crm", "opportunity:acme-q2"]
  }
}
```

**Creating a Meeting from CRM:**
```
POST /functions/v1/commander-calendar-operations
Body: {
  "action": "create_event",
  "data": {
    "title": "Needs Analysis — Acme Corp",
    "start_time": "2026-03-03T14:00:00Z",
    "end_time": "2026-03-03T15:00:00Z",
    "description": "Deep dive on requirements with Jane Smith, VP Sales"
  }
}
```

### 6. Activity Form Components

**Log Call Dialog:**
- Fields: outcome (connected, voicemail, no_answer, wrong_number), duration (minutes), notes
- Updates `last_contacted_at` on lead or contact

**Create Task Dialog:**
- Fields: title (pre-filled: "Follow up with {name} re: {opportunity}"), description, due date, priority
- Creates Commander Task, shows confirmation with Taskboard link

**Schedule Meeting Dialog:**
- Fields: title, date/time, duration, description
- Creates Commander Calendar event, shows confirmation

**Add Note Dialog:**
- Fields: subject (optional), body

**Log Email Dialog:**
- Fields: subject, body, direction (sent/received)

### 7. Account List (/accounts)

Table view:
- Columns: name, domain, industry, type (badge), contacts count, open opps, pipeline value, owner
- Search by name or domain
- Filter by type and industry
- "New Account" button

---

## Acceptance Criteria

- [ ] Opportunity Pipeline shows qualified deals in Kanban columns
- [ ] Drag-and-drop moves opportunities between stages, persists to DB
- [ ] Moving to Closed Won/Lost prompts for reason
- [ ] Account Detail shows contacts, opportunities, revenue summary, aggregated activities
- [ ] Contact Detail shows "Converted from Lead" panel with link to original lead
- [ ] Opportunity Detail shows stage progression bar
- [ ] "Log Call" creates activity, updates `last_contacted_at`
- [ ] "Create Task" creates Commander Task AND CRM activity linked to it
- [ ] "Schedule Meeting" creates Commander Calendar event AND CRM activity
- [ ] Activities on converted leads are visible on the Contact timeline
- [ ] Account List with search and filters
- [ ] Navigation between Lead → Contact → Account → Opportunity works via links
