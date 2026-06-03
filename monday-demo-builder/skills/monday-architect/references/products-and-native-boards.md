# Products, native boards, workspaces & object types

> Verified reference for the `monday-architect` skill (progressive disclosure). Loaded on demand; the decision core lives in that skill's SKILL.md.

### monday CRM (`crm` workspace) — verified native board set

The CRM workspace provisions these boards. Every monday CRM account has them.

| Board | `item_terminology` | Key native column IDs | Cross-wired to |
|---|---|---|---|
| **Leads** | `Lead` | `lead_status` (status: New Lead/Attempted to contact/Contacted/Qualified/Unqualified), `lead_email` (email), `lead_phone` (phone), `lead_company` (text), `lead_owner` (people), `date` (Last activity), `location5` (location), `long_text` (Comments) | Contacts (duplicate detector), Accounts (existing account detector) |
| **Contacts** | `Contact` | `contact_email` (email), `contact_phone` (phone), `contact_company` (text), `contact_account` (board_relation → Accounts), `contact_deal` (board_relation → Opportunities), `status` (Type: Customer/Vendor/Partner/VIP/N/A), `title5` (dropdown: **CEO/COO/CIO only** — API rejects any other label; see §25 Round-4), `date` (Last interaction), `location` (location) | Accounts, Opportunities, Leads |
| **Accounts** | `Account` | `company_domain` (link), `industry` (dropdown — 51 standard industry values), `status` (Account Status: Buying process/Client/Past Client), `account_contact` (board_relation → Contacts), `account_deal` (board_relation → Opportunities), `mirror` (Account Value — SUM mirror of Won deal values), `employee_count` (text), `headquarters_loc` (text) | Contacts, Opportunities, Projects (post-sale) |
| **Opportunities / Deals** | `Opportunity` | `deal_stage` (status: New/Discovery/Proposal/Negotiation/Legal/Won/Lost), `deal_value` (numbers $), `deal_owner` (people), `deal_expected_close_date` (date), `deal_close_date` (date), `deal_contact` (board_relation → Contacts), `deal_close_probability` (formula % from stage), `deal_forecast_value` (formula: value × probability), monthly actual formulas (Jan–Dec), `deal_length` (formula), `dup__of_deal_age` (stage length formula) | Contacts, Accounts, Legal, Onboarding/Finance |
| **Sales Activities** | `Activity` | `activity_item` (board_relation → Leads/Contacts/Accounts/Opportunities/Activities), `activity_owner` (people), `activity_start_time` / `activity_end_time` (date), `activity_status` (status: Open/Done), `activity_type` (status: Meeting/Call summary), `long_text` (Description) | All CRM boards (unified activity log) |
| **Products & Services** | `Product` | Product/pricing catalog linked to Opportunities | Opportunities |
| **Accounts Management / Onboarding** | `Client` | Post-sale project tracker — cross-wired to Accounts and Opportunities | Accounts, Opportunities, Projects |

**CRM relationship wiring (all pre-built on a native CRM workspace):**
- Leads ↔ Contacts: duplicate email detector via `board_relation` + status indicator ("New Contact" / "Existing Contact")
- Leads ↔ Accounts: duplicate company detector via `board_relation` + status indicator ("New Account" / "Existing Account")
- Contacts ↔ Accounts: primary relationship — `contact_account` column links each contact to their account
- Contacts ↔ Opportunities: `contact_deal` column — deal history per contact
- Accounts ↔ Opportunities: `account_deal` + `mirror` (Account Value) — all deals visible on the account
- Accounts → Onboarding: `link_to_accounts_management` — won deal → onboarding project
- Opportunities → Legal: `board_relation` column (ID varies by account, e.g. `connect_boards41`) → Legal Requests board
- Opportunities → Finance/Invoices: `board_relation` column (ID varies by account, e.g. `connect_boards4`) → Finance & Collections board

