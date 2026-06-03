# Column model & cross-board relationships

> Verified reference for the `monday-architect` skill (progressive disclosure). Loaded on demand; the decision core lives in that skill's SKILL.md.

## 4. Column model — the typed schema layer

Verified `ColumnType` enum (use these strings, not UI labels):

`auto_number`, `board_relation`, `button`, `checkbox`, `color_picker`, `country`, `creation_log`, `date`, `dependency`, `doc`, `direct_doc`, `dropdown`, `email`, `file`, `formula`, `hour`, `integration`, `item_assignees`, `item_id`, `last_updated`, `link`, `location`, `long_text`, `mirror`, `numbers`, `people`, `phone`, `progress`, `rating`, `status`, `subtasks`, `tags`, `team`, `text`, `timeline`, `time_tracking`, `vote`, `week`, `world_clock`. Plus `name`, `group`, `unsupported` (system-only). `person` is **deprecated** — use `people`.

Pick by intent:

| Need | Column |
|---|---|
| Workflow stages with fixed labels & colors | `status` |
| Multi-select tags from fixed list | `dropdown` |
| Free text / long text | `text` / `long_text` |
| Date / date+time | `date` |
| Time range | `timeline` |
| Calendar week | `week` |
| Time of day | `hour` |
| Assignee (person/team) | `people` (NEVER `person` — deprecated) |
| Whole-team assignment | `team` |
| Currency / count / score | `numbers` |
| File attachments | `file` |
| Hyperlink | `link` |
| Email / phone (with click-to-action) | `email` / `phone` |
| Cross-board relationship | `board_relation` (NOT `text`) |
| Surfaced value from related item | `mirror` |
| Computed value | `formula` |
| Auto-incrementing ID | `auto_number` / `item_id` |
| Created by + when | `creation_log` |
| Last updater + when | `last_updated` |
| Vote / rating / progress battery | `vote` / `rating` / `progress` |
| Geolocation | `location` |
| World clock | `world_clock` |
| Country | `country` |
| Item dependencies | `dependency` |
| Sub-tasks | `subtasks` |
| Doc embed | `doc` / `direct_doc` |
| Action button on the row | `button` |
| Time tracking widget | `time_tracking` |
| Cross-board tag | `tags` |
| Color swatch (design system) | `color_picker` |
| Item-level checkbox flag | `checkbox` |
| Auto-aggregate of all assignees | `item_assignees` |

Workflow:
1. `create_column` to add a column. For `status` and `dropdown`, prefer **managed columns** (`create_status_managed_column` / `create_dropdown_managed_column`) when the same labels should be reusable across boards. Managed-column IDs are UUIDs (e.g. `debd05ab-5c5b-4a6e-9760-35b07a9b4dd1`), not the usual `prefix_xxx`.
2. **Column ID prefix doesn't match the type name (verified):** column IDs are auto-assigned by the API and DO NOT use the `columnType` enum string. Common mismatches: `numbers` → `numeric_*`, `creation_log` → `pulse_log_*`, `mirror` → `lookup_*`, `status` → `color_*`, `people` → `multiple_person_*`, `timeline` → `timerange_*`, `time_tracking` → `duration_*`, `tags` → `tag_*`, `checkbox` → `boolean_*`. Match these patterns when constructing columnValues payloads.
3. **`columnSettings` for `create_column` must be UNWRAPPED** — pass the inner contents only. `get_column_type_info` returns a schema like `{schema: {properties: {settings: {properties: {labels: ...}}}}}` — the `settings` wrapper there is descriptive, NOT literal. Send `{"labels": [...]}` directly, NOT `{"settings": {"labels": [...]}}`. (Verified — sending the wrapped form throws `must NOT have additional properties`.)
4. To set a value on an item via the MCP `change_item_column_values` tool (the wrapper), use these verified value shapes — column-value JSON is *NOT* the same as column-settings JSON:
   - `text` / `long_text`: plain string
   - `numbers`: bare number (not wrapped)
   - `status`: `{"label": "..."}`
   - `dropdown`: `{"labels": ["..."]}` (note: plural `labels`, array)
   - `date`: `{"date": "YYYY-MM-DD", "time"?: "HH:MM:SS"}`
   - `timeline`: `{"from": "YYYY-MM-DD", "to": "YYYY-MM-DD"}`
   - `email`: `{"email": "...", "text": "..."}`
   - `phone`: `{"phone": "+digitsonly", "countryShortName": "DE"}` (no spaces in the phone string)
   - `link`: `{"url": "...", "text": "..."}`
   - `country`: `{"countryCode": "DE", "countryName": "Germany"}`
   - `rating`: `{"rating": 1..5}`
   - `checkbox`: `{"checked": "true" | "false"}` (string-typed booleans)
   - `people`: `{"personsAndTeams": [{"id": <int>, "kind": "person"|"team"}]}`
   - `board_relation`: `{"item_ids": [<int>, ...]}`
   - `time_tracking`: `{"running": <bool>, "startDate": <unix_ts>, "duration": <seconds>}`
   - `tags`: `{"tag_ids": [<int>, ...]}`
   When in doubt for an unfamiliar column type, call `get_column_type_info` for the schema, then sanity-check by reading an existing item's value via `get_board_items_page` with `includeColumns: true`. **But never echo a read value straight back as a write value** — read payloads carry extra/different fields. The sharpest case: `board_relation`/`connect` *reads* return `linkedPulseIds` (or `linkedPulseId`), yet you *write* `{"item_ids":[...]}`. Read shapes also include `changed_at` etc. that writes must omit. Read-only types (`formula`, `mirror`, `auto_number`, `item_id`, `creation_log`, `last_updated`, `progress`) cannot be written at all — `formula` is readable only via `display_value` (2025-01+), and `files` are set via `add_file_to_column` (multipart), not a JSON value.
