---
name: refresh-monday-skill
description: Verify the monday-architect skill against the live monday MCP and report any drift since the last verification. Run this when the user types `/refresh-monday-skill`, after a monday API release announcement, before a high-stakes demo, or on a periodic cadence (e.g. monthly via the schedule skill). Re-introspects every named mutation/query, every enum cited in the skill, and the widget/view catalogs (Layers 1–2), then produces a diff report and (with user approval) patches `the `monday-architect` skill files (its SKILL.md + everything in references/)`. Also includes an opt-in Layer 3 "documentation coverage" audit that reads monday's prose developer docs to find behavioral/conceptual gaps (limits, auth, payload shapes, gotchas) that schema introspection cannot see — run that when the user asks "check the docs / is the architect missing anything".
---

# refresh-monday-skill — Layer 2 verification routine

> **Adapted for the `monday-demo-builder` plugin.** Two things differ from the original single-file skill:
>
> 1. **The architect's facts are split across files.** Wherever this skill says to grep or read the monday-architect `SKILL.md`, grep/read the **entire `monday-architect` skill folder** — `SKILL.md` plus every file in `references/` — because the verified enums, mutation names, value-shapes, and gotchas now live across those files (section numbers like §4, §7, §27 are preserved). When patching a drifted fact, edit whichever file (SKILL.md or the relevant `references/*.md`) actually contains it.
> 2. **The MCP tool prefix is connector-specific.** Tool names below use `mcp__claude_ai_monday_com__*` from the original Claude.ai monday connector. Your connector may expose the same tools under a different prefix — match on the base tool name (e.g. `get_user_context`), not the prefix.

This skill keeps `the `monday-architect` skill files (its SKILL.md + everything in references/)` in sync with monday. It does NOT execute any account-mutating builds (no test workspaces, no created boards). It works in two modes:

- **Schema verification (Layers 1–2, always run):** introspection queries + MCP tool-roster + version checks, compared to the facts the architect skill asserts. Fast, deterministic, safe for the unattended weekly tripwire.
- **Documentation coverage audit (Layer 3, opt-in — Step 3.5):** reads monday's prose developer docs to find *concepts* the skill is missing entirely (behavioral facts, limits, auth rules, payload shapes, gotchas) that introspection cannot return. Slower, partially bot-gated, judgment-heavy. Run only on explicit request or a slow cadence.

If the user's intent is "is the architect missing anything / check the whole API docs", they want **Layer 3** — don't stop at the schema check. When invoked, work through these steps in order (run Step 3.5 only per its own guidance):

## Step 1 — Confirm prerequisites

- Confirm the monday MCP connector is connected and authenticated. Run `mcp__claude_ai_monday_com__get_user_context` once. If it errors, stop and tell the user to authenticate.
- Read the current state of `the `monday-architect` skill files (its SKILL.md + everything in references/)` so you have the asserted facts to compare against.

## Step 2 — Re-introspect the load-bearing facts

Run these in parallel (they are all read-only):