> **Column ID caveat:** The `contact_*`, `deal_*`, `activity_*`, `lead_*` prefixes are stable native conventions. Columns like `connect_boards41` / `connect_boards4` are auto-generated IDs that vary per account — always call `workspace_info` + `get_board_info` to resolve the actual column IDs on the target account before writing payloads.

**When building CRM:** seed the existing native boards. Groups represent pipeline stages (e.g. "Working pipeline" / "Closed Won" / "Lost"). Do not create new Lead/Contact/Account/Deal boards.

---

### monday Dev (`software` workspace) — TWO native variants

The Dev workspace can be initialised in two distinct ways depending on which template the user picks in the UI. **Both produce native boards. Identify which variant the account has via `workspace_info` before designing.**

#### Variant A — "Simple" Dev setup (Feature-request-driven)

Provisioned when the user picks the basic Dev template:

| Board | `item_terminology` | Key columns | Purpose |
|---|---|---|---|
| **Feature request** | `Request` | `color_*` (Type: New feature/Feature improvement/Performance improvement), `long_text_*` (request + why), `email_*` (user email), `button` (To prioritize?) | External intake — comes with a Form view pre-built |
| **Product backlog** | `Feature` | `assignee` (people), `backlog_timeline` (timeline), `status` (Ideation/Ready for dev/Dev in progress/Done/Stuck/Deprioritized), `backlog_impact` (High/Medium/Low), `backlog_effort` (XL/L/M/S), `backlog_priority` (Critical/High/Medium/Low), `backlog_files` (file — for PRD attachment) | Quarterly planning with Gantt + Roadmap views |
| **Customer feedback** | `Feedback` | `status` (Not read/To follow-up/Treated), `rating_*` (rating), `long_text_*` (what to improve), `dropdown_*` (Tags), `email_*` (user email) | External feedback — comes with a Form view pre-built |
| **Quarterly goals** | `Goal` | `person` (people/Owners), `numeric_*` (Q start/Q goal/Q current — all %), `formula_*` (% of goal achieved) | OKR/goal tracking with Goal progression + Hierarchy views |
| **PRD template** | `item` | `files` (file) | Product requirements doc — uses FeatureBoardView (Doc app) |

#### Variant B — "Sprint Template" setup (Engineering team)

Provisioned when the user adds the **sprint template** in the UI (`Workspace → Add → Templates → Sprint`). This is what most engineering teams actually want:

| Board | `item_terminology` | Key columns | Purpose |
|---|---|---|---|
| **Sprints** | `Sprint` | `sprint_goals` (long_text), `sprint_timeline` (timeline), `sprint_start_date` / `sprint_end_date` (date), `sprint_capacity` (numeric), `sprint_tasks` (board_relation → Tasks), `sprint_activation` (status — `v` is Active), `sprint_completion` (checkbox) | Sprint container — one item per sprint |
| **Tasks** | `Task` | `task_owner` (people), `task_status` (status), `task_priority` (status), `task_type` (status — Bug/Feature/Test), `task_estimation` / `task_actual_effort` (numeric — hours or points), `task_epic` (board_relation → Epics), `task_sprint` (board_relation → Sprints) | Sprint task list |
| **Epics** | `Epic` | `epic_owner` (people), `timeline` (timeline), `epic_status` (status — In Progress/Planned/Backlog), `epic_priority` (status — Critical/High/Medium/Low), `epic_tasks` (board_relation → Tasks), `monday_doc_v2` (doc) | Quarterly epic planning |
| **Bugs Queue** | `Bug` | `people1` (Reporter), `bug_status` (status — Open/In Progress/Done/Pending Deploy/etc), `priority_1` (status), `time_tracking` (Time until resolution), `bug_tasks` (board_relation → Tasks) | Bug intake & triage |
| **Retrospectives** | `Retrospective` | (varies — typically `what_went_well`, `what_didnt`, `action_items`) | Sprint retro notes |
| **Capacity** | `Capacity` | (varies — typically `person`, `available_hours`, `sprint_link`) | Per-person sprint capacity |
| **Getting Started** | (doc) | — | Auto-generated walkthrough doc |

