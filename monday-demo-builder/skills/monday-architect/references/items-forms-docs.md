# Items, groups, bulk operations, forms, docs, updates

> Verified reference for the `monday-architect` skill (progressive disclosure). Loaded on demand; the decision core lives in that skill's SKILL.md.

## 6. Items, groups, and bulk operations

- **Create item:** `create_item` with `boardId, name, columnValues` (and optional `groupId`).
- **Create subitem:** use `create_item` with `parentItemId` set — there is no separate `create_subitem` MCP tool. Subitems live on a hidden auto-generated board (e.g. `Subitems of [BoardName]`); the response includes that board's ID. The raw GraphQL `create_subitem` mutation also exists for use via `all_monday_api`.
- **Update values:** `change_item_column_values`. Pass `createLabelsIfMissing: true` when writing a status/dropdown label that may not yet exist on the column — without it the call fails. (Requires permission to change board structure.)
- **Move/duplicate:** `move_object` (relocate boards/docs/dashboards between workspaces or folders), `move_item_to_group(item_id, group_id)` (move item to a different group within the same board — distinct from `move_item_to_board` which crosses boards).
- **Delete:** `delete_item`, `delete_group`, `delete_board`.
- **Group:** `create_group`, `update_group`, `delete_group`. Groups are intra-board sections. New items always land in the **top group** — make the most relevant group the top group.
- **Bulk read:**
  - `get_board_items_page` (single page, up to 500/page; supports filters, search, ordering, sub-items, item-description).
  - `next_items_page` via `all_monday_api` (continue with cursor) — required for >500 items.
  - `items_page_by_column_values` (search across columns).
