# Worker Previews Starter

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/thomas-desmond/worker-previews-starter)

<!-- dash-content-start -->

The starter app for the Worker Previews workshop. It's a small Activity Log: a Worker with a D1 database and a UI for adding and deleting entries. The page shows a badge telling you whether you're on Production or a Preview.

In the workshop you give your branch its own Preview with a separate D1 database, test a new schema there without touching production, track down a bug the new schema causes (it's deliberate) using that Preview's Observability logs, and merge a fix.

<!-- dash-content-end -->

<!-- TODO: link the workshop guide URL here once it has a permanent home. -->

## Learn more

- [Worker Previews announcement](https://blog.cloudflare.com/worker-previews/)
- [Worker Previews docs](https://developers.cloudflare.com/workers/previews/)
- [D1 docs](https://developers.cloudflare.com/d1/)

## What's in the repo

| Path | Purpose |
| --- | --- |
| `src/` | The Worker and its HTML UI |
| `migrations/` | Production schema and seed data, applied on deploy |
| `preview-migrations/` | The Preview database's schema, seed rows, and a candidate schema change. Applied only through `wrangler.preview-migrations.jsonc` (created in the workshop), so it never reaches production. |
| `wrangler.json` | Production settings at the top level; Preview settings in the `previews` block |
| `AGENTS.md` | Instructions for your coding agent: keep Preview settings in `previews`, never write to production data, test on the Preview URL, read the Preview's logs. `CLAUDE.md` points Claude Code at it. Copy it into your own repos. |
| `.mcp.json`, `.cursor/`, `.vscode/`, `.codex/`, `opencode.json` | Connect Claude Code, Cursor, VS Code (Copilot), Codex, and OpenCode to the read-only [Workers Observability MCP server](https://github.com/cloudflare/mcp-server-cloudflare/tree/main/apps/workers-observability), so your agent can read your Preview's logs itself. Sign in with Cloudflare the first time your agent uses it. |

## Getting started

Click **Deploy to Cloudflare** above. It creates a copy of this repo in your GitHub account, provisions a production D1 database, connects Workers Builds, and deploys. Keep the default names in the deploy form so the workshop steps match.

## Manual setup (without the button)

1. Install dependencies:
   ```bash
   npm install
   ```
2. Create a [D1 database](https://developers.cloudflare.com/d1/get-started/) and put its ID in the `database_id` field of `wrangler.json`:
   ```bash
   npx wrangler d1 create activity-log-db
   ```
3. Deploy. The `predeploy` script applies `migrations/` to the production database first:
   ```bash
   npm run deploy
   ```