**⚠️ `create_sprint` mutation does NOT exist in API release `2026-07`.** Verified by full schema introspection. The skill has historically claimed this mutation creates the paired Sprints/Tasks boards — that is wrong for the current API.

**The native sprint board set must be provisioned via the UI.** If the user wants engineering sprint workflow:
1. Tell them to open the Dev workspace → Add → Templates → **Sprint** in the monday UI.
2. Wait for them to confirm the boards exist.
3. Then use `workspace_info` to get the auto-generated board IDs (Sprints, Tasks, Epics, Bugs Queue).
4. Seed sprints by creating items on the **Sprints** board directly. Add tasks via `create_item` on the Tasks board with `task_sprint: {item_ids: [<sprint_item_id>]}` and `task_epic: {item_ids: [<epic_item_id>]}`.

**Querying sprint state:**
- `get_monday_dev_sprints_boards` — find sprint board pairs (works for both variants).
- `get_sprints_metadata` — sprint definitions on a board.
- `get_sprint_summary` — burnup/burndown/summary for a sprint.

**Native cross-board wiring (sprint template):**
- `task_sprint` (Tasks → Sprints): pre-wired with `boardIds` set
- `task_epic` (Tasks → Epics): pre-wired
- `epic_tasks` (Epics → Tasks): pre-wired (reverse side, auto-populated)
- `bug_tasks` (Bugs Queue → Tasks): pre-wired
- `sprint_tasks` (Sprints → Tasks): pre-wired (reverse side)

These all ship with `boardIds` configured — you do NOT hit the §5 boardIds limitation when using these native columns. **Don't recreate them — use them.**

**Dev board relationships (logical flow):**
- Feature request → Product backlog (Variant A) OR Bugs Queue → Tasks (Variant B): the intake → work conversion
- Product backlog / Tasks → Sprint: pulled in via `task_sprint`
- Customer feedback / Bugs Queue → Tasks: feedback/bug informs sprint priority
- Quarterly goals → Epics: goals link to epics in the active quarter
- **CRM → Dev link:** Variant A uses Customer feedback ↔ CRM Accounts. Variant B has no built-in CRM link — add a `board_relation` from Bugs Queue or Tasks to CRM Accounts manually (and remember §5: the user must configure `boardIds` in the UI).

---

### monday Service (`service` workspace) — native board set

The Service workspace may be **empty on some accounts** (not always seeded at signup). When building Service:

| Board | `item_terminology` | Key columns to create | Purpose |
|---|---|---|---|
| **Tickets** | `Ticket` | `status` (New/Open/In progress/On hold/Resolved/Closed), `priority` (status: Critical/High/Medium/Low), `ticket_type` (status: Bug/Question/Feature/Other), `assignee` (people), `reporter` (people), `due_date` (date), `sla_breach` (formula or date), `contact` (board_relation → CRM Contacts), `account` (board_relation → CRM Accounts) | Core support queue |
| **Knowledge Base** | `Article` | `category` (dropdown), `status` (Draft/Review/Published), `assignee` (people), `related_ticket` (board_relation → Tickets) | Self-service docs, links to ticket patterns |
| **SLA Policies** | `Policy` | SLA tiers with response/resolution time targets | Reference board for SLA formula logic |

**Service ↔ CRM wiring (critical for demos):**
- Tickets link to CRM Contacts (`board_relation`) — the support agent sees who the customer is
- Tickets link to CRM Accounts — account-level support volume visible on Account board via `mirror`
- Won deals in Opportunities trigger ticket creation (via automation/webhook)

**If the Service workspace is empty:** follow the Case 2 stop rule above — tell the user to open the Service workspace and click "Get started" to initialise the native boards. The schema above is the reference for what to expect once they do. Do NOT create a Tickets board manually; the native board comes with pre-built views, SLA integrations, and form intake that a manually-created board won't have.