- **Bulk read everything (one board):** `get_full_board_data` is internal-only and triggered by UI components — DO NOT call directly. Use `get_board_info` + `get_board_items_page` paginated.
- **Aggregations:** `board_insights` (named tool) supports SUM/AVG/MIN/MAX/COUNT/COUNT_DISTINCT/MEDIAN with group-by + filters. The `aggregate` raw GraphQL query is even more flexible. Use these instead of fetching all items + counting client-side.
- **Bulk write:** prefer multiple `create_item` calls; for large batches, fall back to `all_monday_api` with a multi-mutation document or use bulk-import jobs (`UploadJobInit`, `ItemsJobStatus`).
- **Async bulk delete/archive:** `bulk_delete_items` and `bulk_archive_items` — asynchronously delete or archive all items on a board. Returns a job ID; poll with `fetch_job_status`. Useful for demo reset between sessions (faster than per-item deletion).
- **Undo:** `undo_action(id: ID!)` — undo a previously completed action or cancel one still in flight. Returns `UndoResult`. Useful for error recovery after a bad bulk operation.
- **Tags:** `create_or_get_tag`.
- **Files:** `update_assets_on_item` (replace assets), `add_file_to_column` (append to a file column), `add_file_to_update` (attach to an update).
- **Archive vs delete:** archive (reversible) preferred for user-facing data — `archive_board`, `archive_group`, `archive_item`. Use `delete_*` only when permanent removal is intended.
- **Position / movement:** `change_item_position` (within group), `move_item_to_board` (move item across boards), `move_object` (relocate boards/docs/dashboards between workspaces or folders).
- **Item description (rich-text body):** `set_item_description_content` accepts markdown; `add_content_to_doc_from_markdown` appends to a doc.
- **Column structural changes:** `change_column_metadata`, `change_column_title`, `delete_column`, `add_required_column`, `remove_required_column`. **`change_column_metadata.column_property` is an enum with only `title` and `description`** — it does NOT expose column settings (e.g., `boardIds` for `board_relation`). For label/option updates use `update_status_column` / `update_dropdown_column`. For `board_relation.boardIds` see §5 — UI-only.
- **Board-level metadata:** `update_board(board_id: ID!, board_attribute: BoardAttributes!, new_value: String!)` — the response is bare JSON, NOT a `Board` object, so do NOT add a `{ id }` selection (you'll get `Field "update_board" must not have a selection since type "JSON" has no subfields`). `BoardAttributes` enum includes `description`, `name`, etc. Multiple `update_board` calls in one mutation document need aliases (e.g. `a: update_board(...)`, `b: update_board(...)`).
- **Board hierarchy:** `update_board_hierarchy(input: UpdateBoardHierarchyAttributesInput!)` — update parent/child board relationships for portfolio-style hierarchies (`BoardHierarchy` enum). Use when wiring project boards into a portfolio structure.
- **Dependency updates:** `update_dependency_column` (single item) or `batch_update_dependency_column` (≤50 items per batch) — both update dependency relationships on items.
- **Clear updates:** `clear_item_updates(item_id: ID!)` — removes all updates (activity thread) from a single item. Useful for resetting demo items between sessions.
- **Bulk import jobs:** `backfill_items` (no side effects, ≤20k rows, for migrations) vs `ingest_items` (full side effects, ≤10k rows, for ongoing integrations). Each returns a job ID + upload URL; check status with `fetch_job_status`.
- **Alternative single-value writes:** `change_column_value`, `change_simple_column_value`, `change_multiple_column_values` exist alongside `change_item_column_values` — use whichever matches the granularity needed.
- **Timeline items:** `create_timeline_item` / `delete_timeline_item` (raw GraphQL) — create/delete custom timeline entries on an item.
- **Custom activities:** `create_custom_activity` / `delete_custom_activity` (raw GraphQL) — manage custom activity types for the activity log.

---

## 8. Forms

- **`create_form` auto-creates a backing board for responses** — you do NOT need to create the board first. Pass `destination_workspace_id` (required), optional `destination_name`, `destination_folder_id`, `board_kind`, owners/subscribers. Returns `{board_id, form_token}`. Use the `form_token` for all subsequent question/setting operations.
- **Edit questions:** `form_questions_editor` with `action: "create"|"update"|"delete"` and a question payload. Alternatively, use the dedicated typed mutations: `create_form_question`, `update_form_question`, `delete_question`. Verified question types (24): `Boolean`, `ConnectedBoards`, `Country`, `DISPLAY_TEXT`, `Date`, `DateRange`, `Email`, `File`, `HOUR`, `Link`, `Location`, `LongText`, `MultiSelect`, `Name`, `Number`, `PAGE_BLOCK`, `People`, `Phone`, `Rating`, `ShortText`, `Signature`, `SingleSelect`, `Subitems`, `Updates`. Question type cannot be changed after creation — always include `type` in update calls.
- **Question settings (verified by type):** `defaultCurrentDate`/`includeTime` (Date), `display` (Single/MultiSelect: Dropdown/Horizontal/Vertical), `optionsOrder` (Alphabetical/Custom/Random), `labelLimitCount`+`label_limit_count_enabled` (MultiSelect), `prefill` (`{enabled, source: Account|QueryParam, lookup}`), `prefixAutofilled`/`prefixPredefined` (Phone), `default_answer`, `skipValidation` (Link), `checkedByDefault` (Boolean), `locationAutofilled`.
- **Conditional logic:** `show_if_rules` with `operator: OR` and rule conditions referencing `building_block_id`.
- **Form lifecycle:** `activate_form` (make visible, accept submissions) / `deactivate_form` (hide form, stop submissions — data preserved). Both take `form_token`.
- **Password protection:** `set_form_password(input: SetFormPasswordInput!)` — enables password restriction on a form.
- **Shorten URL:** `shorten_form_url` — generates and stores a shortened link; returns `FormShortenedLink`.
- Modify settings with `update_form` / `update_form_settings`. Inspect with `get_form`.
- Use the `form` query (raw GraphQL) by token to fetch a form for display/processing.
- Form features available via settings: tags, AI translate, password protection, response limits, close date, redirect after submission, accessibility, draft submissions, prefill, pre/post submission views, custom logo/background/layout.

---

## 9. Docs (workdocs)

- **Create:** `create_doc` accepts a `markdown` parameter directly — the markdown is auto-imported as blocks. Two location modes: `location: "workspace"` (with `workspace_id` + optional `folder_id` + `doc_kind`) or `location: "item"` (attaches doc to an item via a doc column).
- **Other:** `read_docs`, `update_doc`, rename with `update_doc_name`, delete with `delete_doc`, `duplicate_doc`.
- **Block-level editing** (raw GraphQL mutations): `create_doc_block`, `create_doc_blocks`, `update_doc_block`, `delete_doc_block` — use for granular doc updates after initial markdown import. **`create_doc_blocks` args are `docId` and `blocksInput` (camelCase variant); max 25 blocks per call (verified — 26 throws `A maximum of 25 blocks can be created at once`).**
- **`CreateBlockInput` block types (verified union):** `text_block`, `list_block`, `notice_box_block`, `image_block`, `video_block`, `table_block`, `layout_block`, `divider_block`, `page_break_block`. (Mentions are NOT a top-level block type — they're embedded within text blocks.)
- **Text block content shape** (verified): `{text_block: {delta_format: [{insert: {text: "..."}}]}}`. The `insert` field is an `InsertOpsInput` object with `text` or `blot`, NOT a bare string.
- **Append markdown to existing doc:** `add_content_to_doc_from_markdown` — easier than building blocks manually.
- Advanced (raw GraphQL): `doc_version_history`, `doc_version_diff`, `export_markdown_from_doc`, `import_doc_from_html`, `articles` / `article_blocks` / `update_article_block` (knowledge base).
- Use docs for: PRDs, meeting notes, wikis, runbooks, briefs, SOPs.
- Don't put narrative content in a `long_text` column — it's not searchable, structured, or shareable the same way.
- **`create_view_table` / `update_view_table`** — typed mutations for table views specifically (alternative to the generic `create_view` / `update_view`). Use when you need explicit table-view creation with strongly-typed settings.

---

## 10. Updates, replies, notifications, assets

- `create_update` — post an update on an item (the canonical activity log). Supports mentions.
- `get_updates` — fetch updates.
- `replies` query (raw GraphQL) — get replies on updates.
- `like_update` / `unlike_update` / `pin_to_top` / `unpin_from_top` / `edit_update` / `delete_update` (raw GraphQL).
- `create_notification` — push a notification to a user. `NotificationTargetType` has two values: `Post` (an Update) and `Project` (an Item or Board) — pick by what the user should be linked to.
- `update_notification_setting` — update a user's notification preferences (raw GraphQL).
- `notifications_settings` / `mute_board_settings` (raw GraphQL) — read user preferences.
- `get_assets` — list files attached to items/updates.
- `get_board_activity` — board audit log.

---

