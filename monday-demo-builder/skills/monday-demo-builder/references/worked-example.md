# Worked example — quality anchor (Sales → onboarding)

> Reference for the `monday-demo-builder` skill. A full transcript-to-blueprint-to-build
> example showing the expected depth and the prospect's-language discipline.

## Transcript snippet

> **AM:** "So just to recap, you mentioned your sales team is using Salesforce but you're losing visibility into post-sale handoffs to the CS team. Is that the main pain?"
> **Prospect:** "Yeah. We close a deal in Salesforce, then it disappears for two weeks before CS gets to it. Half the time onboarding kicks off late and we're already eating into our 90-day success window."
> **AM:** "How big is the team?"
> **Prospect:** "30 reps, 12 CSMs, growing. Series B SaaS, mostly mid-market — average ACV around $80k."
> **Prospect:** "We also have a Slack thing where the AE pings the CSM when a deal closes but it gets lost in the noise."

## Expected blueprint (abbreviated)

**Phase 1:**

- Industry: B2B SaaS, Series B
- Pain: deal-to-onboarding gap, 2-week dead time, late onboarding start
- Scale: 30 reps + 12 CSMs, $80k ACV mid-market
- Multi-product: CRM (deals) + Work Management (CS onboarding)
- Integration: Slack (UI-only — flagged for Phase 5)

**Phase 2:**

- Products: `crm` + `core`
- Cross-product flow: Won Deal in CRM → triggers Onboarding Project in Work Management
- Workspaces:
  - CRM workspace: `[Sales]` folder (BRIGHT_BLUE) — native boards used
  - Main workspace: `[Customer Success]` folder (DONE_GREEN) — new boards created
- Boards:
  - `[Sales] Accounts` — native CRM board, seeded with prospect accounts
  - `[Sales] Deals` — native CRM board, seeded with pipeline
  - `[CS] Onboarding Projects` (public, Main workspace) — `board_relation` → Deals; mirror Account name, Deal Value, Close Date
  - `[CS] Onboarding Tasks` (public, Main workspace) — `board_relation` → Onboarding Projects; subitems
- Relational map:
  - Deals → Accounts: native CRM `board_relation` (already exists)
  - Onboarding Projects.Linked_Deal → Deals (`board_relation` + mirrors: Account, Deal Value, Close Date)
  - Onboarding Tasks.Project → Onboarding Projects (`board_relation`)
- Key columns:
  - Deals: Stage (status: Discovery/Qualified/Proposal/Negotiation/Won/Lost), Deal Value (numbers), Close Date (date), Win Probability (numbers), Source (dropdown)
  - Onboarding Projects: Status (status: Not Started/Kickoff/Active/At Risk/Live), Health (rating), Onboarding Lead (people), Days Since Close (formula), Timeline (timeline), Linked Deal (board_relation → Deals)
  - Onboarding Tasks: Status, Priority, Owner (people), Due Date (date)
- Dashboard: **Sales-to-CS Pipeline Health** (Main workspace)
  - NUMBER: avg days Closed Won → Onboarding Kickoff
  - CHART (column): Onboarding projects by Status
  - BATTERY: % of wins hitting kickoff within 5 business days
  - CALENDAR: upcoming kickoffs by Onboarding Lead
- Doc: "Onboarding Playbook — 90-day path to value" (Main workspace)

**Phase 5 handoff pre-identified:**

- Slack integration: Workspace → Integrations → Slack → Add to workspace
- Automation: Deals board → Automate → "When Status changes to Closed Won, create item in [CS] Onboarding Projects"
- Kanban view on Onboarding Projects: Board → "+" → Kanban → group by Status
- Email & Activities: Account → Email & Activities → Connect mailbox

## Final summary format

> ✅ Built `[DEMO] <Prospect> — Sales-to-Onboarding`.
>
> **What was built:**
> - `[Sales] Accounts` — 8 items (seeded native CRM board)
> - `[Sales] Deals` — 12 items (seeded native CRM board)
> - `[CS] Onboarding Projects` — 6 items
> - `[CS] Onboarding Tasks` — 24 items
> - Cross-board wiring: Onboarding Projects linked to Deals via `board_relation`; Account name, Deal Value, and Close Date mirrored onto each onboarding project; Tasks linked to their parent Onboarding Project
> - Dashboard: Sales-to-CS Pipeline Health — NUMBER / CHART / BATTERY / CALENDAR
> - Doc: Onboarding Playbook — 90-day path to value
>
> **Manual steps for the AM:**
> - Slack: Workspace → Integrations → Slack → Add to workspace
> - Automation: Deals board → Automate → "When Status changes to Closed Won, create item in [CS] Onboarding Projects"
> - Kanban view: Onboarding Projects → "+" → Kanban → group by Status
> - Email & Activities: Account → Email & Activities → Connect mailbox
>
> Want me to add a Customer Health board, scale to ~50 deals, or tear this down?