---

### monday Work Management (`core` workspace — typically "Main workspace") — native board set

The core workspace is for projects, tasks, ops, OKRs. It is **not** for CRM, Dev, or Service entities.

Native patterns (not fixed templates — varies by account):
- **Project boards** — timeline, owner, status, priority, dependency columns
- **Task boards** — linked to project board via `board_relation`
- **Resource planner** — auto-created alongside project boards when using `create_project`
- **OKR / Goals board** — company/department/team goal hierarchy
- **Portfolio** — created via `create_portfolio` + `connect_project_to_portfolio` (NOT a regular board)

**Work Management ↔ CRM wiring (the "deal-to-project" handoff):**
- Won Deal → create a Project in Work Management (most valuable cross-product flow)
- Project board → link back to Account and Deal in CRM
- Project milestones → Service tickets (issue tracking for deliverables)

---

### monday Projects (`project_management`) — Portfolio architecture

`project_management` is a **layer on top of core boards**, not a separate workspace. It adds:
- **`create_project`** → generates TWO boards: a parent project board + a tasks board with a 14-column native template
- **`create_portfolio`** → generates a portfolio board with 11 native columns (health, progress mirror, timeline mirror, etc.)
- **`connect_project_to_portfolio`** → takes the **tasks board ID** (lower-numbered), not the parent project board ID

Portfolio board native columns: `portfolio_project_owner` (people), `portfolio_project_rag` (status: At risk/On track/Off track), `portfolio_project_progress` (mirror — auto-rollup of `project_status`), `portfolio_project_priority` (Critical/High/Medium/Low), `portfolio_project_step` (Upcoming/In progress/Completed), `portfolio_project_planned_timeline` (timeline), `portfolio_project_actual_timeline` (mirror — rollup of `project_timeline`), `portfolio_project_doc` (doc), `portfolio_project_scope` (text), `portfolio_project_link` (board_relation to all connected projects).

---

### monday Marketer (`marketing_campaigns` workspace)

Native board set (varies by account — always check `workspace_info` first):

| Board | `item_terminology` | Key columns | Purpose |
|---|---|---|---|
| **Campaigns** | `Campaign` | `status` (Planning/Active/Completed/Paused), `owner` (people), `timeline` (timeline), `budget` (numbers $), `channel` (dropdown), `target_audience` (text) | Campaign pipeline |
| **Content Calendar** | `Content` | `status`, `channel` (dropdown), `publish_date` (date), `assignee` (people), `campaign` (board_relation → Campaigns) | Editorial calendar — use Calendar view |
| **Briefs** | `Brief` | `status`, `campaign` (board_relation → Campaigns), `assignee` (people), `due_date` (date) | Creative brief + Doc column for brief content |
| **Assets** | `Asset` | `file` (file column), `status`, `campaign` (board_relation) | Creative asset management |

**Marketer ↔ CRM wiring:**
- Campaigns link to CRM Accounts (target accounts for ABM campaigns)
- Campaign leads → CRM Leads board (form submissions flow in)
- Campaign performance dashboards pull from both Campaigns board + CRM deal data

**When to put Campaigns inside the CRM workspace instead:** for SMB demos where the marketing team is the same people as the sales team, or where every campaign target list is sourced from the CRM Accounts/Leads, the cleanest setup is a `[Campaigns & Marketing]` folder **inside the CRM workspace** (not a separate Marketing workspace). This keeps the cross-board navigation natural — every Account is one click from the campaigns targeting it. Build a separate `marketing_campaigns` workspace only when the marketing team is functionally distinct (different users, different cadence, different reporting) — typical at companies of 50+ FTEs.

---

### Cross-product architecture: the full monday flywheel

The most powerful monday demos show how all products form a single connected system. The standard end-to-end flow:

