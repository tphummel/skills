---
name: bumpkit-counter
description: Create, increment (bump), and read counters using the Bumpkit API at bumpkit.tphummel.workers.dev. Use when you need a persistent named counter that can be incremented and queried by ID.
---

# Bumpkit Counter

Manage persistent named counters (called "bumpers") via the Bumpkit API.

**Base URL**: `https://bumpkit.tphummel.workers.dev`

A bumper is identified by a string `id` you choose. Bumping auto-creates the bumper if it doesn't exist yet — there's no separate create step.

---

## Operations

### Read a counter

```bash
curl https://bumpkit.tphummel.workers.dev/bumper/{id}.json
```

Returns 404 if the bumper doesn't exist yet.

Response:
```json
{
  "id": "my-counter",
  "count": 42,
  "created_at": "2024-01-01T00:00:00Z",
  "updated_at": "2024-01-02T00:00:00Z",
  "expires_at": null
}
```

### Bump (increment) a counter

Creates the bumper if it doesn't exist, then increments count by 1.

```bash
curl -X POST https://bumpkit.tphummel.workers.dev/bumper/{id}/bump.json
# or GET also works:
curl https://bumpkit.tphummel.workers.dev/bumper/{id}/bump.json
```

Returns the updated bumper JSON (same schema as read).

### Delete a counter

```bash
curl -X DELETE https://bumpkit.tphummel.workers.dev/bumper/{id}
```

Returns 200 on success, 404 if not found.

### Read metadata only (no body)

```bash
curl -I https://bumpkit.tphummel.workers.dev/bumper/{id}
```

Useful response headers: `bumper-id`, `bumper-count`, `bumper-created-at`, `bumper-updated-at`, `bumper-expires-at`.

---

## Response Formats

Append an extension to any path to control the format:

| Extension | Content-Type        |
|-----------|---------------------|
| `.json`   | `application/json`  |
| `.txt`    | `text/plain`        |
| `.xml`    | `application/xml`   |
| `.p8s`    | Prometheus metrics  |
| _(none)_  | `text/html`         |

---

## No Authentication Required

The Bumpkit API is open — no API key or token needed.

---

## Examples

```bash
# Bump a deploy counter and show the new count
curl -s -X POST https://bumpkit.tphummel.workers.dev/bumper/my-deploys/bump.json \
  | jq .count

# Read current value of a counter
curl -s https://bumpkit.tphummel.workers.dev/bumper/my-deploys.json \
  | jq '{id, count}'

# Get count as plain text
curl -s https://bumpkit.tphummel.workers.dev/bumper/my-deploys.txt

# Delete a counter
curl -X DELETE https://bumpkit.tphummel.workers.dev/bumper/my-deploys
```

---

## Validation

After bumping or reading, confirm the response contains:
- [ ] `id` matches the requested counter name
- [ ] `count` is a non-negative integer
- [ ] HTTP status is 200
