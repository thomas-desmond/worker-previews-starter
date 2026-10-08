# Agent instructions

This repo is a Cloudflare Worker with a D1 database. Every branch you push gets its own [Worker Preview](https://developers.cloudflare.com/workers/previews/): an isolated copy of the Worker with its own URL, bindings, and logs. Workers Builds posts the Preview URL on the branch's pull request. Use the Preview to check your own work before a human reviews it.

## Rules

- **Production settings live at the top level of `wrangler.json`. Preview settings live in its `previews` block.** Previews don't inherit production vars or bindings. When a Preview needs a binding, add it under `previews`. Never point a `previews` binding at a production resource.
- **Never write to production data.** Don't run `wrangler d1 execute`, `d1 migrations apply`, or any other write against a production database. Production is the `database_name` in the top-level `d1_databases` (`activity-log-db` by default).
- **Previews and production never share a schema path.** `migrations/` is production's, applied automatically on deploy. `preview-migrations/` is the Preview database's, applied only through `wrangler.preview-migrations.jsonc`. Never copy a file from one folder to the other.
- **Don't edit a database schema to make a bug go away.** Fix the code. It has to work with production's schema and the Preview's.
- **Never merge a pull request, deploy to production, or run `wrangler deploy` unless a human explicitly tells you to.** Pushing to a PR branch is fine: it only updates that branch's Preview. `npm run check` runs `wrangler deploy --dry-run`, which doesn't deploy anything, so it's fine.

## Finding the Preview URL

Workers Builds builds every push to a PR branch and comments on the PR with the **Preview URL** (one per branch, always the latest push) and a **deployment URL** for each commit.

1. Wait for the build: `gh pr checks --watch` (the check is named `Workers Builds: <worker-name>`).
2. Read the bot comment: `gh pr view --comments`. The Preview URL looks like `https://<branch>-<worker-name>.<subdomain>.workers.dev`.
3. The comment's deployment table lists the commit for each deployment. If your latest commit isn't in it yet, the Preview is still serving the old code.

## Changing the Preview database's schema

The Preview D1 database is shared by every Preview of this Worker. Its migrations live in `preview-migrations/` and are applied with a separate config file, `wrangler.preview-migrations.jsonc`. That file declares the Preview database under the binding `PREVIEW_DB`, so the command can't reach production's `DB`.

1. Check that `database_name` and `database_id` in `wrangler.preview-migrations.jsonc` match `previews.d1_databases` in `wrangler.json`. If they don't, stop and tell the human.
2. Apply: `npx wrangler d1 migrations apply PREVIEW_DB --remote --config wrangler.preview-migrations.jsonc`.
3. D1 records applied migrations, so running it again only applies new files. Because the database is shared, a migration applied for one branch shows up in every Preview.

## Testing a Preview

Test against the Preview URL, not `localhost` and not production. The Preview has its own database, so you can add and delete data there freely.

The app's JSON API:

| Method | Path | Does |
| --- | --- | --- |
| `GET` | `/api/entries` | List entries. Each has `id`, `text`, `created_at`. |
| `POST` | `/api/entries` | Add an entry. Body: `{"text": "..."}` |
| `DELETE` | `/api/entries/<id>` | Delete the entry with that `id`. |

A full check covers all three: add an entry, list it, delete it, and list again to confirm it's gone. Report the status codes and response bodies. A `409` means the Preview database is bound but its migrations haven't been applied. An HTML page saying no database is bound means the `previews` block has no D1 binding.

## Reading Preview logs

When a request fails, read that Preview's logs instead of guessing. The Worker logs structured events like `activity_log.delete_failed` that include the underlying error.

Use the **Cloudflare Workers Observability MCP server** (`cloudflare-observability`, configured in this repo for common agents). It's read-only.

- Filter on all three fields:
  - `$workers.scriptName` = the Worker `name` in `wrangler.json`
  - `$workers.preview.slug` = the Preview's name: the branch name, as it appears at the start of the Preview URL's hostname
  - `$metadata.level` = `error`, or `event` = the event name
- Use `view: "events"` and a timeframe covering the last hour. Logs can take a minute to appear.
- If `query_worker_observability` errors while parsing the events response, read the fields you need (for example `error` and `entryId`) with `observability_values`, using the same filters.
- If the tools ask for an `account_id`, use the account the Worker is deployed to (`npx wrangler whoami` lists them).
- Production logs never include Preview traffic. If you don't filter on `$workers.preview.slug`, you may be reading the wrong environment.

If the MCP tools aren't available, say so and ask the human to open the Worker in the Cloudflare dashboard, switch the environment dropdown from **Production** to the branch, and open **Observability**.

## Before you say you're done

1. Push, then wait for the new deployment to show your commit.
2. Re-run the full API check against the Preview URL.
3. Give the human the Preview URL, the results, and what you changed, then let them decide whether to merge.