```
[Marketer] Campaign → Lead Form
        ↓
[CRM] Lead → Contact → Account → Opportunity (Deal)
        ↓ (Deal Won)
[Core/Projects] Project created → tasks assigned → milestones
        ↓
[Dev] Feature requests from customer → Product backlog → Sprint
        ↓
[Service] Support ticket → resolved → linked back to Account
        ↓
[CRM] Account Value updated via mirror → renewal opportunity created
```

**How to wire it via MCP:**
1. CRM → Projects: add a `board_relation` from Opportunities to a Work Management project board. When a deal is won, create a project item and link it.
2. CRM → Service: add a `board_relation` from CRM Accounts to Tickets board. Use a `mirror` on Account to show open ticket count.
3. Dev → CRM: Customer feedback board has a `board_relation` to CRM Accounts — enabling ARR-weighted feature prioritization.
4. Projects → Dev: sprint tasks can link back to project milestones via `board_relation`.
5. Any product → Dashboard: cross-product dashboards pull widgets from boards across all workspaces.

**Dashboard as the cross-product lens:**
Dashboards are workspace-scoped in creation but widgets can pull from boards in any workspace. Always build at least one cross-product dashboard for demos: CRM pipeline funnel + open tickets + active sprints + project health — on a single dashboard.

---

### Build checklist for any product build

Before writing any mutation:

- [ ] `get_user_context` — confirm which products are enabled on this account
- [ ] `list_workspaces` — find the workspace for each product you'll build in
- [ ] `workspace_info(workspace_id)` for each target workspace — find existing native boards
- [ ] For each entity you need: use existing board if present, create only if absent
- [ ] Place new boards in the correct product workspace, organised into folders
- [ ] Wire cross-product `board_relation` columns for any flow that spans products
- [ ] Build a cross-product dashboard showing the full flow

---

## 2. Workspaces and folders

- **Workspace** — top-level scope. `WorkspaceKind`: `open`, `closed` (enterprise-only), `template`.
  - Tools: `create_workspace`, `update_workspace`, `list_workspaces`, `workspace_info`.
  - Filter workspaces by product via `WorkspacesQueryInput.kind` (raw GraphQL).
- **Folder** — group related boards/dashboards/docs inside a workspace.
  - Tools: `create_folder` (16-color enum: `AQUAMARINE`, `BRIGHT_BLUE`, `BRIGHT_GREEN`, `CHILI_BLUE`, `DARK_ORANGE`, `DARK_PURPLE`, `DARK_RED`, `DONE_GREEN`, `INDIGO`, `LIPSTICK`, `NULL`, `PURPLE`, `SOFIA_PINK`, `STUCK_RED`, `SUNSET`, `WORKING_ORANGE`; plus `customIcon` and `fontWeight` enums), `update_folder`, `delete_folder`, `move_object`.
  - **`create_board` does NOT take a folder ID** — boards land at workspace root. After creating, call `move_object` with `objectType: "Board"`, `id: <boardId>`, `parentFolderId: <folderId>`. (Verified end-to-end.)
  - `workspace_info` returns up to 100 of each object type per workspace; paginate for larger.
- Use folders. Don't dump 20 boards at the workspace root.
- Naming: prefix related objects (`[Sales] Leads`, `[Sales] Deals`, `[Sales] Accounts`) so they cluster in lists and search.

### CRITICAL: Match product kind to the correct existing workspace

**Before creating any board, you MUST identify the right workspace for the chosen product kind.** Do NOT create a generic `open` workspace and build everything there. Follow the 8-step Required Workflow in §1.5 — it covers workspace identification, native-board discovery, and the two STOP cases.

Quick reference: CRM boards → "CRM" workspace; Dev boards → "Dev" workspace; Service boards → "Service" workspace; Work Management → `core` workspace (typically "Main workspace"). Pass `workspace_id` to `create_board`.

