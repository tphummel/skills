---
name: search-web-via-searxng
description: Override Claude Code's built-in WebSearch tool to route web searches through a self-hosted SearXNG instance via curl instead
argument-hint: [searxng-url]
---

# Route web search through self-hosted SearXNG

You are configuring Claude Code to stop using its built-in `WebSearch` tool and
instead search the web by curling a self-hosted SearXNG instance: `$ARGUMENTS`
(e.g. `https://searxng.example.com`). If no URL was given, ask the user for
their SearXNG instance URL before doing anything else.

There is no native setting to redirect `WebSearch` itself — it's an
Anthropic-managed tool with no config hook. The approach is: deny the built-in
tool via permissions, and teach Claude to curl SearXNG directly as a
replacement. This is much lighter than standing up an MCP server for it.

## Step 1: Confirm SearXNG returns JSON

SearXNG's JSON output format is disabled by default (`format=json` returns a
plain `403 Forbidden`, not a JSON error body — don't mistake this for an auth
problem).

```bash
curl -s -o /dev/null -w "%{http_code}\n" "$ARGUMENTS/search?q=test&format=json"
```

- `200` → JSON is already enabled, skip to Step 3.
- `403` → JSON format needs to be enabled server-side first (Step 2).

## Step 2: Enable JSON format on the SearXNG instance (if needed)

Find the instance's `settings.yml` (for this homelab, check the Ansible repo
first: `rg -l searxng ~/Code/*/ansible* 2>/dev/null`, then look for a
`copy: content:` block that writes `/opt/searxng/settings.yml`). Add a
`search.formats` list including `json` alongside `html`:

```yaml
search:
  formats:
    - html
    - json
```

If the instance is deployed via `ansible-pull` + systemd timers (the pattern
used in this homelab — see the `gitea` or `new-homelab-guest` skills for
repo conventions):

1. Commit and push the settings.yml change to the ansible repo's main branch
   — ansible-pull deploys from the git remote, so an uncommitted local edit
   won't take effect.
2. Trigger an immediate deploy over SSH rather than waiting for the timer:
   ```bash
   ssh <guest-hostname> 'systemctl start ansible-pull.service'
   ```
3. Poll until it finishes and check the result:
   ```bash
   ssh <guest-hostname> 'until ! systemctl is-active --quiet ansible-pull.service; do sleep 3; done; systemctl show ansible-pull.service -p Result,ExecMainStatus'
   ```
   `Result=success` / `ExecMainStatus=0` means it applied cleanly. If it
   restarts the container, the service may be briefly unavailable.

Re-run the Step 1 curl to confirm `200` before moving on.

## Step 3: Deny the built-in WebSearch tool

Add to `~/.claude/settings.json` (create it if it doesn't exist — note the
`Write` tool requires reading a file before overwriting it, so `Read` first
even if you expect it to be missing):

```json
{
  "permissions": {
    "deny": ["WebSearch"],
    "allow": ["Bash(curl:*<searxng-host>*)"]
  }
}
```

Merge with any existing permissions rather than clobbering them. Use the
instance's actual host in the allow pattern so curl calls to it don't prompt
every time.

## Step 4: Teach Claude to curl SearXNG instead

Append to `~/.claude/CLAUDE.md`:

```
To search the web, use curl against my SearXNG instance instead of WebSearch (which is disabled):

curl -s "$ARGUMENTS/search?q=<query>&format=json" | jq '.results[] | {title, url, content}'

URL-encode the query. Use this whenever you'd otherwise reach for WebSearch.
```

## Step 5: Verify end-to-end

```bash
curl -s "$ARGUMENTS/search?q=claude+code&format=json" | jq '.results[] | {title, url, content}'
```

Confirm it returns real, well-formed results (not an empty `results` array —
if empty, check `unresponsive_engines` in the raw JSON for upstream engine
failures, which is a SearXNG-side issue unrelated to this setup).

## Caveats

- No fallback: once `WebSearch` is denied, if SearXNG is down, Claude has no
  web search at all.
- Managed org settings that force-allow `WebSearch` may override a local deny
  rule — check `claude config` precedence if the deny doesn't seem to take.
- `CLAUDE.md` instructions alone are not enough; the deny rule is what
  actually removes `WebSearch` as an option so Claude doesn't just use both.
