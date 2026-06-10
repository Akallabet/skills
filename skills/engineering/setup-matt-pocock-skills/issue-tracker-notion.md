# Issue tracker: Notion

Issues and PRDs for this repo live as pages in a single Notion database. Use the Notion MCP tools (`mcp__claude_ai_Notion__*`) for all operations — no CLI is required.

## Database

The database URL and ID are recorded in `issue-tracker.json` at the repo root:

```json
{
  "type": "notion",
  "databaseUrl": "https://www.notion.so/<workspace>/<database-id>?v=...",
  "databaseId": "<32-char-hex-id>"
}
```

Skills should read `issue-tracker.json` to resolve the database ID before any Notion MCP call. This file is the single source of truth — if you move the database, update the JSON, not this doc.

The database is assumed to exist already. If it does not, create it manually in Notion before using these skills — this skill will not bootstrap it.

## Required schema

The database must expose the following properties. Property names are case-sensitive.

| Property     | Type             | Purpose                                                                                                                                       |
| ------------ | ---------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `Name`       | Title            | Issue title (Notion-mandatory).                                                                                                               |
| `Labels`     | Multi-select     | Triage roles — see `triage-labels.md`. Multiple values allowed. Skills may add new options when first used.                                   |
| `Type`       | Multi-select     | Kind of issue: `story`, `prd`, `bug` (extend as needed). `to-prd` writes `prd`; `to-issues` writes `story`/`bug`.                             |
| `Created by` | Created by       | Author of the page. Auto-managed by Notion. Read-only.                                                                                        |
| `Created at` | Created time     | Auto-managed by Notion. Read-only.                                                                                                            |
| `Updated at` | Last edited time | Auto-managed by Notion. Read-only.                                                                                                            |
| `GitHub PR`  | URL              | Optional link back to the implementing PR.                                                                                                    |
| `Parent`     | Relation (self)  | DUAL self-relation. Stories created from a PRD set this to the parent PRD page; the other side (`Children`) lists derived stories on the PRD. |
| `Children`   | Relation (self)  | Synced inverse of `Parent`. Read-only — set by writing `Parent` on the child.                                                                 |

The database may carry additional properties not listed here (e.g. `Status`, `Priority`, `Area`, `Steps to Reproduce`, `Expected behaviour`, `Actual Behavior`, `Reported By`, `Business Requirement`, `Relation`, `Supporting documents`). Skills should leave them untouched unless the user asks.

If the property names in your database differ, update this file — the skills will read the names from here.

## Conventions

All operations go through the Notion MCP tools. Always pass the database ID from the section above.

> **MCP tool shapes are NOT the Notion REST API shapes.** The MCP layer takes flattened parameters and SQL-like DDL for schema changes. Do not pass Notion-API-style objects (`{multi_select: {options: [...]}}`, `{rich_text: [...]}`) — they will be silently no-op'd or rejected. The exact shapes that work are listed below.

- **Create an issue / PRD** — `mcp__claude_ai_Notion__notion-create-pages` with `parent: { database_id: "<ID>" }` and a `pages: [{ properties, content }]` array. Inside `properties`, **multi-select values are a comma-separated string, not an array** (e.g. `"Labels": "ready-for-agent,needs-info"`, `"Type": "prd"`). Body goes in `content` as a Markdown string (not `children` block objects).
- **Read an issue** — `mcp__claude_ai_Notion__notion-fetch` with `id` (page ID or URL — the param name is `id`, not `urls`). Use `mcp__claude_ai_Notion__notion-get-comments` for the discussion thread.
- **List issues** — `mcp__claude_ai_Notion__notion-search` with `query` (free-text) plus `filter.property = "Labels"` / `filter.value = "<role>"` to scope by triage state. Or use a Notion view URL the user has saved.
- **Comment on an issue** — `mcp__claude_ai_Notion__notion-create-comment` with the page ID and `rich_text` body.
- **Apply / remove labels** — `mcp__claude_ai_Notion__notion-update-page` requires THREE top-level params: `page_id`, `command: "update_properties"`, and `properties: { "Labels": "ready-for-agent,needs-info" }` (comma-separated string, replaces the existing set — read first if you want to preserve others).
- **Close** — Notion has no "closed" state. Apply the `wontfix` label (or whatever the triage mapping says for terminal states) and stop updating the page. Optionally archive via `notion-update-page` with `command: "update_properties"` and `archived: true` if the user wants pages removed from default views.
- **Set parent (PRD → story)** — when publishing a story derived from a PRD, set the `Parent` relation on the child to the PRD's page URL: `properties: { "Parent": "https://app.notion.com/p/<prd-page-id>" }`. The relation column accepts a comma-separated string of page URLs (one URL per relation target). The inverse `Children` column on the PRD updates automatically via the DUAL synced relation — never write `Children` directly. Prefer this property over a Markdown link in the body; the body link is redundant when the relation is set.

### Adding new multi-select options (Labels, Type, etc.)

If you try to set a `Labels` / `Type` value the data source hasn't seen before, Notion returns `validation_error: Value must be one of the following: …`. You must add the option to the schema first.

Use `mcp__claude_ai_Notion__notion-update-data-source` with **SQL DDL statements**, NOT a `properties` array:

```
ALTER COLUMN "Labels" SET MULTI_SELECT(
  'needs-triage':gray,
  'needs-info':orange,
  'ready-for-agent':green,
  'ready-for-human':blue,
  'wontfix':default
)
```

Pass as `{ data_source_id: "collection://<uuid>", statements: "ALTER COLUMN \"Labels\" SET MULTI_SELECT(...)" }`. The `data_source_id` is the `collection://…` URL from `notion-fetch`'s `<data-source url="...">` tag, not the database ID. `ALTER COLUMN ... SET` replaces the full option set — include every option you want to keep.

**Important:** passing the Notion-API-shaped `properties: [{name, type, multi_select: {options: [...]}}]` to `notion-update-data-source` returns a success-looking response but silently no-ops. Always use the `statements` DDL string.

### Fallback order when publishing a labelled page

1. Try `notion-create-pages` with the desired `Labels` value.
2. If you get `validation_error … Value must be one of the following`, call `notion-update-data-source` with the full DDL to add the missing option(s).
3. Re-apply the label via `notion-update-page` (or recreate the page).

## When a skill says "publish to the issue tracker"

Create a Notion page in the database above with `Type` set to `story` or `bug` (or `prd` for `to-prd`).

## When a skill says "fetch the relevant ticket"

Call `notion-fetch` with the page ID, URL, or title the user provides. If only a title is given, run `notion-search` first to resolve it.
