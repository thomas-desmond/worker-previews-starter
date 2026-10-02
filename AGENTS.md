# Agent instructions

This repo is a Cloudflare Worker with a D1 database. Every branch with an open pull request gets its own [Worker Preview](https://developers.cloudflare.com/workers/previews/): an isolated copy of the Worker with its own URL, bindings, and logs. Use the Preview to check your own work before a human reviews it.

## Rules

- **Production settings live at the top level of `wrangler.json`. Preview settings live in its `previews` block.** Previews don't inherit production vars or bindings. When a Preview needs a binding, add it under `previews`. Never point a `previews` binding at a production resource.
- **Never write to production data.** Don't run `wrangler d1 execute`, `d1 migrations apply`, or any other write against a production database. Production is the `database_name` in the top-level `d1_databases` (`activity-log-db` by default). Files in `workshop/` are for the Preview database only.
- **Don't edit a database schema to make a bug go away.** Fix the code. It has to work with production's schema and the Preview's.
- **Never merge a pull request, deploy to production, or run `wrangler deploy` unless a human explicitly tells you to.** Pushing to a PR branch is fine: it only updates that branch's Preview.

## Finding the Preview URL

Workers Builds builds every push to a PR branch and comments on the PR with the **Preview URL** (one per branch, always the latest push) and a **deployment URL** for each commit.

1. Wait for the build: `gh pr checks --watch` (the check is named `Workers Builds: <worker-name>`).
2. Read the bot comment: `gh pr view --comments`. The Preview URL looks like `https://<branch>-<worker-name>.<subdomain>.workers.dev`.
3. The comment's deployment table lists the commit for each deployment. If your latest commit isn't in it yet, the Preview is still serving the old code.

## Testing a Preview

Test against the Preview URL, not `localhost` and not production. The Preview has its own database, so you can add and delete data there freely.

The app's JSON API:

| Method | Path | Does |
| --- | --- | --- |
| `GET` | `/api/entries` | List entries. Each has `id`, `text`, `created_at`. |
| `POST` | `/api/entries` | Add an entry. Body: `{"text": "..."}` |
| `DELETE` | `/api/entries/<id>` | Delete the entry with that `id`. |

A full check covers all three: add an entry, list it, delete it, and list again to confirm it's gone. Report the status codes and response bodies. A `409` means the Preview database is bound but has no schema yet. An HTML page saying no database is bound means the `previews` block has no D1 binding.

## Reading Preview logs

When a request fails, read that Preview's logs instead of guessing. The Worker logs structured events like `activity_log.delete_failed` that include the underlying error.

Use the **Cloudflare Workers Observability MCP server** (`cloudflare-observability`, configured in this repo for common agents). It's read-only.

- Filter on all three fields:
  - `$workers.scriptName` = the Worker `name` in `wrangler.json`
  - `$workers.preview.slug` = the Preview's name: the branch name, as it appears at the start of the Preview URL's hostname
  - `$metadata.level` = `error`, or `event` = the event name
- Use `view: "events"` and a timeframe covering the last hour. Logs can take a minute to appear.
- If the tools ask for an `account_id`, use the account the Worker is deployed to (`npx wrangler whoami` lists them).
- Production logs never include Preview traffic. If you don't filter on `$workers.preview.slug`, you may be reading the wrong environment.

If the MCP tools aren't available, say so and ask the human to open the Worker in the Cloudflare dashboard, switch the environment dropdown from **Production** to the branch, and open **Observability**.

## Before you say you're done

1. Push, then wait for the new deployment to show your commit.
2. Re-run the full API check against the Preview URL.
3. Give the human the Preview URL, the results, and what you changed, then let them decide whether to merge.
