# Building monday.com marketplace apps (SDK / iframe)

> Verified reference for the `monday-architect` skill (progressive disclosure). Loaded on demand; the decision core lives in that skill's SKILL.md.

## 28. Building monday.com marketplace apps (SDK / iframe apps)

This section covers building **apps installed inside monday.com** — not API automation scripts that call monday from outside, but React apps embedded as board views, item views, or dashboard widgets via the monday.com Apps Framework.

### App types and where they live
- **Board view app** — renders as a full-screen view tab on a board. Gets the host board's ID from context.
- **Item view app** — renders inside the item detail panel.
- **Dashboard widget** — renders inside a dashboard.
- **Account-level apps** — run independently of a specific board.

### The SDK vs the API
The `monday-sdk-js` npm package is the bridge between your iframe app and the monday.com parent shell:
- `monday.listen('context', cb)` — the primary hook. Fires on load and whenever the user changes context. `res.data` includes: `boardIds`, `itemId`, `user.isAdmin`, `account.slug`, `theme`, `locale`.
- `monday.api(query, { variables })` — runs GraphQL against the monday API. Works identically to the REST/HTTP API from outside, but is pre-authenticated via the SDK. No token management needed.
- `monday.storage.instance.getItem(key)` / `monday.storage.setItem(key, value)` — **instance storage** is per-user per-install. `monday.storage.getItem(key)` (no `.instance`) is **shared storage** — same value for all users/installs.
- `monday.storage` key/value strings are capped at **~6KB per key**. For larger configs, split across multiple keys or use a dedicated overflow key pattern.
- `monday.execute('openItemCard', { itemId })` — opens the item card. Similar `execute` calls for other SDK actions.
- `monday.setApiVersion("2024-10")` — pin the API version at SDK init. Always set this.

### Context-driven board ID vs user-selected board ID
A common pattern is apps that let users switch which board they're targeting. This is architecturally tricky:
- `monday.listen('context', ...)` provides the **host board ID** — the board the app is installed on. This is the config/storage anchor.
- If the user selects a *different* board to work with (e.g., a routing destination), store that separately. **Never overwrite the config-board ID with a user selection**, or storage reads/writes will target the wrong board.
- Use a `useRef` guard (e.g., `userSelectedBoardRef.current = true`) to prevent the context listener from overriding a manual board selection after initial load.
- When switching the active board for display, reload the board schema + form config but **carry forward** any metadata (allowed board IDs, routing rules) that lives on the host/config board.

### Storage architecture for per-board config
- Key your config by board ID: `config_${boardId}`. This lets an app support multiple boards simultaneously.
- Always use `monday.storage.setItem` (shared, not instance) so config is visible to all users of the app — not just the admin who set it.
- Keep an **overflow key** for large arrays (e.g., `allowed_boards_${boardId}`) so one large field doesn't push the main config over the 6KB limit. Read both on load; write both on save. If the main config is rejected, the overflow key may still succeed, giving you partial recovery.
- Always write `JSON.stringify(value)` and `JSON.parse(value)` — storage is always string.
- When loading config, check: shared key → fallback to legacy instance key → fallback to computed defaults. Migrate from instance to shared on first load (one-time write).

### Board schema loading
- On every board switch, fetch schema via `boards(ids: [$id]) { columns { id title type settings_str } }`. Filter out `id` and `name` columns (read-only system columns) before presenting to users.
- Add a 8-second fallback timer: if schema/config fetches hang (e.g. network, rate limit), force the UI into a degraded-but-usable state.
- Cache fetched board lists in `sessionStorage` (not localStorage) with a TTL (5 minutes works well). This avoids re-fetching the full board list on every panel open. Serve from cache on repeat opens; fall through to live API only on miss/expiry.
- When fetching all boards (for a picker), use `limit: 500, page: N` pagination rather than `limit: 50, page: N * 40`. Fewer round trips, same result. Emit progressive results via an `onProgress` callback so the UI is populated after page 1, not after all pages complete.

