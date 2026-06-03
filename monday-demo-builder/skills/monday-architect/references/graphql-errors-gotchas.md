# Schema introspection, raw GraphQL, errors & verified gotchas

> Verified reference for the `monday-architect` skill (progressive disclosure). Loaded on demand; the decision core lives in that skill's SKILL.md.

## 19. Schema introspection (deeper)

- `get_object_schemas` — account-level object schemas (board structure templates).
- `get_column_type_schema` — column-type JSON Schema for `update_column` defaults.
- `get_view_schema_by_type` — JSON Schema for `create_view` settings.
- `complexity` query — current GraphQL complexity budget. Monitor this for bulk operations.
- `version` / `versions` — API version info; pin if needed via `API-Version` header (raw GraphQL).
- `platform_api { daily_limit, daily_analytics }` — API quota and usage (verified fields). The `DailyAnalytics` type aggregates by app/day/user under sub-fields.

---

## 20. Raw GraphQL fallback (`all_monday_api`)

Use only when:
- A purpose-built MCP tool doesn't exist.
- You need a multi-operation document (batch write).
- You need fields/args not exposed by a named tool.
- You need to subscribe to webhooks, run integration blocks, or hit advanced queries (`audit_logs`, `aggregate`, `objects`, `notetaker`, `connections`, `sprints`, `complexity`, etc.).

Best practices:
- Validate against `get_graphql_schema(operationType)` and `get_type_details(typeName)` before sending.
- Watch `complexity` — bulk reads can exceed the budget; paginate via cursors (`next_items_page`).
- Pass `API-Version` header if you need a specific version (default = **Current** stable, not the latest release-candidate). Version string is `YYYY-MM`. Three rings run in parallel: Release Candidate (preview) / Current (default) / Maintenance; each is stable ≥6 months and deprecations are announced ≥6 months ahead. A request for an unknown version silently falls back to Maintenance.
- **Endpoint & auth (when constructing raw HTTP, e.g. outside the MCP):** `POST https://api.monday.com/v2`, `Content-Type: application/json`. Token goes in the `Authorization` header — **personal API tokens have NO `Bearer` prefix; OAuth access tokens DO require `Bearer `**. (Mixing these up is a frequent 401 source.) **File uploads use a different endpoint** — `POST https://api.monday.com/v2/file` with `multipart/form-data` (`query` + `map` + the file part), the `File!` GraphQL scalar, max 500 MB. The MCP `get_asset_upload_url → PUT → finalize_asset_upload` flow is the wrapper alternative; both exist.
- Returned errors include `error_code`, `error_message`, and `extensions` data. Common code seen on column-value mismatches: `ColumnValueException` — fix by re-fetching the type schema (`get_column_type_info`) and rebuilding the payload, not retrying blindly. Other codes (complexity-budget, permission, validation) come back with their own `error_code` strings — read what the API returns instead of guessing.

### 20.1 Raw `items_page` filtering (when the `get_board_items_page` tool isn't enough)

The MCP tool wraps this, but for raw `all_monday_api` reads you need the `ItemsQuery` shape (verified against the live schema):