| Tool / query | What it confirms |
|---|---|
| `all_monday_api` running `query { versions { kind value } version { kind value } }` | **Layer 2 — API version watch.** The list of API versions and the connector's current default. Compare the newest `release_candidate` against the version the architect skill says it was verified against (header line + inline `release_candidate` mentions). |
| `ToolSearch` for `mcp__claude_ai_monday_com` (high `max_results`, e.g. 80) | **Layer 2 — MCP tool roster.** The set of tools the connector currently exposes. Diff against the roster snapshot in this skill (see check #17). New tools = new connector functionality not reachable via GraphQL introspection. |
| `mcp__claude_ai_monday_com__get_graphql_schema(operationType: "read")` | Full list of query field names + GraphQL type names |
| `mcp__claude_ai_monday_com__get_graphql_schema(operationType: "write")` | Full list of mutation field names |
| `mcp__claude_ai_monday_com__all_widgets_schema` | Every dashboard widget type and its config schema |
| `mcp__claude_ai_monday_com__get_user_context` | Active product kinds on this account |
| `get_type_details(BoardKind)` (raw GraphQL) | Board visibility enum |
| `get_type_details(WorkspaceKind)` | Workspace visibility enum |
| `get_type_details(DashboardKind)` | Dashboard visibility enum |
| `get_type_details(ColumnType)` | Full column type enum |
| `get_type_details(WebhookEventType)` | All webhook event types |
| `get_type_details(WorkspacesQueryAccountProductKind)` | All product kinds |
| `get_type_details(ViewKind)` | Creatable view types |
| `get_type_details(SearchStrategy)` | Search strategy enum |
| `get_type_details(NotificationTargetType)` | Notification target enum |
| `get_type_details(DuplicateBoardType)` | Board duplication mode enum |
| `get_type_details(BoardMuteState)` | Board mute state enum |
| `get_type_details(BulkImportState)` | Bulk import job state enum |
| Tool schema for `create_widget` (via ToolSearch select) | The 7 creatable widget kinds |
| Tool schema for `create_column` | The full ColumnType enum exposed via the MCP wrapper |
| Tool schema for `create_folder` | Folder color/icon/font-weight enums |

> **Expect the `get_graphql_schema(operationType: "write")` result to overflow the tool output limit** (~55k chars) and be written to a file instead of returned inline — the tool result will give you the file path. Don't treat this as an error. Read the file (or, faster, parse it with a `python3 -c` one-liner: `json.load` → `obj["mutation_fields"]` → `[m["name"] for m in ...]`) to get the full mutation-name list. The `read` schema usually fits inline. Run all the rows above in parallel — they are independent read-only calls.

## Step 3 — Build a diff report

For every fact the architect skill asserts, mark one of:

- **OK** — schema matches the skill verbatim.
- **DRIFT** — schema differs from the skill (new value added, old value removed, renamed). Capture the specific change.
- **GONE** — the named mutation/query/type no longer exists. High-priority fix.
- **NEW** — the schema has something the skill doesn't mention (a new mutation, a new enum value, a new widget type). Lower priority but worth noting.

Specifically check these load-bearing claims in `SKILL.md`:

1. **Product kinds** (§1, §27): `core`, `crm`, `software`, `service`, `marketing` / `marketing_campaigns`, `project_management`, `forms`, `whiteboard`. Compare against `WorkspacesQueryAccountProductKind` enum.
2. **BoardKind** (§3, §27): `private`, `public`, `share`.
3. **WorkspaceKind** (§2, §27): `open`, `closed`, `template`.
4. **DashboardKind** (§7, §27): `PUBLIC`, `PRIVATE`.
5. **ColumnType** (§4, §27): the 40+ values in §4. Compare against the live enum AND against the `create_column.columnType` tool-schema enum.
6. **WebhookEventType** (§11, §27): the 21 values in §11.
7. **Widget catalog** (§7, §27): `NUMBER`, `CHART`, `BATTERY`, `CALENDAR`, `GANTT`, `LISTVIEW`, `APP_FEATURE`. Compare against both `all_widgets_schema` keys AND `create_widget.widget_kind` tool-schema enum.
8. **CHART graph_type variants** (§7): 24 values — 8 base types (`pie`, `donut`, `column`, `bar`, `area`, `line`, `smooth_line`, `bubbles`) + 8 underscore stacked (`stacked_bar`, `stacked_bar_percent`, `stacked_column`, `stacked_column_percent`, `stacked_area`, `stacked_area_percent`, `stacked_line`, `stacked_line_percent`) + 8 camelCase stacked aliases (`stackedBar`, `stackedBarPercent`, …). Both stacked forms are accepted by the API. Compare against the `graph_type` enum inside the `CHART` entry of `all_widgets_schema`.
9. **ViewKind** (§3, §27): `DASHBOARD`, `TABLE`, `FORM`, `APP`.
10. **DuplicateBoardType** (§3.5): `duplicate_board_with_pulses`, `duplicate_board_with_pulses_and_updates`, `duplicate_board_with_structure`.
11. **NotificationTargetType** (§10): `Post`, `Project`.
12. **SearchStrategy** (§15): `SPEED`, `BALANCED`, `QUALITY`.
13. **BoardMuteState** (§25): `NOT_MUTED`, `MUTE_ALL`, `MENTIONS_AND_ASSIGNS_ONLY`, `CUSTOM_SETTINGS`, `CURRENT_USER_MUTE_ALL`.
14. **All cited mutation names** — **derive the list at run time, don't trust the frozen copy below.** First `grep` the architect `SKILL.md` for backticked snake_case tokens and intersect them with the live write-schema mutation field names; that intersection is the authoritative "cited mutations" set for this run. Verify each appears in the write schema. The list below is a **fallback snapshot** (last refreshed 2026-06-01) — if your grep yields names not in it, the architect skill added mutations since this snapshot, and the snapshot itself is stale and should be regenerated. Snapshot of names cited:
    `create_workspace`, `update_workspace`, `delete_workspace`, `create_folder`, `update_folder`, `delete_folder`, `move_object`, `use_template`, `update_board_hierarchy`, `create_board`, `update_board`, `delete_board`, `archive_board`, `duplicate_board`, `create_column`, `update_column`, `delete_column`, `create_status_column`, `update_status_column`, `create_dropdown_column`, `update_dropdown_column`, `attach_status_managed_column`, `attach_dropdown_managed_column`, `create_status_managed_column`, `update_status_managed_column`, `create_dropdown_managed_column`, `update_dropdown_managed_column`, `activate_managed_column`, `deactivate_managed_column`, `delete_managed_column`, `create_group`, `update_group`, `delete_group`, `archive_group`, `duplicate_group`, `create_item`, `create_subitem`, `change_item_column_values`, `change_simple_column_value`, `change_multiple_column_values`, `change_column_value`, `change_column_metadata`, `change_column_title`, `change_item_position`, `move_item_to_board`, `move_item_to_group`, `delete_item`, `archive_item`, `duplicate_item`, `bulk_delete_items`, `bulk_archive_items`, `undo_action`, `clear_item_updates`, `add_file_to_column`, `add_file_to_update`, `update_assets_on_item`, `set_item_description_content`, `add_required_column`, `remove_required_column`, `update_dependency_column`, `batch_update_dependency_column`, `create_timeline_item`, `delete_timeline_item`, `create_custom_activity`, `delete_custom_activity`, `create_doc`, `update_doc`, `update_doc_name`, `delete_doc`, `duplicate_doc`, `read_docs`, `create_doc_block`, `create_doc_blocks`, `update_doc_block`, `delete_doc_block`, `add_content_to_doc_from_markdown`, `import_doc_from_html`, `create_view_table`, `update_view_table`, `create_dashboard`, `update_dashboard`, `update_overview_hierarchy`, `delete_dashboard`, `create_widget`, `delete_widget`, `create_view`, `update_view`, `delete_view`, `create_form`, `form_questions_editor`, `update_form`, `update_form_settings`, `update_form_question`, `delete_question`, `create_form_question`, `create_form_tag`, `update_form_tag`, `delete_form_tag`, `activate_form`, `deactivate_form`, `set_form_password`, `shorten_form_url`, `create_update`, `edit_update`, `delete_update`, `like_update`, `unlike_update`, `pin_to_top`, `unpin_from_top`, `create_notification`, `update_notification_setting`, `update_mute_board_settings`, `create_webhook`, `delete_webhook`, `execute_integration_block`, `create_validation_rule`, `update_validation_rule`, `delete_validation_rule`, `create_or_get_tag`, `create_workspace`, `create_team`, `delete_team`, `add_users_to_team`, `remove_users_from_team`, `assign_team_owners`, `remove_team_owners`, `add_users_to_board`, `add_teams_to_board`, `delete_subscribers_from_board`, `delete_teams_from_board`, `add_users_to_workspace`, `add_teams_to_workspace`, `delete_users_from_workspace`, `delete_teams_from_workspace`, `invite_users`, `activate_users`, `deactivate_users`, `update_users_role`, `update_multiple_users`, `update_email_domain`, `create_department`, `update_department`, `delete_department`, `assign_department_members`, `assign_department_owner`, `clear_users_department`, `unassign_department_owners`, `update_directory_resources_attributes`, `create_project`, `convert_board_to_project`, `create_portfolio`, `connect_project_to_portfolio`, `enroll_items_to_sequence`, `create_object_schema`, `update_object_schema`, `delete_object_schema`, `connect_board_to_object_schema`, `create_object_schema_columns`, `update_object_schema_columns`, `set_object_schema_column_active_state`, `detach_boards_from_object_schema`, `bulk_object_schema_column_actions`, `update_object`, `create_object`, `delete_object`, `archive_object`, `publish_object`, `unpublish_object`, `create_object_relations`, `delete_object_relation`, `add_subscribers_to_object`, `create_article`, `publish_article`, `update_article_block`, `delete_article`, `backfill_items`, `ingest_items`, `set_board_permission`.
15. **All cited query names** — same exercise against the read schema (`get_graphql_schema(operationType: "read")` → `query_fields[].name`). Derive at run time by intersecting the architect skill's backticked tokens with the live read-schema field names. Fallback snapshot of cited queries (last refreshed 2026-06-01):
    `account`, `account_connections`, `account_roles`, `account_trigger_statistics`, `account_triggers_statistics_by_entity_id`, `aggregate`, `all_widgets_schema`, `allowed_sequences_to_enroll`, `article_blocks`, `articles`, `ask_developer_docs`, `audit_event_catalogue`, `audit_logs`, `block_events`, `complexity`, `connection`, `connection_board_ids`, `connections`, `departments`, `doc_version_diff`, `doc_version_history`, `export_events`, `export_markdown_from_doc`, `fetch_job_status`, `form`, `get_column_type_schema`, `get_directory_resources`, `get_object_schemas`, `get_view_schema_by_type`, `items_page_by_column_values`, `knowledge_base_search`, `marketplace_ai_search`, `marketplace_fulltext_search`, `marketplace_hybrid_search`, `marketplace_vector_search`, `me`, `mute_board_settings`, `next_items_page`, `notifications_settings`, `object_relations`, `object_types_unique_keys`, `objects`, `replies`, `search`, `sprints`, `tags`, `teams`, `timeline`, `tool_events`, `trigger_event`, `trigger_events`, `user_connections`, `users`, `validations`, `version`, `versions`, `webhooks`.

16. **Negative claims — verify what the skill says does NOT exist is still absent.** The architect skill makes load-bearing *absence* claims that the present-check logic above cannot guard. Grep the architect `SKILL.md` for phrases like "does NOT exist", "is NOT in the schema", "not a callable tool", "deprecated". For each named mutation/query/tool in such a claim, confirm it is **still absent** from the live schema. Known negative claims to check every run:
    - **`create_sprint`** — asserted absent in 4 places (§1.5/§13, lines ~180/699/841/984). If `create_sprint` now appears in the write schema, this is a **high-priority DRIFT** (the skill would otherwise keep telling users to provision sprints via the UI for no reason). Verify against `get_graphql_schema(write)`.
    - **`link_board_items_workflow`** — asserted to be referenced as a precondition but NOT a callable MCP tool. Confirm it is still not exposed as a tool (ToolSearch).
    - **`person` column type** — asserted deprecated in favor of `people`. Confirm `person` still carries the "(deprecated)" description in the live `ColumnType` enum.
    A negative claim that has become false is at least as harmful as a missing mutation — treat newly-existing items as **GONE-equivalent** (high priority) in the report.

17. **MCP tool roster (Layer 2 — not visible to GraphQL introspection).** The connector exposes a set of `mcp__claude_ai_monday_com__*` tools that is *not* the same as the GraphQL mutation/query list — some tools are wrappers, some are MCP-only (e.g. `get_board_items_page`, `board_insights`, `manage_agent*`, `create_automation`, `create_workflow`). New connector functionality shows up here first. Run `ToolSearch` for `mcp__claude_ai_monday_com` with a high `max_results` and diff the returned tool names against the snapshot below. Report any **added** tool as 🆕 (new connector capability — worth teaching the architect skill) and any **removed** tool as ❌ (a tool the architect skill may still reference). Roster snapshot — **60 tools**, last refreshed 2026-06-01:
    `agent_catalog`, `all_monday_api`, `all_widgets_schema`, `board_insights`, `change_item_column_values`, `create_automation`, `create_board`, `create_column`, `create_dashboard`, `create_doc`, `create_folder`, `create_form`, `create_form_submission`, `create_group`, `create_item`, `create_notification`, `create_update`, `create_view`, `create_view_table`, `create_widget`, `create_workflow`, `create_workspace`, `finalize_asset_upload`, `form_questions_editor`, `get_asset_upload_url`, `get_assets`, `get_board_activity`, `get_board_info`, `get_board_items_page`, `get_column_type_info`, `get_form`, `get_full_board_data`, `get_graphql_schema`, `get_monday_dev_sprints_boards`, `get_notetaker_meetings`, `get_sprint_summary`, `get_sprints_metadata`, `get_type_details`, `get_updates`, `get_user_context`, `list_automations`, `list_users_and_teams`, `list_workspaces`, `manage_agent`, `manage_agent_knowledge`, `manage_agent_skills`, `manage_agent_triggers`, `manage_workflows`, `move_object`, `publish_workflow`, `read_docs`, `search`, `update_doc`, `update_folder`, `update_form`, `update_view`, `update_view_table`, `update_workflow`, `update_workspace`, `workspace_info`.

18. **API version watch (Layer 2).** Run `all_monday_api` with `query { versions { kind value } version { kind value } }`. The architect skill records which `release_candidate` it was verified against (header line + inline mentions). Report: (a) a **new `release_candidate`** not previously verified against → suggest a targeted re-introspection of that version via the `API-Version` header; (b) the connector's **default `version`** changing kind/value; (c) a version the skill cites having dropped to `maintenance` or disappeared. As of the 2026-06-01 refresh: default = `release_candidate 2026-07`, `current` = `2026-04`, newest RC = `2026-10`. None of this changes enum/mutation facts on its own — flag as **low/medium**, informational.

## Step 3.5 — Documentation coverage audit (Layer 3 — opt-in, heavier)

**Why this exists.** Layers 1–2 verify the *schema surface* (names, enums, tool roster, versions). They CANNOT detect a missing **concept** — a behavioral fact, limit, payload shape, auth rule, or gotcha that lives in monday's prose docs but never appears in GraphQL introspection (e.g. "personal tokens use no `Bearer` prefix", "echo the webhook `challenge` token", rate-limit numbers, multi-level-board query args, read-value≠write-value). Closing that requires reading the docs, not the schema.

**When to run it.** This phase is **slow, token-heavy, and partially bot-gated** (developer.monday.com frequently 403/429s WebFetch). Do NOT run it on the unattended weekly tripwire. Run it when: the user explicitly asks "check the docs / is the architect missing anything", after a major monday product announcement, or on a slow cadence (quarterly). If you're in a fast/scheduled context, skip Step 3.5 and note in the report that the doc audit was not run.

**How to run it.** Fan out parallel research agents (Agent tool, `general-purpose` or `Explore`), each owning ONE doc area, each given the architect skill's CURRENT coverage of that area so it reports only genuine gaps — not restatements. Suggested split (one agent each):
- **API mechanics** — `/docs/rate-limits`, `/reference/complexity`, `/docs/error-handling`, `/docs/api-versioning`, `/docs/basics`, `/docs/authentication`. Targets: rate-limit tier numbers, complexity budgets, error codes + HTTP status, version rings, endpoint/header/`Bearer` rules.
- **Column-value JSON shapes** — `/reference/column-types-reference` + per-type pages. Targets: exact write-value JSON per column type, read-only types, read-vs-write differences.
- **Changelog / recent features** — `/api-reference/changelog`, `/docs/release-notes`. Targets: new column types, mutations, widgets, object types, deprecations/sunset dates, the API version they landed in.
- **Webhooks / files / OAuth / auth** — `/docs/webhooks`, `/reference/webhooks`, `/docs/files`, `/reference/assets-1`, `/apps/docs/oauth`. Targets: challenge handshake, `config` per event, payload field names (`pulseId`), multipart `/v2/file` upload, scope list, token types.
- **Item querying** — `/reference/items-page`, `/reference/items-page-by-column-values`, `ItemsQuery`/operator types. Targets: raw `query_params` shape, full operator enum, `groups` AND/OR nesting, limits, cursor lifetime.

**Critical discipline (learned the hard way):**
- **Cross-check every doc claim against the live schema before reporting it as a gap.** The docs site bot-gates fetches, so agents fall back to secondary sources (community posts, third-party guides) that are often stale or wrong. Any fact reachable via `get_type_details`/`get_graphql_schema` should be confirmed there — the live schema outranks the prose docs for type/enum/arg shapes.
- **Filter ruthlessly against what the skill already covers.** Most raw agent findings will already be in the architect skill (it's ~1200 lines). `grep` the skill for each candidate before proposing it. Report only true additions.
- **Mark confidence.** Facts confirmed against the live schema = "verified". Facts that came only from 403-gated prose / secondary sources = label them "doc-sourced — verify against live" in both the report and any patch text. Never assert a secondary-source number as confirmed.
- These are **judgment-heavy additions**, not mechanical enum swaps — always show proposed edits and get approval (Step 5) before writing. Lean toward fewer, higher-confidence additions.

Feed confirmed gaps into the Step 4 report under a dedicated **"Doc-coverage gaps (Layer 3)"** block.

## Step 4 — Report findings

Output a concise report to the user with this structure:

```
monday-architect skill verification report — <date> (API <release_candidate version>)

✅ <N> facts verified unchanged
⚠️  <N> drifts detected
❌ <N> facts now invalid (mutation/query/enum gone)
🆕 <N> new things in the schema not yet in the skill

Drifts:
- <area>: was "X", now "Y" (impact: <high/medium/low>)
...

Gone:
- <name>: <description>

Negative claims now false (high priority — skill asserts these don't exist but they do):
- <name>: <description>

New (worth adding):
- <name>: <one-line description>

MCP tool roster delta (Layer 2):
- + <new tool>: <what it does>   /   - <removed tool>

API version watch (Layer 2):
- <e.g. new RC 2026-10 not yet verified against; default still 2026-07>

Doc-coverage gaps (Layer 3 — only if Step 3.5 was run; else write "doc audit not run this pass"):
- <concept the skill is missing>: <one-line fact> [verified | doc-sourced] (<source URL>)
```

Cap the report at ~30 lines (Layer 3 may push longer — that's fine when the doc audit ran). Don't dump full enum lists unless something changed.

## Step 5 — Patch the skill (with user approval)

For each DRIFT and GONE finding:
- Show the exact `Edit` you propose to make to `SKILL.md`.
- Wait for user approval.
- Apply the edit.

For NEW findings, ask the user whether to add them. Don't add silently — new mutations/types are often beta features the user may not want to rely on yet.

After patches are applied, update the "Verified against monday API" line at the top of `SKILL.md`. If that line doesn't exist yet, add one right after the YAML frontmatter:

```markdown
> Last verified end-to-end against the monday MCP on <YYYY-MM-DD> (API release_candidate <version>). Run `/refresh-monday-skill` to re-verify.
```

## Step 6 — If clean, just report and stop

If everything checks out (no DRIFT, GONE, or NEW), output a one-liner:

```
monday-architect verified clean against API <version> on <date>. No changes needed.
```

…and update the "Last verified" line in the skill so the user can see it ran.

## Out of scope

This skill does NOT:
- Re-execute end-to-end build flows (that's Layer 3 — the manual demo dry-run).
- Test webhooks, file uploads, validation rules, or anything that would mutate the live account.
- Verify the skill's *opinionated* content (demo seed-data sizes, cross-product archetypes, anti-patterns) — those are judgment calls, not API facts.

If the user asks "is the skill 100% correct?", the honest answer is: this skill verifies the API surface; it does not verify behavioral changes that introspection can't see (e.g. a mutation that still exists but now requires a permission it didn't before). For that, recommend running the round-2 manual test from the prior chat.