**Anti-pattern to refuse:** building CRM, Dev, or Service boards inside the default Work Management workspace. monday.com maintains product-specific workspaces with native context (column templates, views, integrations). Boards placed in the wrong workspace lose that context and are harder for end-users to find.

---

## 3. Pick the right object type

| User wants… | Build | Tool |
|---|---|---|
| Generic project / task list | **Board** (kind: `public` / `private` / `share`) | `create_board` |
| Hierarchical work in one board | **Sub-items** | `subitems` / `subtasks` column on the board |
| Cross-board project rollup / portfolio | **Project + Portfolio** (NOT regular board + mirrors) | `convert_board_to_project`, `create_project`, `create_portfolio`, `connect_project_to_portfolio` (raw GraphQL — see §3.5 for verified arg shapes) |
| Narrative content (PRD, meeting notes, wiki, SOP) | **Doc** | `create_doc` (NOT a long-text column) |
| Intake from humans | **Form** bound to a board | `create_form`, `form_questions_editor`, `update_form`, `get_form` |
| Cross-board visualization | **Dashboard + widgets** | `create_dashboard`, `create_widget` |
| Sprint/epic tracking | **Dev sprint board** | Native `software` workspace + sprint queries |
| Saved view of a board | **View** (table or app-embed only — see note) | `create_view`, `update_view`, `delete_view`; pre-fetch `get_view_schema_by_type(type: <ViewKind>, mutationType: CREATE)` |
| Reusable column definitions | **Managed column** | `create_status_managed_column`, `create_dropdown_managed_column`, then `update_status_managed_column`, `update_dropdown_managed_column`, `activate_managed_column`, `deactivate_managed_column`, `delete_managed_column` |
| Provision from template | `use_template(template_id, ...)` | Creates a board/workspace from a saved template |

**Important:** the `BoardKind` enum exposes only `private` / `public` / `share` — visibility, not template type. There is NO "portfolio board kind". Portfolios are built from **Projects** (`create_project` / `create_portfolio` / `connect_project_to_portfolio`) — a separate object hierarchy. To convert an existing board into a project, use `convert_board_to_project`.

**Multi-level boards (the hidden board hierarchy).** Separate from `BoardKind`, the live `BoardHierarchy` enum has two values: **`classic`** and **`multi_level`**. Multi-level boards support deeper nesting than a single subitems layer (up to ~5 layers). Two gotchas that bite raw queries:
- **They're excluded from the default `boards` query** — pass the `hierarchy_types` argument (e.g. include `multi_level`) or you simply won't see them.
- **Rollup columns return empty by default** — a rollup aggregates child-item values, but the API returns blank values unless you request `capabilities: [CALCULATED]` on the column.
- `create_board` does **not** currently expose creating a multi-level board via the API (provision in UI, then operate on it). Treat multi-level as "query-and-edit," not "create," for now.

### 3.5 Project & Portfolio mutation shapes (verified end-to-end)

- **`create_project(input: CreateProjectInput!)`** — `input` fields: `name` (required), `board_kind` (required: `public`/`private` — `share` not supported), optional `template_id`, `companions: ["resource_planner"]`, `workspace_id`, `folder_id`, `callback_url`. Returns `CreateProjectResult { success, message, error, process_id }`. **In practice `process_id` came back as `null`** despite the docs implying async — the project boards appear in the workspace immediately. Possibly synchronous-but-callback-supported.
  - **Each `create_project` call produces TWO boards:** a parent project board + a tasks board with **2 seed tasks (Task 1, Task 2) and a 14-column native template**: `project_owner` (people), `project_resource` (board_relation), `project_status` (status), `project_priority` (status), `project_timeline` (timeline), `project_dependency` (dependency), `project_planned_effort`/`project_effort_spent`/`project_duration`/`project_budget` (numbers), `project_task_completion_date` (date), `subtasks_*` (subtasks), and a back-link `board_relation` to the parent.
  - **Of those two boards, only the lower-numbered (tasks) board IS "the project"** for `connect_project_to_portfolio` — the higher-numbered parent board is NOT recognized. Verified error: `"Failed to connect project to portfolio. the following boards are not projects: [<parent_id>]"`.
  - To find the project's task-board ID, query `boards(workspace_ids: [...])` after creation — both auto-generated boards share the project name.
