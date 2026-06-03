---
name: monday-architect
description: >-
  Operator's manual for building on monday.com via the monday MCP connector.
  Use ANY time the user is building, modifying, or reasoning about a monday.com
  account - workspaces, boards, dashboards, docs, forms, items, columns, widgets,
  CRM pipelines, Dev sprints, projects/portfolios, board_relation/mirror links,
  formulas, automations, triggers, webhooks, audit logs, or anything queryable
  through the monday GraphQL API. Forces correct product/architecture choices and
  routes to verified reference files for column value-shapes, widget catalogs,
  per-product native boards, and error gotchas. Trigger on any mention of
  monday.com, monday boards/dashboards/docs, leads/deals/CRM, sprints/epics, item
  linking, mirrors/portfolios, or the monday MCP tool prefix (e.g. a
  mcp__*_monday_*__ tool).
metadata:
  version: 2026-06-01-patch19
  author: Workiflow (adapted from Greg's monday-architect skill)
---
# monday.com architect — operator's manual

You operate the monday.com MCP connector. The user expects builds that use native monday objects, correct typed schemas, and the full advanced API surface — not generic-board workarounds. This skill is your reference. Work through the relevant sections for the task at hand.

> The API facts in this skill (product kinds, column types, board kinds, widget types, webhook events, dashboard visibility, mutation/query names, value shapes, error codes) were re-verified against the monday GraphQL schema and the live MCP connector on **2026-06-01**; the connector still defaults to API release `release_candidate 2026-07` (now `2026-04` is `current`/stable and `2026-10` is the newest release candidate). Account-specific facts (board IDs, column IDs, group IDs, which products are enabled) MUST come from live MCP calls — never assume.
>
> **To re-verify against your own account and detect drift, run `/refresh-monday-skill`** (sister skill — same repo). Run it monthly, before high-stakes demos, or after any monday API release announcement.
>
> **Behavior may differ by account tier, plan, enabled products, and per-user permissions.** Some mutations cited (e.g. `create_validation_rule`, `update_mute_board_settings` for non-owners, certain bulk-job paths) may return "feature not currently supported" or permission errors depending on your account's configuration. Treat the skill as a map of what *can* exist, not a guarantee of what *will* work for your role.

## 0. Always introspect first — never write payloads from memory

Your training data on monday.com is stale. Before designing or executing, query the live system. **Cache the responses for the duration of the task.**

| When | Tool | Why |
|---|---|---|
| Starting any build | `get_user_context` | Account tier, active products on THIS account (subset of: `core`, `crm`, `software`, `service`, `marketing` / `marketing_campaigns`, `project_management`, `forms`, `whiteboard`), favorites, relevant boards/people. Don't assume which products are enabled — check. Note: `account.products` may return `marketing_campaigns` but the `WorkspacesQueryAccountProductKind` enum (used for raw workspace filtering) only accepts `marketing`. |
| Choosing where to build | `list_workspaces` (paginated) → `workspace_info(workspace_id)` | Lists boards/docs/folders in a workspace; capped at 100 per object type — paginate or filter |
| Designing data model | `get_graphql_schema(operationType: "read"|"write")` | Authoritative list of queries/mutations + all GraphQL type names |
| Drilling into a type | `get_type_details(typeName)` | Fields, args, enum values for a specific type |
| Creating a column | `get_column_type_info(columnType)` | JSON Schema 7 for column settings — required for `create_column` |
| Updating column values | call `get_column_type_schema` (via `all_monday_api`) or pull from `get_column_type_info` | Per-type value JSON shape — required for `change_item_column_values` |
| Creating widgets | `all_widgets_schema` | Full JSON Schema 7 for every widget type |
| Creating views | call `get_view_schema_by_type` (via `all_monday_api`) | Schema for `create_view` settings parameter |
| Anything not covered by a named tool | `all_monday_api` | Raw GraphQL fallback (validate against `get_graphql_schema` first) |
| Searching existing content | `search` | Global search across boards/items/docs |
| Inspecting a board | `get_board_info` → `get_board_items_page` (paginated, with cursor) | Metadata + items. Do NOT call `get_full_board_data` — it is marked internal-only / UI-triggered. |
| Sprints (Dev) | `get_monday_dev_sprints_boards`, `get_sprints_metadata`, `get_sprint_summary` | Native sprint objects |
| Notetaker | `get_notetaker_meetings` | Meeting transcripts/talking points |

**Rule:** if you're about to write a payload, mutation, column-value JSON, or widget config from memory — stop and call the relevant introspection tool first.

---

## 1. Pick the right product

monday.com is **eight product kinds** (verified via `WorkspacesQueryAccountProductKind`):

| `kind` | Marketing name | Use for |
|---|---|---|
| `core` | Work Management | Generic projects, ops, content, OKRs, tasks |
| `crm` | monday CRM | Leads, contacts, accounts, deals, pipelines, sales activity |
| `software` | monday Dev | Sprints, epics, bugs, releases, roadmaps |
| `service` | monday Service | Tickets, SLA, support queues, customer ops |
| `marketing` | monday Marketer | Campaigns, briefs, content calendar |
| `project_management` | monday Projects | Project portfolios, project health, resource planning |
| `forms` | WorkForms | Standalone form-building |
| `whiteboard` | Canvas | Whiteboards |

**Decision rules:**
- "CRM", "leads", "deals", "pipeline", "accounts", "contacts" → `crm`. Never a Status column on a `core` board.
- "Sprint", "epic", "story points", "backlog", "bug tracking" → `software`. Use sprint queries (`sprints`, `get_monday_dev_sprints_boards`, `get_sprint_summary`).
- "Ticket", "SLA", "queue", "first response time" → `service`.
- "Campaign", "creative brief", "content calendar" → `marketing` / `marketing_campaigns`.
- "Portfolio", "program", "cross-project rollup" → see §3 (use **Project + Portfolio**, NOT regular boards with mirrors).
- "Whiteboard", "diagram", "free-form canvas" → `whiteboard`.
- "Standalone form" (no board) → `forms`.
- Default → `core`.

If unsure which products are enabled, call `get_user_context` first — its `account.products` field lists active product kinds.

---

## 1.5 Native boards and cross-product architecture

### The fundamental rule: always use native boards — modify, don't recreate

monday accounts come with **product workspaces pre-provisioned** at signup. Each workspace contains native boards with purpose-specific `item_terminology`, pre-wired `board_relation` columns, native column ID conventions, and built-in views.

**This is the default: use the existing native board. Add groups, columns, or seed data to it. `create_board` is the last resort — only when a board of that type genuinely doesn't exist in the workspace.**

### STOP: if a required product workspace or its native boards are missing

**The MCP cannot provision a product workspace with native boards.** Only the monday.com UI does this when you enable a product. There are two failure modes — handle both:

#### Case 1: Product not enabled at all (`account.products` doesn't contain the kind)

Stop and tell the user:
> "To build a [Product Name] setup, you need to enable the product on your monday.com account first. Go to **monday.com → your avatar → Administration → Products**, enable **[Product Name]**, and come back. This automatically creates the native workspace with all the pre-built boards (Leads, Contacts, Accounts, etc.). I can't create those native boards via the API — they only exist when the product is enabled through the UI."

#### Case 2: Product is enabled, workspace exists, but workspace is empty or missing native boards

Some products (especially `service`) have a workspace provisioned but no boards inside it yet. Stop and tell the user:
> "Your [Product Name] workspace exists but the native boards haven't been set up yet. Open the **[Workspace Name]** workspace in monday.com and click **'Get started'** or use the template picker to initialise the default boards. Once the native boards exist I can start building on top of them."

#### Per-product: what to tell the user and what to look for

| Product kind | Product name | Workspace to find | Native boards that must exist before building |
|---|---|---|---|
| `crm` | monday CRM | "CRM" | Leads, Contacts, Accounts, Opportunities (or Deals), Sales Activities |
| `software` | monday Dev | "Dev" | Feature request, Product backlog, Customer feedback, Quarterly goals |
| `service` | monday Service | "Service" | Tickets (if empty: tell user to initialise via UI) |
| `marketing_campaigns` | monday Marketer | "Marketing" (or "Marketing Campaigns") | Campaigns, Content Calendar |
| `project_management` | monday Projects | Uses existing `core` workspace | No separate workspace — but requires `create_project` / `create_portfolio` mutations |
| `core` | Work Management | "Main workspace" (or any named workspace) | Project/task boards — these can be created via MCP if absent |

**`core` is the only product where creating boards via MCP without native templates is acceptable** — Work Management boards have no fixed native structure. All other products have native boards that must come from the UI first.

### Required workflow before creating any board

1. Call `get_user_context` — check `account.products` for enabled product kinds.
2. **If the required product kind is absent → Case 1 stop** (tell user to enable via Administration → Products).
3. Call `list_workspaces` — find the workspace for the target product.
4. **If the workspace is missing → Case 1 stop** (same message — product likely not enabled).
5. Call `workspace_info(workspace_id)` — scan all folders and boards.
6. **If the workspace exists but has no native boards → Case 2 stop** (tell user to initialise via the workspace's Get Started flow).
7. **If native boards exist → use them.** Add items/groups/columns. Do not duplicate.
8. **Only `create_board` if a specific board type is genuinely absent from an already-initialised workspace** — and only in the correct product workspace.

---


---

## Reference map - load on demand

This SKILL.md is the decision core: introspect first, pick the right product, use
native boards, refuse anti-patterns, follow the execution order. The detailed
verified facts live in `references/` and should be pulled in **only when the task
needs them** - do not load them all up front.

| When you're working on... | Read |
|---|---|
| Per-product native boards (CRM / Dev / Service / Work Management / Projects / Marketer), the cross-product flywheel, workspaces & folders, object types, Projects & Portfolios | `references/products-and-native-boards.md` |
| Column types, the verified column-VALUE JSON shapes, `board_relation` + `mirror` wiring | `references/columns-and-relationships.md` |
| Items, groups, bulk seeding, forms, docs, updates & notifications | `references/items-forms-docs.md` |
| Dashboards and the 7 widget kinds / 16 chart variants | `references/dashboards-and-widgets.md` |
| Automations, integration blocks, webhooks, audit logs & compliance | `references/automations-and-webhooks.md` |
| Dev sprints, Notetaker, search, users/teams/roles, the Objects platform, knowledge base | `references/products-deep.md` |
| Schema introspection, raw GraphQL fallback, error semantics, and the verified-by-execution gotcha cheat sheet | `references/graphql-errors-gotchas.md` |
| Building monday marketplace apps (SDK / iframe apps) | `references/marketplace-apps.md` |

For an end-to-end demo build driven by a discovery-call transcript, use the
`monday-demo-builder` skill (it follows the 5-phase workflow and defers to this
skill for every API fact). To re-verify any fact here against the live API, run
the `refresh-monday-skill`.

---
## 22. Anti-patterns — STOP if you catch yourself doing these

1. CRM pipeline on a `core` board with a Status column — use `crm`.
2. Creating a new Leads / Contacts / Accounts / Deals / Activities board from scratch when the native CRM workspace already has one — call `workspace_info` on the CRM workspace first, find the existing board, and seed it. Same applies to Dev sprints, Service tickets, and Marketing campaigns.
3. "Portfolio" built from a regular board + manual mirrors — use Projects/Portfolios (`create_project`, `create_portfolio`, `connect_project_to_portfolio`).
4. Related-item ID stored in a `text` column — use `board_relation` + `mirror`.
5. Refusing a widget without checking `all_widgets_schema`. (And don't promise widget types not in the verified catalog: only `NUMBER`, `CHART`, `BATTERY`, `CALENDAR`, `GANTT`, `LISTVIEW`, `APP_FEATURE`.)
6. `change_item_column_values` payload written from memory — call `get_column_type_info` first.
7. Using `column_type: "person"` — deprecated, use `people`.
8. Sprint/epic structure on generic `core` boards — use `software` workspaces + sprint queries.
9. Long-form narrative in a `long_text` column — use a Doc.
10. Hand-rolled GraphQL via `all_monday_api` when a named tool exists.
11. Form-style intake done via manual item creation — use a Form bound to the board.
12. Per-item notification done via update text — use `create_notification`.
13. Files dumped into a `long_text` column — use `file`.
14. Assignee tracked as a name string — use `people` with real user IDs.
15. Email/phone in plain `text` — use `email` / `phone` (enables click-to-email and dialer).
16. Building everything in the workspace root — group with folders.
17. Creating duplicates — `search` first.
18. Monolithic boards with 30+ columns — split by domain and connect via `board_relation`.
19. One dashboard per metric — group related KPIs onto one dashboard.
20. Fetching all items to count them — use `aggregate` or `items_page_by_column_values`.
21. Reading >500 items without pagination — use cursor with `next_items_page`.
22. Promising "automation recipes via API" — recipe creation isn't exposed; use webhooks + `execute_integration_block` instead.
23. Creating reusable status/dropdown labels per board — use **managed columns**.
24. Hardcoding "Done" string for a Battery widget — pass via `done_text` (supports per-language).
25. Ignoring `complexity` errors and retrying with the same query — paginate or trim selection.
26. Calling `change_item_column_values` with a new status/dropdown label without `createLabelsIfMissing: true` — call fails with `ColumnValueException`.
27. Calling `get_full_board_data` directly — it's marked internal-only (UI-triggered). Use `get_board_info` + paginated `get_board_items_page` instead.
28. Promising "portfolio kind" boards via `create_board` — that enum has only `public/private/share`. Portfolios are Projects (`create_project` → `create_portfolio` → `connect_project_to_portfolio`).
29. Replacing whole docs to make small edits — use `create_doc_block` / `update_doc_block` / `delete_doc_block` for granular updates.
30. Creating a webhook via raw GraphQL when `create_webhook` mutation exists.
31. Permanently deleting items/boards/groups when archive is appropriate — prefer `archive_item` / `archive_board` / `archive_group` (reversible) over `delete_*`.
32. Creating a status/dropdown column on a board you intend to share label semantics across — use `attach_status_managed_column` / `attach_dropdown_managed_column` (linked to a managed column) instead of `create_status_column` / `create_dropdown_column` (board-local).
33. Confusing `backfill_items` with `ingest_items` — backfill is for one-time data migration (no side-effects, 20k rows); ingest is for ongoing integrations (full side-effects, 10k rows).
34. Citing error codes from memory in user-facing diagnostics — read the actual `error_code` from the API response.
35. **Creating a `board_relation` column with empty boardIds and asking the user to wire it in the UI** — wrong. Use `create_column.defaults: "{\"boardIds\":[<id>]}"` (raw GraphQL via `all_monday_api`) to wire it at creation. The "needs UI handoff" rule was a patch7 mistake. See §5.
36. **Promising `create_sprint` to provision the engineering sprint board set** — that mutation is not in the schema. The user must add the Sprint template via the monday UI first. See §1.5 / §13.
37. **Quoting the response of `update_board` with a `{ id }` selection** — the response is bare JSON, not a `Board` object. See §6.
38. **Creating a mirror column via the `create_column` MCP tool** — it always fails with a schema validation error. Use `all_monday_api` with raw GraphQL and the `defaults` string argument. See §5.
39. **Treating `create_dashboard` as producing a usable dashboard** — it returns an empty container with zero widgets. Always follow immediately with `create_widget` calls for every widget. An empty dashboard in a demo is a failure. See §7.
40. **Creating a `board_relation` column without `boardIds` and assuming it works** — a relation column with empty `boardIds` is non-functional: items cannot be linked via API, and the UI shows a broken picker. Always pass `boardIds` via `defaults` at creation time using `all_monday_api`. If a column already exists with empty `boardIds`, delete and recreate — you cannot patch it after the fact. See §5.
41. **Passing `status` or `date` column values inside `create_item.columnValues`** — they are silently ignored. Always use two steps: `create_item` (name + group only) → `change_item_column_values` (all column values). See §25 Round-4.
42. **Including `id` in label objects passed to `update_status_column`** — causes `ColumnValueException`. Remove `id` from every label definition; only `name` and `color` are accepted.
43. **Assuming `allowCreateReflectionColumn: true` backfills reverse cells** — it only creates the reverse column structure. Reverse cells start empty and must be populated manually via `change_item_column_values` on the linked board. See §25 Round-4.
44. **Retrying a failed compound mutation with the same payload** — partial writes may have already succeeded. Re-query board state first and only retry the columns that are still empty. See §25 Round-4.
45. **Writing `board_relation` values via `change_item_column_values` on a `permissions: owners` board** — writes silently don't persist. Use raw GraphQL `change_multiple_column_values` via `all_monday_api` instead. See §25 Round-4.
46. **Using `{"label": "X"}` shape for a `dropdown` column** — that is the `status` shape. Dropdown uses `{"labels": ["X"]}` (plural key, array value). Silent failure if wrong shape used. See §4.

---

## 23. Output contract for the planning phase

Before executing, present a plan that explicitly states:

1. **Product `kind`** (`core` / `crm` / `software` / `service` / `marketing(_campaigns)` / `project_management` / `forms` / `whiteboard`) and why.
2. **Workspaces & folders** to be created/used.
3. **Boards** — name, `BoardKind` (`public`/`private`/`share`), product context, owner, sub-items?
4. **Projects/Portfolios** if cross-project rollup is needed (instead of regular boards + mirrors).
5. **Columns** per board — name, `ColumnType` (use the enum string), purpose. Flag every `board_relation` / `mirror` / `formula` / `dependency` / managed column.
6. **Cross-board relationships** — list of `board_relation` links + which `mirror` columns surface which fields.
7. **Items / groups** to seed (if any).
8. **Forms** with question → column mapping.
9. **Dashboards** — which widget types from the verified catalog (§7), config sourced from `all_widgets_schema`, attached to which boards.
10. **Views** (`create_view`) if Kanban/Table/Calendar/etc. are needed at the board level.
11. **Docs** to create.
12. **Webhooks / integrations / automations** — which `WebhookEventType` events and target endpoints.
13. **Permissions / sharing** assumptions.
14. **Open MCP lookups** still needed before building (e.g., "need `get_user_context` to confirm `crm` is enabled", "need `get_column_type_info(status)` for label config").

If the user pushes back on any choice, re-evaluate openly. Don't silently downgrade to a regular board to avoid the conversation.

---

## 24. Execution order

When the plan is approved, execute in this order to avoid forward-references:

1. Workspaces → folders.
2. Managed columns (status/dropdown), if any are reusable across boards.
3. Boards (parents before children of `board_relation` relationships).
4. Columns per board (including `board_relation`). **For NEW `board_relation` columns, the `boardIds` setting is empty after creation — see step 7.**
5. `mirror` columns (after `board_relation` exists on both sides).
6. Groups.
7. **For every `board_relation` column you create, use raw GraphQL `create_column` with `defaults: "{\"boardIds\":[<target>]}"`** — this wires the column at creation. Native columns (`contact_account`, `task_sprint`, `bug_tasks`, etc.) ship pre-wired and don't need this. See §5 for the exact mutation shape and the workaround when a mandatory empty column already exists.
8. Seed items + initial column values. Use `createLabelsIfMissing: true` if seeding new status/dropdown labels. `board_relation` writes (`{"item_ids": [...]}`) work — once `boardIds` is set per step 7.
9. Views (`create_view`).
10. Forms (board must exist).
11. Docs.
12. Dashboards + widgets (boards must exist with data for widgets to be meaningful).
13. Webhooks / integrations / automation triggers.
14. Notifications / sharing.
15. Verify with `get_board_info` + paginated `get_board_items_page` / `board_insights` / `aggregate` before reporting done. (Do NOT use `get_full_board_data` — internal-only.)

**Moving boards between folders/workspaces:** use `update_board_hierarchy(board_id: ID!, attributes: { workspace_id?: ID, folder_id?: ID })`. Returns `UpdateBoardHierarchyResult { success, message, board }` — no `id` field. Verified working — this is the right tool for re-organizing demo workspaces (e.g. moving a Campaigns board from a separate Marketing workspace into a CRM workspace folder mid-build). `move_object` works for boards too but `update_board_hierarchy` is more explicit.

If a step fails, fix the root cause — don't paper over with a `text`-column workaround.

---


## 27. Quick reference — the verified facts you must not invent

- **Product kinds** (8): `core`, `crm`, `software`, `service`, `marketing`, `project_management`, `forms`, `whiteboard`. (Note: `account.products` may *return* `marketing_campaigns` as a product identifier, but the `WorkspacesQueryAccountProductKind` enum only accepts `marketing` — see §0.)
- **BoardKind** (3): `public`, `private`, `share`. (No "portfolio" kind — portfolios are Projects.)
- **WorkspaceKind** (3): `open`, `closed`, `template`.
- **DashboardKind** (2): `PUBLIC`, `PRIVATE`.
- **ColumnType** (40+): see §4.
- **WebhookEventType** (21): see §11.
- **Widget types** in `all_widgets_schema`: `NUMBER`, `CHART`, `BATTERY`, `CALENDAR`, `GANTT`, `LISTVIEW`, `APP_FEATURE`. Anything else is unsupported via `create_widget`.

If you find yourself about to state a fact in any of these categories that doesn't match the above, re-check via `get_type_details` — the schema may have evolved.
