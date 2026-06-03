# Automations, integrations, webhooks, audit & compliance

> Verified reference for the `monday-architect` skill (progressive disclosure). Loaded on demand; the decision core lives in that skill's SKILL.md.

## 11. Automations, triggers, integrations, webhooks (advanced)

- **Webhooks**: first-class mutations `create_webhook` / `delete_webhook`; query existing with `webhooks`. `WebhookEventType` (verified, 21 events): `change_column_value`, `change_specific_column_value`, `change_status_column_value`, `change_name`, `change_subitem_column_value`, `change_subitem_name`, `create_item`, `create_subitem`, `create_column`, `create_update`, `create_subitem_update`, `edit_update`, `delete_update`, `item_archived`, `item_deleted`, `item_restored`, `item_moved_to_any_group`, `item_moved_to_specific_group`, `move_subitem`, `subitem_archived`, `subitem_deleted`. Use webhooks to drive external automations.
  - **Challenge handshake (must implement on the receiver):** on subscription monday POSTs `{"challenge": "<token>"}` to your URL; the endpoint MUST echo the same body back or the subscription fails. HTTPS required.
  - **Several events take a `config` arg** (JSON string): `change_specific_column_value` → `config: "{\"columnId\":\"<id>\"}"`; `change_status_column_value` (fire only when a status hits an index) → `config: "{\"columnId\":\"<id>\",\"columnValue\":{\"index\":<n>}}"`. Without `config` these events fire on all columns.
  - **Payloads reference items as `pulseId`, not `itemId`** (legacy "pulse" naming) — a classic parsing gotcha. Webhooks created via an integration-app token also carry a JWT `Authorization` header with a `shortLivedToken` for calling the API back; verify it with your app Signing Secret. Retries run ~1×/min for ~30 min; respond `200` fast. *(Handshake/JWT details are doc-sourced — verify against the live webhooks reference.)*
- **Integration blocks**: `execute_integration_block` mutation — run an integration block with provided field values.
- **Trigger / automation analytics**: `trigger_events`, `trigger_event`, `block_events`, `tool_events`, `account_trigger_statistics`, `account_triggers_statistics_by_entity_id` (raw GraphQL) — diagnose automation runs.
- **Validations**: `validations` query + mutations `create_validation_rule` / `update_validation_rule` / `delete_validation_rule` — board-level data validation rules.
- **Connections**: `connections`, `user_connections`, `account_connections`, `connection`, `connection_board_ids` — reusable auth connections (e.g., for sequences/integrations).

If the user asks "automate X when Y happens", first decide:
1. Is there a built-in automation recipe in the UI? (Most cases — recommend the user wires it there; the API doesn't currently expose recipe creation directly.)
2. If external system needs to react → **webhook** subscribing to the right `WebhookEventType`.
3. If a one-shot integration action → `execute_integration_block`.

---

## 12. Audit logs and compliance

- `audit_event_catalogue` (raw GraphQL) — list all audit event types.
- `audit_logs` (raw GraphQL) — query account audit log with filters (user_id, events, ip_address, time range, paginated).
- `export_events` — export board events for a date range (requires `X-Tool-Execution-Secret` header).

---

