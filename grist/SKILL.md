---
name: grist
description: Interact with a self-hosted Grist instance via its REST API. Use this skill whenever the user wants to read or modify data in Grist — listing orgs/workspaces/docs, finding tables and columns, querying or filtering records, adding/updating/deleting rows, running read-only SQL against a doc, or managing table schema. Trigger on requests like "what's in my Grist doc", "add a row to the Inventory table", "update the status column for record 12", "run a query against my budget doc", "what tables are in this workspace", or any mention of Grist docs, tables, or records.
---

# Grist API Skill

Grist is a spreadsheet-database hybrid. Each **doc** contains **tables**, each table has **columns** and **records** (rows). This skill covers reading and writing that data through Grist's REST API.

## Configuration

Two environment variables must be present:

| Variable | Example |
|----------|---------|
| `GRIST_BASE_URL` | `https://grist.hummel.casa` |
| `GRIST_API_KEY` | Personal API key, from Profile Settings in the Grist UI |

All requests need:
```
Authorization: Bearer $GRIST_API_KEY
Accept: application/json
Content-Type: application/json   (for POST/PATCH/PUT)
```

The API root is `$GRIST_BASE_URL/api`.

## Finding your way to a table

Grist nests resources: **org → workspace → doc → table**. Most day-to-day work only needs a `docId` and `tableId`, but if the user hasn't given you one, resolve it top-down:

```bash
# 1. List orgs (usually just one on a self-hosted single-tenant instance)
curl -s -H "Authorization: Bearer $GRIST_API_KEY" \
  "$GRIST_BASE_URL/api/orgs" | jq .

# 2. List workspaces and docs within an org
curl -s -H "Authorization: Bearer $GRIST_API_KEY" \
  "$GRIST_BASE_URL/api/orgs/$ORG_ID/workspaces" | jq .

# 3. List tables in a doc
curl -s -H "Authorization: Bearer $GRIST_API_KEY" \
  "$GRIST_BASE_URL/api/docs/$DOC_ID/tables" | jq .

# 4. List columns in a table (useful to see field names/types before writing)
curl -s -H "Authorization: Bearer $GRIST_API_KEY" \
  "$GRIST_BASE_URL/api/docs/$DOC_ID/tables/$TABLE_ID/columns" | jq .
```

Doc and workspace IDs are opaque strings/numbers (e.g. `9PJhBDZPyCNoayZxaCwFfS`) — list them rather than guessing. Table IDs and column IDs are the human-readable names shown in Grist (e.g. `Inventory`, `Status`), matching the table/column titles with spaces removed.

Once you know a `docId` and `tableId` for the session, reuse them rather than re-resolving every call.

## Reading records

```bash
curl -s -H "Authorization: Bearer $GRIST_API_KEY" \
  "$GRIST_BASE_URL/api/docs/$DOC_ID/tables/$TABLE_ID/records" | jq .
```

Records come back as `{"records": [{"id": 1, "fields": {"ColName": value, ...}}, ...]}`.

Useful query parameters:
- `filter` — JSON object mapping column name to an array of allowed values, URL-encoded. E.g. filter `Status` to `Open` or `Blocked`:
  ```bash
  curl -s -H "Authorization: Bearer $GRIST_API_KEY" \
    --get --data-urlencode 'filter={"Status": ["Open", "Blocked"]}' \
    "$GRIST_BASE_URL/api/docs/$DOC_ID/tables/$TABLE_ID/records"
  ```
- `sort` — comma-separated column names, prefix with `-` for descending (e.g. `sort=-CreatedAt`).
- `limit` — cap the number of rows returned (0 = no limit).

Use `--get --data-urlencode` (as above) rather than hand-building the query string — it handles the JSON-in-URL encoding correctly.

## Adding records

```bash
curl -s -X POST \
  -H "Authorization: Bearer $GRIST_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"records": [{"fields": {"Name": "Widget", "Quantity": 42}}]}' \
  "$GRIST_BASE_URL/api/docs/$DOC_ID/tables/$TABLE_ID/records" | jq .
```
Add multiple rows in one call by including more objects in the `records` array. Returns the new row `id`s.

## Updating records

