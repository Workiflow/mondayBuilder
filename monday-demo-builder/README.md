# monday-demo-builder

A Claude plugin for building verified monday.com demo accounts from a discovery-call
transcript, live in your account via the monday MCP connector. Built for Workiflow's
pre-sales / solutions team, who spin up bespoke monday demos to support new-business deals.

It is a repackaging of Greg's [`monday-architect-skill`](https://github.com/greg375/monday-architect-skill)
as a multi-skill plugin, with the 1,200-line operator's manual refactored into a lean
decision-core plus on-demand reference files (progressive disclosure), and the demo-build
workflow lifted out of a pasted Project prompt into a first-class skill.

## What's inside

Three skills:

| Skill | What it does | Triggers on |
|---|---|---|
| **`monday-demo-builder`** | The 5-phase workflow: analyze a discovery transcript → propose a blueprint → **STOP for approval** → verify against the live account → build → hand off. | "build a monday demo", pasting a discovery/sales-call transcript, "spin up a demo for this prospect" |
| **`monday-architect`** | The operator's manual: forces correct product/architecture choices and holds every verified mutation, query, enum, column value-shape, widget, and gotcha. A ~260-line decision core plus 8 `references/` files loaded on demand. | any monday.com build/modify/reasoning task — boards, dashboards, CRM, sprints, portfolios, mirrors, columns, automations |
| **`refresh-monday-skill`** | Drift detector. Re-introspects the live monday API and reports/patches anything in `monday-architect` that has changed. | `/refresh-monday-skill`, before a high-stakes demo, after a monday API release, or on a monthly cadence |

```
monday-demo-builder/
├── .claude-plugin/plugin.json
├── skills/
│   ├── monday-demo-builder/
│   │   ├── SKILL.md
│   │   └── references/ (worked-example, demo-build-playbook)
│   ├── monday-architect/
│   │   ├── SKILL.md                  # decision core
│   │   └── references/ (8 files: products & native boards, columns &
│   │       relationships, items/forms/docs, dashboards & widgets,
│   │       automations & webhooks, products-deep, graphql/errors/gotchas,
│   │       marketplace-apps)
│   └── refresh-monday-skill/SKILL.md
├── CHANGELOG.md
├── LICENSE
└── README.md
```

## Setup

1. Install the plugin.
2. **Connect the monday.com connector** in Claude (Settings → Connectors). The skills act on
   your account through this connector — the plugin does not bundle it.
   - The skills reference monday tools by their base name (`get_user_context`, `create_board`, …).
     The exact MCP prefix depends on how your connector is registered, so nothing here is
     hardcoded to one prefix.
3. Use a monday.com account you have permission to build in. Some features (validation rules,
   certain bulk paths, owner-only mutations) are gated by plan/role.

## Usage

- **Build a demo:** open a chat with the plugin enabled and the monday connector connected,
  then paste a discovery-call transcript (or describe the prospect). `monday-demo-builder`
  runs the 5 phases and stops at Phase 3 for your approval before any mutation.
- **Ad-hoc monday work:** any monday build/modify request pulls in `monday-architect`
  automatically; it loads only the reference files the task needs.
- **Stay current:** run `/refresh-monday-skill` before high-stakes demos or monthly.

## Maintenance & versioning

`monday-architect` is a snapshot of the monday API as of its `version` date
(`2026-06-01-patch19`). monday ships fast, so run `refresh-monday-skill` periodically to
detect schema drift and patch the relevant file (SKILL.md or a `references/*.md`).
`CHANGELOG.md` carries the verified facts and gotchas captured at each release of the
original skill.

## Credit

Original `monday-architect` and `refresh-monday-skill` content by Greg
(<https://github.com/greg375/monday-architect-skill>), verified end-to-end against the live
monday MCP connector. Repackaged into this plugin by Workiflow. MIT licensed.