- **`items_page` is a field on `Board` only — NOT a root query** (`boards(ids:…){ items_page(...) { cursor items { id name } } }`). Root-level paging fields are `next_items_page`, `items_page_by_column_values`, and `items`.
- **`limit` default = 25, max = 500.** (The `500` is the cap, not the default — don't assume 500.)
- `items_page(query_params: ItemsQuery)` where `ItemsQuery = { rules, groups, ids, operator, order_by }`.
  - `rules: [{ column_id: ID!, compare_value: CompareValue!, operator: ItemsQueryRuleOperator, compare_attribute: String }]`
  - **`operator` between rules is lowercase `and` / `or`** (default `and`) — NOT `AND`/`OR`.
  - **Mixed AND/OR requires nested `groups`**, each group having its own `rules` + `operator`; a flat `rules` array shares one operator only.
  - `ids: [ID!]` inline fetches a specific item set — **max 100 IDs**.
  - `order_by: [{ column_id: String!, direction: ItemsOrderByDirection }]` — a LIST (multi-sort); **direction is lowercase `asc` / `desc`**.
- **Full `ItemsQueryRuleOperator` enum (16, live-verified):** `any_of`, `not_any_of`, `is_empty`, `is_not_empty`, `greater_than`, `greater_than_or_equals`, `lower_than`, `lower_than_or_equal`, `between`, `contains_text`, `not_contains_text`, `contains_terms`, `starts_with`, `ends_with`, `within_the_next`, `within_the_last`. **Note the asymmetry: `greater_than_or_equals` (plural) vs `lower_than_or_equal` (singular)** — a real API inconsistency, easy to typo.
- **`compare_value` encoding is per-column-type and opaque** (status → label index integers, people → `"person-<id>"` strings, date → keywords like `"TODAY"` or `["EXACT","2026-01-01"]`). Don't guess — confirm per column via `get_column_type_info` (`fetchMode: "guidelines"`).
- **`next_items_page(cursor, limit)` carries the original query context** — you set filters/order once on the first `items_page` call and cannot change them mid-pagination. Cursor lifetime is ~60 min (doc-sourced).

---

## 21. Error semantics & rate limits

- Mutations on column values that fail validation throw `ColumnValueException` with details — call `get_column_type_info` and rebuild the payload, don't retry blindly.
- Item-creation cap is per-board; over-large boards should be split or migrated to portfolio/project structure.
- Complexity budget is per-minute; for big read sets, paginate with `next_items_page` (cursor-based) and avoid wide nested selections.
- For very large item sets, prefer `aggregate` over fetching items.
- **Rate limiting is three independent dimensions, not just complexity** (the MCP wrapper hides this, but raw `all_monday_api` bulk loops can trip any of them):
  - **Complexity budget** — per-user personal token ~10M points/min (1M for trial/free); app tokens get separate ~5M read + ~5M write budgets. Single query hard ceiling ~5M. Sliding 60-second window. Read the `complexity { before after query reset_in_x_seconds }` field to measure.
  - **Request rate** — per-minute call limit by tier (roughly Enterprise 5,000 / Pro 2,500 / other 1,000 per min) plus daily caps. Over-limit → **HTTP 429** with a `Retry-After` header. Honour it; don't hammer-retry.
  - **Concurrency** — max simultaneous in-flight requests by tier (~Enterprise 250 / Pro 100 / other 40). Over → 429 `maxConcurrencyExceeded`. Keep batch loops sequential-ish (10–25 mutations per document, per §26).
  - Application errors return **HTTP 200** with an `errors` array; only rate/transport errors use 4xx/5xx. Responses carry a `request_id` in `extensions` — quote it in support tickets. *(Tier numbers are doc-sourced and approximate — verify against the live `rate-limits` page for the account's plan.)*

---

## 25. Verified-by-execution gotchas (cheat sheet)

These are the things that bit during a real end-to-end build. Read this before you write payloads.

### Naming
- **Column ID prefixes don't match the `columnType` enum.** `numbers`→`numeric_*`, `creation_log`→`pulse_log_*`, `mirror`→`lookup_*`, `status`→`color_*`, `people`→`multiple_person_*`, `timeline`→`timerange_*`, `time_tracking`→`duration_*`, `tags`→`tag_*`, `checkbox`→`boolean_*`. Match the prefix when targeting a column by ID.
- **Managed-column IDs are UUIDs**, not the usual `prefix_xxx` shorthand.
- **Subitems live on a hidden auto-generated board** named `Subitems of <BoardName>`; the response includes its `board_id`.

### Settings vs values
- **`columnSettings` for `create_column` is UNWRAPPED.** Send `{"labels": [...]}`, NOT `{"settings": {"labels": [...]}}`. The schema returned by `get_column_type_info` *describes* the `settings` field — that wrapper is conceptual, not literal payload.
- **Column-value JSON ≠ column-settings JSON.** Column values use shapes like `{"label": "..."}`, `{"date": "..."}`, `{"item_ids": [...]}`. See §4 step 4 for the verified table.
- **Status `index` is remapped to color ID.** When you create a status with `index: 0` and `color: "american_gray"`, monday stores the label keyed by `17` (the color ID). Don't trust your assigned indexes for index-based writes — use label-based writes (`{"label": "X"}`) to avoid the trap.
- **Phone numbers can't have spaces** in the `phone` field. `+4940123456` works; `+49 40 123 4567` fails (the API tries to parse the second token as `countryShortName`).
- **Status colors in raw GraphQL are enum literals (unquoted)**, e.g. `color: done_green`. Inside `create_column.columnSettings` (a JSON string), use `"done_green"`. The MCP wrapper converts JSON → enum at the boundary.

### Architecture
- **`create_board` does NOT take a folder ID.** Boards land at workspace root. Call `move_object(objectType: "Board", id, parentFolderId)` after creation.
- **`create_form` auto-creates its backing board.** Don't create the response board first — the form mutation returns the board it just made plus a `form_token`.
- **`create_doc` accepts `markdown` directly.** No need to chain `create_doc_blocks` for the initial body.
- **`create_doc_blocks` shape:** `{text_block: {delta_format: [{insert: {text: "..."}}]}}`. Args are `docId` + `blocksInput`. Max 25 blocks per call.
- **`create_project` is async.** Returns `process_id`; the actual `project_id` arrives via callback URL or by polling. Plan for the latency.
- **`create_portfolio` doesn't return a board ID.** Query `boards(workspace_ids: [...])` after to find the new portfolio board.
- **`delete_workspace` cascades** — *when it runs.* When the API actually executes the mutation, it removes all boards, items, dashboards, docs, forms, projects, portfolios, subitem-boards, columns, groups, and updates in one call. Useful for demo cleanup. **But** the MCP harness's Stage 2 safety classifier frequently blocks this mutation; have the user delete the workspace via the monday UI as the reliable fallback.

### MCP tool quirks
- **`get_full_board_data` is internal-only.** Even though it appears in the tool list, its description says it's UI-triggered only. Use `get_board_info` + paginated `get_board_items_page` instead.
- **`link_board_items_workflow` is referenced as a precondition but is NOT a callable tool.** Writing to `board_relation` columns works fine without it (assuming `boardIds` is set per §5). Treat the precondition note in `change_item_column_values` and `get_board_items_page` as legacy documentation.
- **`create_subitem` is NOT a top-level MCP tool.** Use `create_item` with `parentItemId`. (The raw GraphQL `create_subitem` mutation does exist.)
- **No `delete_workspace` MCP tool either** — and the raw GraphQL mutation is **frequently blocked by the MCP harness's safety classifier** ("Stage 2 classifier — permission denied"). When you need to delete a workspace and the harness blocks it, tell the user to delete it via the UI (sidebar → right-click → Delete workspace) rather than retrying.
- **Webhook creation may be sandbox-blocked.** In some environments, creating webhooks pointed at external URLs (even `example.com`) is blocked by tooling-level guardrails. The API itself accepts the call; the harness denies it. Tell the user the URL/event combination you'd create rather than failing silently.
- **`board_relation.boardIds` IS settable via `create_column.defaults: "{\"boardIds\":[<id>]}"`.** See §5. `update_column(settings)` and `change_column_metadata` cannot set boardIds AFTER creation, but `create_column.defaults` works at creation time. Use raw GraphQL via `all_monday_api` since the MCP `create_column` tool wrapper exposes `columnSettings` (column-type config, not the same arg). When a mandatory column already exists with empty boardIds, add a supplementary `board_relation` column instead.

### Errors
- **`ColumnValueException`** is the standard error for bad column values. The error response includes `extensions.error_data` with `column_validation_error_code` (e.g. `missingLabel` for an unrecognized status label). `createLabelsIfMissing: true` is the fix for the `missingLabel` case.
- **The underlying mutation for `change_item_column_values` is `change_multiple_column_values`** — useful when reading error stack traces or constructing raw GraphQL.
- **Status label errors helpfully list valid options:** `"This status label doesn't exist, possible statuses are: {0: Proposal, 1: Closed Won, ...}"` — read those keys (color IDs) for the canonical mapping.

### API meta
- **Complexity budget:** ~5,000,000 per minute (verified `before/after/query/reset_in_x_seconds` shape). Reads cost roughly nothing per query. Bulk reads are the only realistic complexity risk.
- **API version:** Monday is on rolling release with date-versioned API (e.g. `release_candidate 2026-07`). The default (no `API-Version` header) is the latest stable.
- **Audit log event names use kebab-case** (`create-workspace`, `delete-board`, `add-user-to-team`), not snake_case. 75+ event types are catalogued.

### Round-2 findings (portfolio + advanced)
- **`create_project` is NOT just one board.** It creates two boards (parent + tasks) AND seeds the tasks board with **2 demo tasks (Task 1, Task 2)** and a **14-column native template** (`project_owner`, `project_resource`, `project_status`, `project_priority`, `project_timeline`, `project_dependency`, `project_planned_effort`, `project_effort_spent`, `project_duration`, `project_budget`, `project_task_completion_date`, `subtasks_*`, plus a back-link `board_relation`). Plan for these — they affect demo seeding strategy.
- **`connect_project_to_portfolio` only accepts the project's TASKS board ID** (the lower-numbered of the two), not the parent. The error if you pass the wrong one is: `"Failed to connect project to portfolio. the following boards are not projects: [<id>]"`.
- **`solution_live_version_id` returned from `create_portfolio` is template-version-shared**, NOT the unique portfolio ID. Same value is returned for every portfolio. To find the new portfolio's actual board ID, query `boards(workspace_ids: [<id>])` after creation.
- **A connected portfolio has 11 specific native columns.** See §3.5 for the full list. Two of those columns (`portfolio_project_progress`, `portfolio_project_actual_timeline`) are mirrors that auto-rollup `project_status`/`project_timeline` from connected tasks boards.
- **`connect_project_to_portfolio` wires structure but not item-level links.** The `portfolio_project_link` value on each portfolio item starts null even after a successful connect — you may need to populate it manually. Validate the mirror columns visually if you depend on them.
- **All CHART `graph_type` variants verified working** end-to-end via `create_widget` — the live enum lists 24 entries, which are ≈12 unique chart types each accepted in both snake_case and camelCase. See §7.
- **`ViewKind` enum is just 4 values:** `DASHBOARD`, `TABLE`, `FORM`, `APP`. **There is NO `KANBAN` / `CALENDAR` / `GANTT` view creatable via `create_view`** — those are dashboard widget types only. For board-level kanban/calendar/gantt views, the user must add them manually in the UI.
- **`get_view_schema_by_type` arg names:** `type: ViewKind!, mutationType: ViewMutationKind!` (not `view_type`).
- **`convert_board_to_project.column_mappings` is REQUIRED.** Three required ID fields: `project_status`, `project_timeline`, `project_owner`. Cannot pass empty `{}`. The values must be column IDs on the source board (or `"name"` for the name column as a stand-in for owner).
- **`archive_group` cascades to its items** — items in the archived group become inaccessible via `items()` query and reject `change_simple_column_value` with `"Cannot change column value for inactive items"`. **The mutation's response field `archived: false` is unreliable** — verify the side effect by querying.
- **`change_simple_column_value` for status accepts: label name (`"C"`), color-key-as-string (`"1"`), or numeric color ID.** The status label is keyed by COLOR ID, not your `index` field at creation time.
- **`add_users_to_board` arg `kind` is an enum**: `subscriber` (lowercase). Returns `[{id}]` array.
- **`pin_to_top` uses `id`, not `item_id`** (the `id` is the update ID).
- **`update_mute_board_settings` with `MUTE_ALL` is owner-only** — non-owners get `"User unauthorized to perform action"`. The enum has 5 values: `NOT_MUTED`, `MUTE_ALL` (owner-only), `MENTIONS_AND_ASSIGNS_ONLY`, `CUSTOM_SETTINGS`, `CURRENT_USER_MUTE_ALL`.
- **`create_validation_rule` may return "This feature is not currently supported"** — feature-flagged. Don't promise validations on every account.
- **`backfill_items` ✅ works.** Args: `board_id`, `group_id` (no `on_match` here despite the schema). Returns `UploadJobInit { job_id, upload_url }`. The `upload_url` is a pre-signed S3 PUT URL valid 10 minutes (`X-Amz-Expires=600`). After uploading a CSV, poll via `fetch_job_status(job_id)`.
- **`fetch_job_status` returns `ItemsJobStatus`** with: `status` (BulkImportState enum), `progress_percentage` (0–100), `fully_imported` (bool), `failure_reason`, `failure_message`, `report_url`. Initial status is `UPLOAD_PENDING`.
- **`validations(id: ID!)`** — the arg is named `id`, not `board_id`, but takes the board ID.
- **`pin_to_top`, `like_update`, `unlike_update` etc. all use `id` arg** — not entity-specific names.
- **Update mentions render as embedded HTML `<a class="user_mention_editor router">` tags** in the body when read back. Pass them via `mentionsList: '[{"id":"...","type":"User"}]'`.

### Round-3 findings (cross-product demo build, May 2026)

- **`board_relation.boardIds` IS settable at creation via `create_column.defaults`.** Patch7 incorrectly stated this was impossible — the `defaults: JSON` argument on the `create_column` mutation accepts `{"boardIds":[...]}` and wires the column at creation. Verified end-to-end. Only `update_column.settings` is broken (still rejects `boardIds`). See §5 for the exact mutation shape, mandatory-column workaround, and reverse-side reflection guidance.
- **`create_sprint` is not in the schema.** Native engineering sprint board pair (Sprints/Tasks/Epics/Bugs Queue/Retrospectives/Capacity) must be provisioned via UI sprint template. See §1.5 / §13.
- **`update_board` returns bare JSON.** Args: `board_id, board_attribute, new_value`. NO `{ id }` selection — fails with `must not have a selection since type "JSON" has no subfields`. Multiple updates in one document need GraphQL aliases.
- **`update_board_hierarchy` is the right tool for moving boards between workspaces or folders.** Args: `board_id, attributes: { workspace_id?, folder_id? }`. Response type is `UpdateBoardHierarchyResult { success, message, board }` — NO `id` or `errors` fields. Use this when restructuring demo workspaces mid-build.
- **`change_column_metadata.column_property` enum has only 2 values:** `title`, `description`. Don't try to use this to set column settings.
- **`update_column(settings: JSON)` exists but rejects every realistic payload.** The schema lists the arg; the validator rejects `boardIds`, `allowedValues`, etc. with `Column schema validation failed`. Effectively read-only for type-specific settings post-creation.
- **`delete_workspace` is harness-blocked** (Stage 2 classifier). Hand off to UI deletion.

### Round-4 findings (May 2026 build)

- **`create_item` silently ignores `status` and `date` column values.** Passing `columnValues` with `{"status_column_id": {"label": "..."}, "date_column_id": {"date": "..."}}` on `create_item` creates the item with a name only — the column values are silently dropped. **Two-step pattern is mandatory, not optional:** `create_item` (name + group only) → `change_item_column_values` (all column values). This applies to both the MCP `create_item` tool and the raw GraphQL mutation via `all_monday_api`. Verified across multiple boards in May 2026.
- **`update_status_column` must NOT include `id` in label objects.** The correct label shape is `{label: {name: "X", color: done_green}}` — the `id` field is read-only and rejected. Including `id` returns `"ColumnValueException: label id cannot be set manually"`. Remove `id` from every label definition when calling `update_status_column`.
- **`allowCreateReflectionColumn: true` creates the reverse column structure but does NOT backfill reverse cells.** When you create a `board_relation` column with `allowCreateReflectionColumn: true`, the reverse `board_relation` column appears on the linked board, but all cells are empty. You must backfill the reverse cells manually: for each item on the linked board that should show a relation, call `change_item_column_values` on the reverse column with `{"item_ids": [...]}`. Or write relations on one side only and rely on the reverse to reflect automatically when both columns have `boardIds` configured.
- **Failed compound mutations may partially apply — always re-query after failure.** When a multi-column `change_item_column_values` call fails (e.g., due to one bad label), some column values may already have been written before the error. Do NOT retry the full payload blindly — re-fetch board state first and only update the columns that failed.
- **`permissions: owners` boards silently don't persist `board_relation` writes via `change_item_column_values`.** The mutation returns success but the cell stays empty. Fix: use raw GraphQL `change_multiple_column_values` via `all_monday_api` instead. If you encounter a board that consistently ignores board_relation writes even with correct `boardIds`, check the board's `permissions` setting and switch to raw GraphQL.
- **`title5` dropdown on native CRM Contacts only has 3 labels: `CEO`, `COO`, `CIO`.** The API rejects any other label with `"label does not exist, possible labels are: {1: CEO, 2: COO, 3: CIO}"`. When assigning job titles to CRM contacts, pick the closest of the three available values. Do NOT try to add new labels via `update_dropdown_column` — native column labels may be locked.
- **`move_item_to_group` does NOT need `boardId`.** Passing `boardId` in variables causes `"Variable $boardId is never used in operation"`. Args: `itemId` + `groupId` only. Verified working without `boardId`.
- **Dropdown column value shape is `{"labels": ["X"]}` (plural, array), NOT `{"label": "X"}`.** That is the status shape. Using `{"label": "X"}` on a `dropdown` column silently fails — cell stays empty. Always use `{"labels": [...]}` for dropdown.
- **Mandatory `board_relation` columns on native service boards cannot be hidden via API.** The native ITSM Tickets board has a mandatory `board_relation6` column. If your demo migrates the linked board (e.g., replaces the original Customers board with CRM Accounts), the mandatory column becomes a permanently visible empty artifact. The only fix is UI-only: hide it via the column chooser in each board view. Add this to your pre-demo checklist.
- **`mirror` columns return `"Column value type is not supported"` when read via `get_board_items_page`** — even though they render correctly in the UI. Don't panic when a mirror reads as that string; verify visually in the UI instead.
- **CRM native `board_relation` reads return `null` via API even when set.** `deal_contact`, `account_contact`, etc. populate correctly in the UI but `get_board_items_page` returns null arrays for those columns on some accounts. The data IS there — UI is the source of truth for these specific native columns.
- **"Form auto-creates a Name question."** When you call `create_form`, the backing board comes pre-seeded with a `Name` question (id: `name`, type: `Name`). Trying to `create` another Name question fails with `ExceededUniqueQuestionTypeCount`. To customize, use `form_questions_editor(action: "update", questionId: "name", ...)` instead.
- **Widget API has NO read/list endpoint, but `delete_widget(id: ID!): Boolean` works.** `create_widget` and `delete_widget` are both available as mutations (verified end-to-end). What's missing is any way to *list* widgets on a dashboard — `Dashboard` type has fields `id`, `name`, `workspace_id`, `kind`, `board_folder_id` and nothing else; no `widgets` query exists on `Query`. So you can only delete a widget if you saved its ID from the original `create_widget` response. Widgets pointing at deleted boards stay broken until you delete them by ID (or remove them via the UI).

### Data shapes seen on the wire
- `boards.columns[].settings_str` is a JSON-encoded string — parse to inspect a column's actual stored config.
- `notetaker.meetings` returns a doubly-nested `{meetings: {meetings: [...], page_info: {cursor}}}`.
- `aggregate` query takes `query: AggregateQueryInput!` with fields `select` and `from` (singular). Don't use `selects`/`from_table`. (`board_insights` is the easier MCP-wrapped alternative.)
- `search` namespace returns `{results: [{id, indexed_data, live_data}]}` for items/boards/docs. `live_data` may be null for stale or deleted records.
- `object_types_unique_keys` returns `{object_type_unique_key, app_name, app_feature_name, description}`. Native object keys: `monday_documents::doc`, `monday_workflows::workflow`, `service::portal-object`, `solutionsv2_monday-dashboards::dashboard_object`, `solutionsv2_new-form::form`. Many third-party app keys appear too.

---