- **`create_portfolio(boardName: String!, boardPrivacy: String!, destinationWorkspaceId: Int)`** — note `destinationWorkspaceId` is `Int` (despite IDs elsewhere being strings). Returns `CreatePortfolioResult { success, message, solution_live_version_id }`. **The `solution_live_version_id` is the version ID of the underlying portfolio TEMPLATE — it's the same for every portfolio you create. It is NOT the portfolio's board ID.** Query `boards(workspace_ids: [...])` after creation to find the new portfolio board (named with `boardName`).
- **Portfolio board structure (verified — 11 native columns):** `name`, `portfolio_project_owner` (people), `portfolio_project_rag` (status — Project Health, default labels: At risk / On track / Off track), `portfolio_project_progress` (mirror — auto-rolls up `project_status` from connected tasks boards), `portfolio_project_priority` (status — Critical/High/Medium/Low), `portfolio_project_step` (status — Stage: Upcoming/In progress/Completed), `portfolio_project_planned_timeline` (timeline), `portfolio_project_actual_timeline` (mirror — auto-rolls up `project_timeline`), `portfolio_project_doc` (doc), `portfolio_project_scope` (text — Description), `portfolio_project_link` (board_relation to all connected projects).
- **`connect_project_to_portfolio(projectBoardId: ID!, portfolioBoardId: ID!)`** — both are board IDs. Returns `ConnectProjectResult { success, message, portfolio_item_id }`. **Side effects:**
  - Creates a new item on the portfolio board (one item per project) — the `portfolio_item_id` returned is that item.
  - Adds the project's tasks board ID to `portfolio_project_link.boardIds` (the board_relation column's settings).
  - Wires the mirror columns' `displayed_linked_columns` to point at `project_status`/`project_timeline` on the connected boards.
  - **However**, the `portfolio_project_link` value on each portfolio item is initially null — you may need to populate it manually if the mirrors don't auto-resolve (test in the UI). The `connect_project_to_portfolio` mutation appears to wire structure but not item-level links.
- **`convert_board_to_project(input: ConvertBoardToProjectInput!)`** — async, returns `ConvertBoardToProjectResult` with a `process_id`. **`column_mappings` is REQUIRED** with three required fields: `project_status` (column ID), `project_timeline` (column ID), `project_owner` (column ID — note `"name"` is accepted as a stand-in for the name column). Cannot pass an empty `column_mappings: {}`.

### 3.6 Portfolio quick-start (the demo flow)

To set up a populated portfolio with two connected projects in one go:
1. `create_workspace(name, workspaceKind: "open")` → grab `workspace_id`.
2. `create_portfolio(boardName: "...", boardPrivacy: "public", destinationWorkspaceId: <int>)`.
3. `create_project(input: {name: "Project A", board_kind: public, workspace_id: "..."})` × N projects.
4. Query `boards(workspace_ids: [...])` to find: portfolio board ID, and each project's TWO board IDs. The lower-numbered one of each project is the tasks board (the "real" project).
5. `connect_project_to_portfolio(projectBoardId: <tasks_board_id>, portfolioBoardId: <portfolio_id>)` × N projects.
6. Populate the portfolio item columns (`portfolio_project_rag`, `portfolio_project_priority`, `portfolio_project_step`, `portfolio_project_planned_timeline`, `portfolio_project_scope`, `portfolio_project_owner`) on each portfolio item via `change_item_column_values` with `createLabelsIfMissing: true`.
7. Add real tasks to each project tasks board so the mirror columns have data to roll up.

---

