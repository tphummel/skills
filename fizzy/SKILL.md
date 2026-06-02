---
name: fizzy
description: Interact with the Fizzy kanban tool via its REST API. Use this skill whenever the user wants to read or modify anything in Fizzy — listing boards, finding or creating cards, moving cards between columns, closing cards, adding comments, checking activity, managing tags, or any other Fizzy operation. Trigger on requests like "show me my Fizzy boards", "create a card for X", "what's in the backlog?", "move card #42 to Done", "add a comment to card #7", "what cards are unassigned?", or any mention of Fizzy boards, cards, columns, or kanban work.
---

# Fizzy API Skill

Fizzy is a kanban-style project management tool. This skill covers everything needed to interact with it programmatically.

## Configuration

Two environment variables must be present:

| Variable | Example |
|----------|---------|
| `FIZZY_BASE_URL` | `https://fizzy.example.com` |
| `FIZZY_API_KEY` | Personal access token with Read+Write permission |

All requests need these headers:
```
Authorization: Bearer $FIZZY_API_KEY
Accept: application/json
Content-Type: application/json   (for POST/PUT/PATCH)
```

## Account slug

Every resource URL is scoped to an **account slug** — a path segment like `/897362094` that identifies the account. Resolve it once per session:

```bash
curl -s -H "Authorization: Bearer $FIZZY_API_KEY" -H "Accept: application/json" \
  "$FIZZY_BASE_URL/my/identity" | jq -r '.accounts[0].slug'
```

Cache the result as `ACCOUNT_SLUG` and reuse it. For self-hosted single-account instances, the slug is stable and rarely changes.

## Fizzy workflow model

Cards flow through columns on a board. Fizzy also has three built-in states outside of columns:

- **Triage**: unprocessed inbox — cards land here before being placed in a column
- **Not Now**: explicitly deprioritized — Fizzy's entropy system will resurface these periodically
- **Done**: closed cards — immune to entropy, stay put permanently

Use columns for active workflow stages (e.g. "In Research", "In Development"). Use tags to add metadata or mark terminal states (e.g. `active`, `retired` on closed cards).

## Common workflows

### List all boards
```bash
curl -s -H "Authorization: Bearer $FIZZY_API_KEY" -H "Accept: application/json" \
  "$FIZZY_BASE_URL$ACCOUNT_SLUG/boards" | jq .
```

### Get columns for a board
```bash
curl -s -H "Authorization: Bearer $FIZZY_API_KEY" -H "Accept: application/json" \
  "$FIZZY_BASE_URL$ACCOUNT_SLUG/boards/$BOARD_ID/columns" | jq .
```

### List cards (with filters)
```bash
# All cards
curl -s -H "Authorization: Bearer $FIZZY_API_KEY" -H "Accept: application/json" \
  "$FIZZY_BASE_URL$ACCOUNT_SLUG/cards" | jq .

# Cards in a specific board
curl -s ... "$FIZZY_BASE_URL$ACCOUNT_SLUG/cards?board_ids[]=$BOARD_ID" | jq .

# Unassigned cards
curl -s ... "$FIZZY_BASE_URL$ACCOUNT_SLUG/cards?assignment_status=unassigned" | jq .

# Search cards by keyword
curl -s ... "$FIZZY_BASE_URL$ACCOUNT_SLUG/cards?terms[]=dark+mode" | jq .

# Filter by state (indexed_by values: all, maybe, closed, not_now, stalled, postponing_soon, golden)
curl -s ... "$FIZZY_BASE_URL$ACCOUNT_SLUG/cards?indexed_by=closed" | jq .
```

### Get a specific card
```bash
curl -s -H "Authorization: Bearer $FIZZY_API_KEY" -H "Accept: application/json" \
  "$FIZZY_BASE_URL$ACCOUNT_SLUG/cards/$CARD_NUMBER" | jq .
```
Cards are referenced by their human-readable **number** (e.g. `42`), not their internal ID.

### Create a card
```bash
curl -s -X POST \
  -H "Authorization: Bearer $FIZZY_API_KEY" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"card": {"title": "My new card", "description": "<p>Details here</p>"}}' \
  "$FIZZY_BASE_URL$ACCOUNT_SLUG/boards/$BOARD_ID/cards"
```
Returns `201 Created` with a `Location` header pointing to the new card.

### Update a card
```bash
curl -s -X PUT \
  -H "Authorization: Bearer $FIZZY_API_KEY" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"card": {"title": "Updated title"}}' \
  "$FIZZY_BASE_URL$ACCOUNT_SLUG/cards/$CARD_NUMBER"
```

### Delete a card
```bash
curl -s -X DELETE -H "Authorization: Bearer $FIZZY_API_KEY" \
  "$FIZZY_BASE_URL$ACCOUNT_SLUG/cards/$CARD_NUMBER"
```
Returns `204 No Content`. Permanent — use with care.

### Move a card into a column (triage)
```bash
curl -s -X POST \
  -H "Authorization: Bearer $FIZZY_API_KEY" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"column_id": "$COLUMN_ID"}' \
  "$FIZZY_BASE_URL$ACCOUNT_SLUG/cards/$CARD_NUMBER/triage"
```
Returns `204 No Content` on success.

### Send a card back to triage (remove from column)
```bash
curl -s -X DELETE \
  -H "Authorization: Bearer $FIZZY_API_KEY" \
  "$FIZZY_BASE_URL$ACCOUNT_SLUG/cards/$CARD_NUMBER/triage"
```