5. **`createLabelsIfMissing: true`** on `change_item_column_values` allows writing a status/dropdown label that doesn't exist yet — without it, you get `extensions.code = "ColumnValueException"` with `column_validation_error_code = "missingLabel"`. (Verified.)
6. **Status `index` field gotcha (raw GraphQL):** when creating a status column with explicit indexes, the API stores labels keyed by the **color ID** (the `StatusColumnColors` numeric value), NOT the `index` field you provide. So a label you created with `index: 0` and `color: "american_gray"` is stored at key `17` (the ID of `american_gray`). When using `change_simple_column_value` with an integer index, you must use the **color ID**, not your assigned index. The `change_item_column_values` MCP tool with `{"label": "..."}` avoids this entirely — prefer label-based writes.
7. **In raw GraphQL, status colors are an enum, not a string** — write `color: done_green` (unquoted) inside a mutation, NOT `color: "done_green"`. The MCP `create_column` wrapper accepts strings via `columnSettings` because it serializes JSON, but `all_monday_api` calls into mutations like `create_status_managed_column` need raw enum syntax.
8. To update column-level metadata (e.g., status labels, dropdown options), use `update_status_column`, `update_dropdown_managed_column`, etc.
9. **Managed column lifecycle:** `activate_managed_column(id)` / `deactivate_managed_column(id)` toggles a managed column between active and inactive state across all boards using it. `delete_managed_column(id)` permanently removes it. `update_status_managed_column` / `update_dropdown_managed_column` update labels/options on managed columns.

---

## 5. Cross-board relationships (`board_relation` + `mirror`)

The single biggest design pitfall. Rules:

1. **`board_relation` columns** link items across boards. Never store a related-item ID in a `text` column.
2. **`mirror` columns** surface fields from the linked item. Mirrors are the join — without them you cannot report across boards.
3. **Bidirectional**: add a `board_relation` on each side if both directions need to navigate or report.
4. Mirror columns are **read-only** and have filtering/aggregation limits in some widgets — design around this. If you need heavy filtering on a mirrored value, denormalize via a `formula` or an automation-populated column.
5. For very large many-to-many relationships, use a join board.

### ✅ Setting `boardIds` on `board_relation` columns — use `create_column.defaults`

**The single most under-documented mutation arg in the monday API.** `create_column` accepts a `defaults: JSON` argument that lets you set `boardIds` at column-creation time. Verified end-to-end on `2026-07`:

```graphql
mutation {
  create_column(
    board_id: 12345,
    title: "Customer Account",
    column_type: board_relation,
    defaults: "{\"boardIds\":[67890],\"allowCreateReflectionColumn\":true}"
  ) { id title settings_str }
}
```

The returned `settings_str` confirms the wiring: `"{\"boardIds\":[67890],\"allowCreateReflectionColumn\":true}"`. After that, `change_multiple_column_values` with `{"item_ids": [...]}` works immediately.

**Important caveats (verified):**
- `defaults` is a top-level argument on `create_column`, NOT inside `columnSettings`. The MCP `create_column` tool wrapper exposes `columnSettings` for column-type config (e.g. status labels), but you must use raw GraphQL via `all_monday_api` to access the `defaults` argument for `board_relation`.
- `defaults` works for `board_relation` boardIds. It may also work for other column types' initial config — test before assuming.
- Pass the JSON as a string (escape inner quotes), since `defaults` is typed `JSON` (scalar, accepts string).
- Set `allowCreateReflectionColumn: true` so the reverse-side column gets auto-created on the linked board (typical CRM-style behavior).
- For the reverse column on the OTHER side, create a second `board_relation` column with `defaults: {boardIds:[<this board>], allowCreateReflectionColumn:false}` to avoid an infinite reflection chain.

