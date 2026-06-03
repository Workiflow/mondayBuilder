# Demo-build playbook - data realism, archetypes, seeding, teardown

> Verified reference for the `monday-demo-builder` skill. Loaded on demand.

## 26. Demo-build mode

This skill is most often used to build **demo accounts** — fully-loaded accounts (all products enabled) used to show prospects what monday can do. Demos amplify a few priorities on top of the rest of this manual:

### What "good" looks like in a demo
- **Believable data.** Real-sounding company names, contact names, deal values, dates near today (some past, some future, some overdue). Avoid `Item 1`, `Test`, `asdf`. Avoid obviously LLM-flavoured names like "Acme Innovations".
- **Visual polish.** Use status column colors deliberately (red/yellow/green map to risk; not random). Group items meaningfully — e.g., in a sales pipeline, groups = stages. Set group colors via the `groupColor` arg on `create_group` (palette: `#037f4c`, `#00c875`, `#9cd326`, `#cab641`, `#ffcb00`, `#784bd1`, `#9d50dd`, `#007eb5`, `#579bfc`, `#66ccff`, `#bb3354`, `#df2f4a`, `#ff007f`, `#ff5ac4`, `#ff642e`, `#fdab3d`, `#7f5347`, `#c4c4c4`, `#757575`).
- **Filled cells.** Empty columns look unfinished. Seed every column on every demo item — even if it's a sensible default.
- **Working dashboards.** A dashboard with empty widgets is worse than no dashboard. Seed enough items, with enough variety, that every widget on the dashboard has something to show. NUMBER widgets need numeric data; CHART widgets need >1 group; BATTERY widgets need at least one "done" item.
- **Realistic dates.** Spread dates across past/present/future so timeline/calendar/gantt widgets render meaningfully. Seed items in a mix of statuses including some `Stuck` and some `Done` so progress visualizations are non-trivial.
- **Cross-product moments.** The most impressive demos span products: a Deal in CRM that links to a Project in Work Management that links to a Sprint in Dev that has a Service ticket. Build at least one of these flows when relevant.

### Demo build sequence (overrides §24 ordering for speed)
1. `get_user_context` → confirm products + grab favorites/relevant boards (often you'll re-use existing demo boards).
2. **Search first** — `search` for any name the user mentions. Demo accounts accumulate cruft; check for an existing version before building a new one.
3. Pick one demo workspace per scenario. Use `WorkspaceKind: open`. Name it explicitly — `[DEMO] <Scenario>` so it's obvious in lists.
4. Folders by domain inside the demo workspace.
5. Boards with full column sets, real-feel names. **Always seed at least 8–15 items per board** — this is the threshold below which dashboards/widgets look fake.
6. Cross-board links — `change_item_column_values` with `{"item_ids": [...]}` for `board_relation` cells. No precondition call needed.
7. At least one Doc per scenario (project brief, meeting notes, runbook) — populate it with `create_doc` + `create_doc_blocks` (≤25 blocks per call). Demos lose credibility when "Documents" is empty.
8. At least one Form per scenario where intake makes sense (lead capture, support ticket, request form).
9. Dashboard with **multiple widget types** mixed (one NUMBER, one CHART, one BATTERY, one CALENDAR/GANTT) so the dashboard isn't monotone.
10. Optionally one webhook subscribed to a meaningful event (e.g., `create_item` on the leads board) wired to a placeholder URL — shows the integration story.

### Cross-product demo archetypes (build any of these end-to-end)
- **Lead-to-cash:** CRM Leads → Contacts → Accounts → Deals → (deal-won automation conceptually) → Work Management Project → Dev Sprint tickets → Service for post-sale support.
- **Agency:** Work Management Client board → Project per client (Connect Boards) → Tasks board with sub-items → Timesheet (`time_tracking` column) → Dashboard with workload chart per assignee.
- **Product team:** Dev Roadmap → Epics → Sprints (`get_monday_dev_sprints_boards`) → Bugs board → Customer feedback loop from Service tickets via Connect Boards.
- **Marketing:** `marketing_campaigns` campaign board → content brief Docs → Content calendar (Calendar widget) → asset Files columns → cross-link to CRM accounts being targeted.
- **Service desk:** Service ticket board with SLA → Customer board → escalation to Dev bug board → KB Articles (`create_article`) for self-serve.

### Seed-data techniques
- For 50–500 items, drive `create_item` in a loop via `all_monday_api` multi-mutation documents (10–25 items per request to stay under complexity).
- For 500+ items, use `backfill_items` (≤20k rows, no side effects — perfect for demo seeding) over `ingest_items`. `ingest_items` triggers automations and is for production integrations; you don't want demo seeding to fire emails.
- People columns: pull real user IDs from `list_users_and_teams` and assign them across items so the People column shows avatars.
- Dates: spread across `today - 30d` to `today + 60d`. Mix in 1–2 overdue items so red/warning states render.
- Status: distribute 30/40/20/10 across `Working on it` / `Done` / `Stuck` / blank — don't put everything in one status.
- Numbers: draw from a realistic range for the domain (deal sizes $5k–$500k, story points 1/2/3/5/8/13, ticket priorities 1–4).

### Cloning and templates
- **Duplicating a doc:** `duplicate_doc`.
- **Duplicating a board:** `duplicate_board` mutation. `DuplicateBoardType` enum (verified): `duplicate_board_with_structure` (structure only), `duplicate_board_with_pulses` (structure + items), `duplicate_board_with_pulses_and_updates` (structure + items + updates).
- **Duplicating individual content:** `duplicate_item`, `duplicate_group`.
- **Object schemas:** `create_object_schema` + `connect_board_to_object_schema` lets you define a column structure once and apply it to multiple boards. Use this when building a series of similar demo boards (e.g., one per region). Schema lifecycle: `update_object_schema` (rename/update metadata), `delete_object_schema` (remove schema entirely — detaches all connected boards first). Additional column management: `create_object_schema_columns`, `update_object_schema_columns`, `set_object_schema_column_active_state` (deactivate/reactivate a column), `detach_boards_from_object_schema`, `bulk_object_schema_column_actions` (execute multiple column actions in one request — stops on first failure).
- **Managed columns:** `create_status_managed_column` + `attach_status_managed_column` on each board makes labels consistent across the demo so widgets aggregating across boards group cleanly.

### Demo tear-down / refresh
- Prefer `archive_board` / `archive_group` / `archive_item` (or generic `archive_object`) over `delete_*` so demos are recoverable. Note: workspaces have no archive — only `delete_workspace`.
- For a clean reset between sessions, use `delete_item` in a batch via `all_monday_api`, then re-seed — faster than rebuilding boards.
- Keep a `[DEMO] Master` workspace with golden-source boards. Spin up a fresh prospect demo by calling `duplicate_board` against those masters into a new `[DEMO] <Prospect>` workspace.

### Demo anti-patterns (in addition to §22)
- **Empty dashboards.** Always populate enough seed data that every widget is non-trivial.
- **Single-status items.** Distribute statuses across the spectrum.
- **Lorem ipsum / Item 1 / Test.** Use believable names — pull from a list of realistic company/contact names if needed.
- **One-product demos when multi-product is asked for.** If the prospect's pitch involves 2+ products, the demo should show cross-product flow via `board_relation` + mirrors.
- **No Docs.** A demo without a Doc looks like a half-finished build. Ship at least one per scenario.
- **No People assignments.** Empty People columns kill the visual. Spread users across items.
- **Same-day dates everywhere.** Spread dates so timeline/calendar widgets render variation.
- **Building in `Main workspace`.** Always create a dedicated `[DEMO] <Scenario>` workspace.

---