### Close / reopen a card
```bash
# Close (mark Done)
curl -s -X POST -H "Authorization: Bearer $FIZZY_API_KEY" \
  "$FIZZY_BASE_URL$ACCOUNT_SLUG/cards/$CARD_NUMBER/closure"

# Reopen
curl -s -X DELETE -H "Authorization: Bearer $FIZZY_API_KEY" \
  "$FIZZY_BASE_URL$ACCOUNT_SLUG/cards/$CARD_NUMBER/closure"
```

### Move a card to "Not Now"
```bash
curl -s -X POST -H "Authorization: Bearer $FIZZY_API_KEY" \
  "$FIZZY_BASE_URL$ACCOUNT_SLUG/cards/$CARD_NUMBER/not_now"
```

### Move a card to a different board
```bash
curl -s -X PUT \
  -H "Authorization: Bearer $FIZZY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"board_id": "$TARGET_BOARD_ID"}' \
  "$FIZZY_BASE_URL$ACCOUNT_SLUG/cards/$CARD_NUMBER/board"
```

### Assign / unassign a user (toggle)
```bash
curl -s -X POST \
  -H "Authorization: Bearer $FIZZY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"assignee_id": "$USER_ID"}' \
  "$FIZZY_BASE_URL$ACCOUNT_SLUG/cards/$CARD_NUMBER/assignments"
```
This is a toggle — calling it on an already-assigned user unassigns them.

### Add a comment
```bash
curl -s -X POST \
  -H "Authorization: Bearer $FIZZY_API_KEY" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"comment": {"body": "<p>This looks great!</p>"}}' \
  "$FIZZY_BASE_URL$ACCOUNT_SLUG/cards/$CARD_NUMBER/comments"
```

### Toggle a tag on a card
```bash
curl -s -X POST \
  -H "Authorization: Bearer $FIZZY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"tag_title": "bug"}' \
  "$FIZZY_BASE_URL$ACCOUNT_SLUG/cards/$CARD_NUMBER/taggings"
```
**Caution**: this is a toggle, not an add. Calling it twice removes the tag. Check the card's existing `tags` array before calling to avoid accidentally removing a tag.

### List all tags
```bash
curl -s -H "Authorization: Bearer $FIZZY_API_KEY" -H "Accept: application/json" \
  "$FIZZY_BASE_URL$ACCOUNT_SLUG/tags" | jq .
```

### List users
```bash
curl -s -H "Authorization: Bearer $FIZZY_API_KEY" -H "Accept: application/json" \
  "$FIZZY_BASE_URL$ACCOUNT_SLUG/users" | jq .
```

### Activity feed
```bash
# Recent activity
curl -s -H "Authorization: Bearer $FIZZY_API_KEY" -H "Accept: application/json" \
  "$FIZZY_BASE_URL$ACCOUNT_SLUG/activities" | jq .

# Filter by board
curl -s ... "$FIZZY_BASE_URL$ACCOUNT_SLUG/activities?board_ids[]=$BOARD_ID" | jq .

# Filter by user
curl -s ... "$FIZZY_BASE_URL$ACCOUNT_SLUG/activities?creator_ids[]=$USER_ID" | jq .
```

### My pinned cards
```bash
curl -s -H "Authorization: Bearer $FIZZY_API_KEY" -H "Accept: application/json" \
  "$FIZZY_BASE_URL$ACCOUNT_SLUG/my/pins" | jq .
```

### Pin / unpin a card
```bash
# Pin
curl -s -X POST -H "Authorization: Bearer $FIZZY_API_KEY" \
  "$FIZZY_BASE_URL$ACCOUNT_SLUG/cards/$CARD_NUMBER/pin"

# Unpin
curl -s -X DELETE -H "Authorization: Bearer $FIZZY_API_KEY" \
  "$FIZZY_BASE_URL$ACCOUNT_SLUG/cards/$CARD_NUMBER/pin"
```

### Steps (to-do items on a card)
```bash
# Create a step
curl -s -X POST \
  -H "Authorization: Bearer $FIZZY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"step": {"content": "Write tests"}}' \
  "$FIZZY_BASE_URL$ACCOUNT_SLUG/cards/$CARD_NUMBER/steps"

# Complete a step
curl -s -X PUT \
  -H "Authorization: Bearer $FIZZY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"step": {"completed": true}}' \
  "$FIZZY_BASE_URL$ACCOUNT_SLUG/cards/$CARD_NUMBER/steps/$STEP_ID"
```

## Pagination

All list endpoints are paginated. If there are more results, the response includes a `Link` header:
```
link: <https://fizzy.example.com/.../cards?page=2>; rel="next"
```

Follow the `next` link until it disappears to collect all pages. For most interactive tasks a single page is sufficient — only paginate when a complete list is needed.

## List parameters

When filtering by multiple IDs, repeat the parameter:
```
?board_ids[]=ID1&board_ids[]=ID2
```

## Error handling

| Status | Meaning |
|--------|---------|
| 401 | Bad or missing API key |
| 403 | Insufficient permissions |
| 404 | Resource not found (or no access) |
| 422 | Validation failed — response body has error details |
| 500 | Bad input (e.g. wrong type for a field) — validate before sending |

## Rich text fields

`description` (cards) and `body` (comments) accept HTML. Keep it simple:
```html
<p>Plain paragraph</p>
<p>With <strong>bold</strong> and <em>italic</em></p>
<ul><li>Item one</li><li>Item two</li></ul>
```

## Tips

- Resolve the account slug once via `/my/identity` and cache it for the session.
- Cards are addressed by **number** in URLs (e.g. `/cards/42`), not their internal `id`.
- The `column` field on a card is only present when the card has been triaged into a column; cards in Triage/Not Now/Done have no column.
- `closed: true` means the card is Done.
- Tag toggling via `/taggings` is a toggle — check existing `tags` before calling if you only want to add.
- When you need a board ID or column ID, list them first — they are opaque strings, not human-readable names.