### Board search picker UX patterns
- For accounts with many boards, always implement workspace-based filtering (chip tabs) in addition to text search. Group boards by `workspace.id/name`.
- When the user has already typed a query and data is in local cache, skip the debounce entirely (0ms delay). Only debounce when the lookup will hit the live API.
- Show the currently-active board as highlighted (`aria-selected`) in the dropdown for orientation.

### Config size management (6KB limit)
- The monday storage 6KB cap per key is a real constraint for apps with rich configs (routing rules, field mappings, allowed board lists). Strategies:
  1. **Split large fields to a dedicated overflow key.** `allowed_boards_{boardId}` stored separately from the main config blob.
  2. **Warn when approaching the limit.** At ~5,500 bytes, show a toast warning the admin.
  3. **Always save the overflow key atomically alongside the main config** so they stay in sync. Use `Promise.all([setMain, setOverflow])`.
  4. On load, if `allowedBoardIds` in the main config is empty, try the dedicated overflow key as a recovery path.

### Routing and conditional submission
- When an app routes submissions to different destination boards based on form values:
  - Resolve the routing at submit time, not at form-load time.
  - Fetch the destination board's schema lazily — pre-fetch schemas for all possible destinations in the background after initial load completes.
  - After creating an item on a destination board, **always verify it exists** before showing success: `items(ids: [$newItemId]) { id }` with exponential backoff (300ms × 2^attempt, up to 5 attempts). monday.com item creation is eventually consistent and new items may not be immediately queryable.
  - For cross-board submissions, run a column-mapping pass: prefer exact column ID match → title-match (case-insensitive) → type-compatible match. Allow the admin to configure explicit overrides.

### Connected item creation (board_relation)
- When a form field is a `board_relation` and the user wants to *create* a new connected item inline (not select existing):
  1. Create the connected item first on the linked board.
  2. Then create the main item with `{"item_ids": [<new connected item id>]}` in the relation column.
  3. If the connected item itself has `board_relation` fields, recurse — this is a tree problem. Resolve depth-first.
- The `settings_str` of a `board_relation` column contains `boardIds` — parse this (JSON.parse) to find which board it connects to.

### File uploads
- File uploads via `add_file_to_column` must happen **after** item creation — you can't attach a file during `create_item`.
- Show progress UI (10% → 40% → 70% → 100%) using staged `sleep()` calls while the upload request is in-flight. Users expect visual feedback.
- Handle upload failures gracefully: the item was already created. Show a "retry upload" option on the thank-you screen rather than treating it as a full submission failure.

### Draft persistence
- Auto-save form drafts to `monday.storage.instance` (per-user) keyed by board ID. Save on every field change (debounced). Restore on next open if a draft exists.
- Skip draft restore if the URL has prefill params — URL params take priority.
- Discard draft after successful submission.

### Edit mode / admin panel
- Use an `isEditMode` boolean gate to switch the same form between "user filling it" and "admin configuring it" views. This avoids maintaining two separate UIs.
- In edit mode, field clicks should select the field and open a property inspector sidebar — never trigger validation or submission.
- Show a dirty/saved indicator in the nav bar. Auto-persist simple toggle changes (wizard mode, etc.) immediately; batch rich changes (routing rules, field order) behind an explicit Save button.
- Guard navigation away from unsaved changes with `window.confirm` (or a toast + lock pattern).

### API pagination in apps
- Always paginate `items_page` with a cursor: first call without cursor, subsequent calls with the returned cursor, stop when cursor is null.
- For the board list fetch on app init, use `limit: 500, page: N` — 10 pages covers 5,000 boards. Most accounts have fewer than 1,000 boards.

### Performance patterns
- Fetch users (`users { id name photo_thumb }`) once at startup and cache in state — these rarely change mid-session.
- Fetch all boards once at startup with progressive streaming (see board search picker section above).
- Fetch destination board schemas lazily as routing rules reference new board IDs, not all at once on mount.
- Re-use `sessionStorage` for the boards list across panel opens within the same browser session (5-min TTL).

---