Requires the record `id` (from a prior read):
```bash
curl -s -X PATCH \
  -H "Authorization: Bearer $GRIST_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"records": [{"id": 12, "fields": {"Status": "Done"}}]}' \
  "$GRIST_BASE_URL/api/docs/$DOC_ID/tables/$TABLE_ID/records"
```
Only include the fields you want to change — others are left untouched.

## Add-or-update in one call (upsert)

When you know a record should exist with certain key values but don't know its `id`, use `PUT` with `require` instead of `id`. Grist looks for a matching row and updates it, or creates a new one if none matches:
```bash
curl -s -X PUT \
  -H "Authorization: Bearer $GRIST_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"records": [{"require": {"SKU": "WID-1"}, "fields": {"Quantity": 42}}]}' \
  "$GRIST_BASE_URL/api/docs/$DOC_ID/tables/$TABLE_ID/records"
```

## Deleting records

```bash
curl -s -X POST \
  -H "Authorization: Bearer $GRIST_API_KEY" \
  -H "Content-Type: application/json" \
  -d '[12, 13]' \
  "$GRIST_BASE_URL/api/docs/$DOC_ID/tables/$TABLE_ID/records/delete"
```
The body is a plain JSON array of row IDs (not wrapped in an object). Permanent — confirm with the user before deleting unless they've clearly already decided.

## Ad hoc queries with SQL

For reporting or filters too complex for the `filter`/`sort` params (joins across tables, aggregates, `GROUP BY`), run read-only SQL directly against the doc's underlying SQLite database. Table and column names in SQL are the same IDs used elsewhere.

```bash
curl -s -X POST \
  -H "Authorization: Bearer $GRIST_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"sql": "select Status, count(*) as n from Inventory group by Status"}' \
  "$GRIST_BASE_URL/api/docs/$DOC_ID/sql" | jq .
```
Only `SELECT` (optionally with `WITH`) is allowed — no inserts/updates/schema changes. Use `args` for parameters instead of string-interpolating values:
```bash
-d '{"sql": "select * from Inventory where Quantity < ?", "args": [10]}'
```
For a quick one-off without a JSON body, the GET variant works for simple queries:
```bash
curl -s -H "Authorization: Bearer $GRIST_API_KEY" \
  --get --data-urlencode "q=select * from Inventory limit 5" \
  "$GRIST_BASE_URL/api/docs/$DOC_ID/sql"
```

## Managing columns and tables

```bash
# Add a column
curl -s -X POST \
  -H "Authorization: Bearer $GRIST_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"columns": [{"id": "Notes", "fields": {"label": "Notes", "type": "Text"}}]}' \
  "$GRIST_BASE_URL/api/docs/$DOC_ID/tables/$TABLE_ID/columns"

# Add a table
curl -s -X POST \
  -H "Authorization: Bearer $GRIST_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"tables": [{"id": "Tasks", "columns": [{"id": "Title", "fields": {"type": "Text"}}]}]}' \
  "$GRIST_BASE_URL/api/docs/$DOC_ID/tables"
```
Schema changes (new tables/columns) are less common than data operations — confirm the intent with the user before creating structure, since it's easy to end up with duplicate or oddly-typed columns if the request was ambiguous.

## Error handling

| Status | Meaning |
|--------|---------|
| 401 | Bad or missing API key |
| 403 | Insufficient permissions on that org/workspace/doc |
| 404 | Doc, table, or record not found (or no access) |
| 400 | Malformed request — check the JSON body shape matches the endpoint (e.g. records need `id`/`require` for PATCH/PUT but not for POST) |

## Tips

- Column values (`fields`) are keyed by column **ID**, not by the display label shown in the Grist UI — list columns first if unsure (`.../columns`), since labels and IDs can differ once a column has been renamed.
- Choice and Reference column types store values as plain strings/numbers over the API, but Date/DateTime columns store Unix timestamps (seconds) — check a sample record's existing value before writing to a date column, or check the column's `type` from the columns endpoint.
- `PATCH` and `PUT` behave differently: `PATCH` requires you to already know the row `id`; `PUT` matches on `require` and will create a row if nothing matches. Prefer `PUT`/`require` when you don't already have an `id` in hand — it avoids a separate read-then-decide step.
- Deleting is permanent (no undo via the API) — double-check row IDs against a fresh read before calling `/records/delete`.
- SQL queries are read-only and time out after 1 second by default; pass `"timeout"` (ms) in the POST body for slower aggregate queries over large tables.
