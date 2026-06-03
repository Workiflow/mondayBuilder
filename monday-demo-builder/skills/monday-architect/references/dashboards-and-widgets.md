# Dashboards & widgets - verified catalog

> Verified reference for the `monday-architect` skill (progressive disclosure). Loaded on demand; the decision core lives in that skill's SKILL.md.

## 7. Dashboards and widgets — verified catalog

`all_widgets_schema` returns these widget types (verified):

| Widget type | Use for |
|---|---|
| `NUMBER` | Single KPI (sum/avg/median/min/max of a numbers column, or item count). Supports prefix/suffix, currency/percentage formatting. |
| `CHART` | **24 `graph_type` enum entries in live schema (≈12 unique chart types — each stacked variant is dual-spelled snake_case + camelCase):** `pie`, `donut`, `bar`, `column`, `line`, `smooth_line`, `area`, `bubbles`, `stackedBar`, `stackedColumn`, `stackedArea`, `stackedLine`, `stackedBarPercent`, `stackedColumnPercent`, `stackedAreaPercent`, `stackedLinePercent` (camelCase — verified working end-to-end) plus underscore aliases `stacked_bar`, `stacked_column`, `stacked_area`, `stacked_line`, `stacked_bar_percent`, `stacked_column_percent`, `stacked_area_percent`, `stacked_line_percent`. Both forms accepted. Stacked variants need `z_axis_columns`. Percent-stacked variants display proportions within each X group. Time-series charts (line/area + variants) want `x_axis_group_by: "date"` plus `group_by: "week"|"month"|...`. Bubbles wants 3 axes (x, y, z) and a numeric calc function. |
| `BATTERY` | Progress bar across boards/columns based on a "done" status label. Supports per-board status column lists, optional numeric weighting, group filtering. |
| `CALENDAR` | Date/timeline/week/creation_log/last_updated/lookup/formula columns rendered as calendar events. View modes month/week/day. Color-by board / group / parent / status / person / dropdown / board-relation / subtasks. |
| `GANTT` | Timeline/Gantt for timeline+date columns. Group/color/label by board/group/parent/color. |
| `LISTVIEW` | Standalone list view; configurable columns, item height, subitem display mode (like_items / nested / with_parents). |
| `APP_FEATURE` | Embed an app feature widget by `app_feature_id`. |

**Anything not in the above list** (e.g., Kanban, Workload, Numbers Grouping, Quote, Iframe) is NOT exposed by the connector's `create_widget`. Don't promise it. If the user asks, propose the closest supported widget — or build it as a **board view** (`create_view`) instead, since views support a wider set of layouts (kanban, etc.).

### ⚠️ `create_dashboard` produces an EMPTY CONTAINER — widgets are always a separate step

**Verified end-to-end May 2026.** `create_dashboard` returns a dashboard ID with zero widgets. It is not usable until you call `create_widget` for every widget you want on it. An empty dashboard shown in a demo is worse than no dashboard. This is a two-step pattern, always:

1. `create_dashboard` (in a workspace, optionally a folder; `DashboardKind` is `PUBLIC` or `PRIVATE`). Save the returned `dashboard_id`.
2. Call `all_widgets_schema` to get the full JSON Schema 7 for every widget type you intend to add.
3. For **each** widget: call `create_widget` with `parent_container_id: <dashboard_id>`, `parent_container_type: "DASHBOARD"`, `widget_kind`, `widget_name`, and `settings` conforming to the schema. Boards referenced in widget settings must already exist and have data — widgets sourced from empty boards render as blank.
4. Verify widgets are visible in the UI before calling the dashboard complete.

**Dashboard mutations:** `update_dashboard(id: ID!, ...)` — update dashboard metadata (name, description, visibility). `update_overview_hierarchy` — reorder/rearrange items in the overview/hierarchy view of a dashboard. `delete_dashboard(id: ID!)` — delete a dashboard permanently.

**There is no `list_widgets` query, but `delete_widget(id: ID!): Boolean` IS available as a mutation.** Capture the widget ID from the `create_widget` response and stash it — there is no API to enumerate existing widgets on a dashboard, so without that ID the only way to remove a widget is the dashboard UI. `delete_widget` returns `true` on success; subsequent calls against the same (now-gone) ID return a generic server error rather than a structured "not found", so the response is not idempotent — guard against double-deletes in your own code.

For board-level analytics without building a dashboard, use `board_insights`.

---

