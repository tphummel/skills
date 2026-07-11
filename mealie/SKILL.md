---
name: mealie
description: Search, read, and manage recipes in a self-hosted Mealie instance via its REST API. Use whenever the user wants to find a recipe, look up ingredients/instructions, duplicate or modify a recipe (e.g. testing a variation with a different ingredient), manage shopping lists, or browse tags/categories. Trigger on requests like "find my X recipe", "what's in the Y recipe", "duplicate this recipe so I can tweak it", "add these ingredients to my shopping list", "what recipes have tag Z", or any mention of Mealie or "my recipes".
---

# Mealie API Skill

Mealie is a self-hosted recipe manager and meal-planner. This skill covers the
recipe CRUD, shopping list, and tag/category surface of its REST API.

## Authentication

Base URL and API token are already available as environment variables:

```
MEALIE_BASE_URL   base URL of your Mealie instance
MEALIE_API_KEY    Bearer token, generated in the Mealie UI under user avatar > Manage Your Account > API Tokens
```

All requests:
```
Authorization: Bearer $MEALIE_API_KEY
Accept: application/json
Content-Type: application/json   (for POST/PUT/PATCH)
```

API base: `$MEALIE_BASE_URL/api`

If `MEALIE_API_KEY` isn't set, tell the user it needs to be generated from the
Mealie web UI and wired into the chezmoi-managed
`~/.claude/private_settings.json.tmpl` the same way the other `*_API_KEY`
entries are — don't try to obtain it any other way.

---

## Search — the main entry point

```bash
curl -s -H "Authorization: Bearer $MEALIE_API_KEY" \
  --get --data-urlencode "search=$QUERY" --data-urlencode "perPage=50" \
  "$MEALIE_BASE_URL/api/recipes" | jq '.items[] | {name, slug}'
```

Search is loose (matches on name/description/ingredients) — try a couple of
distinct keywords from the user's phrasing if the first search comes back
empty or with a name that doesn't quite match what they described (e.g. try
both a full phrase and its parts separately).

Other useful query params on `/api/recipes`:
- `categories`, `tags`, `tools`, `foods` — filter by slug or id (repeatable)
- `orderBy`, `orderDirection` (`asc`/`desc`)
- `page`, `perPage` (default 50)

## Get a single recipe

```bash
curl -s -H "Authorization: Bearer $MEALIE_API_KEY" \
  "$MEALIE_BASE_URL/api/recipes/{slug}" | jq .
```
`{slug}` accepts either the recipe's slug or its id. Key fields:
- `recipeIngredient[]` — each has a `note` (free text) plus optional structured `food`/`unit`/`quantity`
- `recipeInstructions[]` — each has a `text`
- `tags[]`, `recipeCategory[]` — each `{id, name, slug}`

For "what's in recipe X" questions, `jq '{name, description, recipeIngredient: [.recipeIngredient[].note], recipeInstructions: [.recipeInstructions[].text]}'` gives a readable summary without the structured-data noise.

## Modifying a recipe

**Prefer duplicating before editing** when the user is testing a variation
("what if I swapped X for Y") rather than correcting the original — this
preserves the source recipe:

```bash
curl -s -X POST -H "Authorization: Bearer $MEALIE_API_KEY" -H "Content-Type: application/json" \
  -d '{"name": "Spicy Mezcal Margaritas (Ancho Reyes variant)"}' \
  "$MEALIE_BASE_URL/api/recipes/{slug}/duplicate" | jq '{name, slug}'
```

Then update the duplicate. PUT replaces the whole recipe; fetch the current
recipe first, edit the JSON in place (e.g. add/replace an ingredient note or
instruction text), and PUT the whole object back — Mealie's PATCH endpoint
also exists but expects the same full `Recipe-Input` shape, so there's no
real partial-update shortcut:

```bash
curl -s -X PUT -H "Authorization: Bearer $MEALIE_API_KEY" -H "Content-Type: application/json" \
  -d @/tmp/edited-recipe.json \
  "$MEALIE_BASE_URL/api/recipes/{slug}"
```

Confirm with the user before editing/deleting a recipe they didn't ask you to
duplicate first — there's version history in the Mealie UI but no API undo.

## Creating a recipe from scratch

```bash
curl -s -X POST -H "Authorization: Bearer $MEALIE_API_KEY" -H "Content-Type: application/json" \
  -d '{"name": "New Recipe Name"}' \
  "$MEALIE_BASE_URL/api/recipes"
```
Returns just the new slug (a bare JSON string) — follow up with a GET, then
PUT the full ingredients/instructions/etc.

Mealie can also import from a URL or pasted HTML/JSON:
```bash
curl -s -X POST -H "Authorization: Bearer $MEALIE_API_KEY" -H "Content-Type: application/json" \
  -d '{"url": "https://example.com/some-recipe", "includeTags": false}' \
  "$MEALIE_BASE_URL/api/recipes/create/url"
```

## Shopping lists

```bash
# list shopping lists
curl -s -H "Authorization: Bearer $MEALIE_API_KEY" "$MEALIE_BASE_URL/api/households/shopping/lists" | jq '.items[] | {id, name}'

# add a recipe's ingredients to a list
curl -s -X POST -H "Authorization: Bearer $MEALIE_API_KEY" -H "Content-Type: application/json" \
  -d '{"recipeId": "{recipe_id}", "recipeIncrementQuantity": 1}' \
  "$MEALIE_BASE_URL/api/households/shopping/lists/{list_id}/recipe"

# add a one-off item
curl -s -X POST -H "Authorization: Bearer $MEALIE_API_KEY" -H "Content-Type: application/json" \
  -d '{"shoppingListId": "{list_id}", "note": "spicy liqueur"}' \
  "$MEALIE_BASE_URL/api/households/shopping/items"
```

## Tags and categories

```bash
curl -s -H "Authorization: Bearer $MEALIE_API_KEY" "$MEALIE_BASE_URL/api/organizers/tags" | jq '.items[] | {id, name, slug}'
curl -s -H "Authorization: Bearer $MEALIE_API_KEY" "$MEALIE_BASE_URL/api/organizers/categories" | jq '.items[] | {id, name, slug}'
```
Pass the resulting slugs/ids as `tags=`/`categories=` filters on `/api/recipes`.

---

## Tips

- **Pagination**: list endpoints return `page`/`total_pages` — bump `page` rather than assuming everything fits in one response (default `perPage` is 50).
- **401 means the token is missing/invalid** — don't retry blindly; check `MEALIE_API_KEY` is actually set.
- **Slug vs id**: most single-recipe endpoints accept either interchangeably; prefer the slug for readability in commands.
- **Recipe-Input vs Recipe-Output**: writes (POST/PUT/PATCH) expect a slightly leaner shape than what GET returns (some computed/read-only fields are dropped) — safest to GET the recipe, mutate the JSON you got back, and send that same object rather than hand-building one.
