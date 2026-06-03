# Dev, Notetaker, search, users/teams, Objects platform, knowledge base

> Verified reference for the `monday-architect` skill (progressive disclosure). Loaded on demand; the decision core lives in that skill's SKILL.md.

## 13. Dev product (monday Software)

- `get_monday_dev_sprints_boards` — find sprint boards.
- `get_sprints_metadata` — sprint definitions on a board.
- `get_sprint_summary` — burnup/burndown/summary for a sprint.
- `sprints` query (raw GraphQL) — full sprint collection.
- Native types: `Sprint`, `SprintSnapshot`, `SprintTimeline`, `SprintState`.
- Use Epics and Releases as native objects on `software` workspaces; don't reinvent them as Status options.
- `enroll_items_to_sequence` (raw GraphQL mutation, verified name) — enroll items into a sequence; pre-check eligibility with the `allowed_sequences_to_enroll` query.

> **Sprint provisioning is UI-only.** `create_sprint` is NOT in the API schema as of `2026-07`. To set up the native engineering board pair (Sprints + Tasks + Epics + Bugs Queue + Retrospectives + Capacity), the user must add the **Sprint template** in the monday UI (`Workspace → Add → Templates → Sprint`). See §1.5 "monday Dev — TWO native variants" for the full board structure of each variant.

---

## 14. Notetaker (meetings)

- `get_notetaker_meetings` — list meetings.
- Raw GraphQL: `notetaker { meetings(limit, cursor, filters) { ... } }` — paginated. `Meeting` fields (verified): `id`, `title`, `start_time`, `end_time`, `recording_duration`, `access_type`, `meeting_link`, `summary`, `participants`, `topics`, `action_items`, `transcript`. (All snake_case.)
- Use to extract action items / decisions from meetings into items.

---

## 15. Search and discovery

- `search` — global, multi-entity. Returns boards/items/docs with tailored filters per entity.
- `search` raw query supports a `SearchStrategy` arg with values (verified): `SPEED` (fastest, lower quality), `BALANCED` (default), `QUALITY` (best quality, slower).
- Marketplace app search comes in four flavors (separate queries, not strategies): `marketplace_vector_search`, `marketplace_fulltext_search`, `marketplace_hybrid_search`, `marketplace_ai_search`.
- `ask_developer_docs` — AI Q&A against monday's developer docs.
- Always run a `search` BEFORE creating something with a name the user mentioned — avoid duplicates.

---

## 16. Users, teams, roles, departments

- `list_users_and_teams` — enumerate principals.
- `get_user_context` — current user.
- Raw GraphQL: `users`, `teams`, `account_roles`, `departments`, `me`.
- User lifecycle mutations: `invite_users`, `activate_users`, `deactivate_users`, `update_users_role`, `update_multiple_users`, `update_email_domain`.
- Team mutations: `create_team`, `delete_team`, `add_users_to_team`, `remove_users_from_team`, `assign_team_owners`, `remove_team_owners`.
- Department mutations: `create_department`, `update_department`, `delete_department`, `assign_department_members`, `assign_department_owner`, `unassign_department_owners`, `clear_users_department`.
- Board membership: `add_users_to_board`, `add_teams_to_board`, `delete_subscribers_from_board`, `delete_teams_from_board`.
- Workspace membership: `add_users_to_workspace`, `add_teams_to_workspace`, `delete_users_from_workspace`, `delete_teams_from_workspace`.
- **Directory resources:** `get_directory_resources` (query) + `update_directory_resources_attributes` (mutation) — manage resource attributes (Job Role, Skills, Location) for multiple users in the directory. `DirectoryResourceAttribute` enum: `JOB_ROLE`, `SKILLS`, `LOCATION`.
- Use real user/team IDs in `people`/`team` columns and notifications.

---

## 17. Objects platform (advanced)

monday's newer Objects platform models things like workflows/projects as first-class objects with relations:

- `object_types_unique_keys` — list available object types. Each is identified by an `object_type_unique_key` formatted as `app_slug::app_feature_slug` (per-schema docstring); examples seen in docs are `'workflows'`, `'projects'`. Don't hardcode these — call the query.
- `objects` — query objects with filters.
- `object_relations` — fetch relations for an object.
- **Full CRUD mutations:**
  - `create_object(object_type_unique_key, ...)` — create any object type (board, doc, dashboard, workflow, CRM, etc.). Under the hood creates a board with the corresponding `app_feature_id`.
  - `update_object(input: UpdateObjectInput!)` — update an object.
  - `delete_object(id: ID!)` — permanently delete (reversible within 30 days). Works for any object type.
  - `archive_object(id: ID!)` — archive (preserves all data, hidden from regular views). Prefer over `delete_object` when reversibility matters.
  - `publish_object(id: ID!)` / `unpublish_object(id: ID!)` — move object between draft and public state.
  - `create_object_relations(...)` / `delete_object_relation(...)` — manage relations between objects.
  - `add_subscribers_to_object(...)` — add users as subscribers or owners (equivalent to `add_users_to_board` for the Objects API).

Use this when working with cross-cutting object types beyond boards/items.

---

## 18. Knowledge base

- `articles`, `article_blocks` — published KB articles (query-side).
- `knowledge_base_search` — search snippets.
- **Article mutations (full CRUD):**
  - `create_article(workspace_id, name?, folder_id?)` — create a new article. Returns article metadata.
  - `publish_article(object_id, privacy?, folder_id?, subscribers?)` — publish a draft article. Sets privacy level and manages subscribers.
  - `update_article_block(block_id, ...)` — update content of a specific block in a **draft** article. Cannot update blocks of published articles.
  - `delete_article(object_id)` — delete an article permanently.

---