#### What does NOT work (also verified)

`update_column(settings: JSON)` and `change_column_metadata` cannot set `boardIds` AFTER creation:
- `update_column` with `settings: "{\"boardIds\":[...]}"` returns `"Column schema validation failed"` (every JSON shape rejected).
- `change_column_metadata.column_property` enum has only `title` and `description`. No `settings` / `boardIds` option.

**Therefore: if a `board_relation` column already exists with empty `boardIds`** (`settings_str: "{}"`), you must:
1. `delete_column` it (if non-mandatory)
2. Recreate via `create_column` with `defaults`

If the existing column is **mandatory** (e.g. `bill_to` on native Quotes & Invoices, `board_relation6` on the native ITSM Tickets board), it cannot be deleted. In that case, create a NEW supplementary `board_relation` column alongside it — the mandatory empty one stays as a UI artifact, but your new column carries the data.

> **Demo note (verified May 2026):** If you migrate away from the column a mandatory `board_relation` was pointing at (e.g. archiving a Customers board and replacing with CRM Accounts), the mandatory column becomes a broken empty artifact visible in every item. It cannot be deleted or hidden via API — only hidden from views manually in the UI. Tell the user to hide it via the column chooser. Do not leave it as a visible empty column in a demo.

#### Native columns — already wired

Native CRM/Dev/Service `board_relation` columns ship with `boardIds` already configured (e.g. `contact_account`, `deal_contact`, `task_sprint`, `bug_tasks`, the ITSM `connect_boards2` Tickets↔Incidents pairing). Don't recreate them — use them.

#### When user has ALREADY created a column with empty boardIds

If the user manually created a `board_relation` column in the UI without picking a connected board (showing `settings_str: "{}"`), the API can't fix it. Two options:
1. Have them open the column header → Settings → Connect boards → pick target → Save (UI fix).
2. Or delete that column and recreate via `create_column.defaults` from the API.

**Note on `link_board_items_workflow`:** The descriptions of `change_item_column_values` and `get_board_items_page` mention a `[REQUIRED PRECONDITION]` to call `link_board_items_workflow` for board-relation tasks — **but that tool is NOT exposed as a callable MCP tool, and writing/reading `board_relation` columns works fine without it (verified end-to-end in May 2026), as long as `boardIds` was set at creation per the `defaults` rule above.** Treat the precondition note as legacy/aspirational documentation, not a real requirement.

CRM-style schema example:
- `Accounts` ← `Contacts` (board_relation on Contacts → Accounts; mirror Account Name back)
- `Accounts` ← `Deals` (board_relation on Deals → Accounts; mirror Account Name; mirror Owner)
- `Deals` ← `Activities` (board_relation on Activities → Deals)

### ⚠️ Mirror column creation — `create_column` tool ALWAYS FAILS — use `all_monday_api` with raw `defaults`

**Verified end-to-end May 2026.** The `create_column` MCP tool rejects mirror column creation with a schema validation error regardless of how you structure the `columnSettings` argument. The tool's JSON Schema does not include the `defaults` field needed for mirrors.

**The only working method** is raw GraphQL via `all_monday_api`:

```graphql
mutation {
  create_column(
    board_id: <board_id>,
    title: "Account Tier",
    column_type: mirror,
    defaults: "{\"mirrorType\":\"lookup\",\"relation_column_id\":\"<board_relation_column_id>\",\"displayed_linked_column_ids\":[\"<source_column_id_on_linked_board>\"]}"
  ) { id title }
}
```

Key rules:
- `relation_column_id` — the ID of the `board_relation` column on THIS board that links to the source board. Must already exist and have `boardIds` configured.
- `displayed_linked_column_ids` — array of column IDs on the **linked board** whose values you want to surface. Get these by calling `get_board_info` on the linked board first.
- `defaults` is a JSON string (escape all inner quotes). It is NOT a nested object.
- The API response will include `"Column value type is not supported"` in the `column_values` field when reading mirror values back via GraphQL — this is a known API quirk. The column **renders correctly in the UI**. Don't treat this as an error.
- If you create a mirror pointing at the wrong source column, you cannot update it — `change_column_metadata` only supports `title` and `description`. Delete the mirror and recreate it with the correct `displayed_linked_column_ids`.
- Always verify the source column IDs with `get_board_info` on the linked board before executing. Pointing at the wrong column ID produces a silent failure (column exists but shows nothing).

---

