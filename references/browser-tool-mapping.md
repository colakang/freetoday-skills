# Mapping playbook actions to host browser tools

The skill doesn't ship its own browser. It uses whatever the host agent has. Here's how to map the 7 playbook actions to the common options. Pick the first row in the table that matches the tools available in this session.

| Host agent has | Use |
|---|---|
| `playwright-cli` on PATH | Section A |
| `camoufox-cli` on PATH (e.g. you're inside OpenClaw or the user has it installed) | Section B |
| Chrome MCP server (`mcp__*__browser_*` tools) | Section C |
| A built-in `browse_url` / `web_browser` / similar | Section D |
| Only `WebFetch` / HTTP capability | Stop — see "When you don't have a browser" |

The host agent's CLAUDE.md or system context will usually disclose which of these is available. If unsure, run `which playwright-cli camoufox-cli 2>/dev/null` first; check tool listings for the MCP path.

## A. playwright-cli

Pattern (from the user's global CLAUDE.md):

```bash
# Once per session — bootstrap if not alive
playwright-cli list | grep "status: open" || {
  playwright-cli open --headed
  playwright-cli state-load ~/.claude/auth-sessions.json
}

# Reuse tab if URL already matches; otherwise navigate the current tab
playwright-cli tab-list
playwright-cli tab-select 0          # switch focus to first tab
```

Action mapping:

| Playbook action | Command |
|---|---|
| `navigate` | `playwright-cli goto "<url>"` |
| `wait_for` | `playwright-cli wait-for "<selector>" --timeout <ms>` |
| `click` | `playwright-cli click "<selector>"` |
| `fill` | `playwright-cli fill "<selector>" "<value>"` |
| `extract` | `playwright-cli text "<selector>"` |
| `ask_user` | Ask in chat. No browser involvement. |
| `confirm_with_user` | Ask in chat. No browser involvement. |

State persistence (login session): on the FIRST successful run, after login, `playwright-cli state-save ~/.claude/auth-sessions.json` so subsequent runs skip the OTP step. Per CLAUDE.md the session is a long-lived OS process — don't kill it.

Sandbox note (also from CLAUDE.md): playwright-cli usually needs `dangerouslyDisableSandbox: true` to launch Chrome; that's a Claude Code-specific flag, irrelevant on other hosts.

## B. camoufox-cli

Same shape, anti-fingerprint Firefox. Used by sibling skills like `quest-appointment` for Angular SPAs.

```bash
# Session bootstrap (headed + persistent — both required, or state vanishes)
camoufox-cli --session deal-today --headed --persistent --timeout 3600 open "<url>"
```

Action mapping mostly mirrors playwright-cli; check `camoufox-cli --help` for the exact subcommand spellings. Notable difference: camoufox's `fill` does a strict visibility check that fails on hidden inputs (common in Material/Vuetify). For those, fall back to evaluating a JS snippet that sets value + dispatches `input` + `change`. quest-appointment has examples.

## C. Chrome MCP

If the host exposes Chrome MCP tools (names like `mcp__chrome__browser_navigate`, `..._click`, `..._type`, `..._snapshot`), use them directly:

| Playbook action | MCP tool (typical name) |
|---|---|
| `navigate` | `browser_navigate` |
| `wait_for` | `browser_wait_for` (or `browser_snapshot` + poll) |
| `click` | `browser_click` |
| `fill` | `browser_type` or `browser_fill_form` |
| `extract` | `browser_evaluate` returning `document.querySelector(...).innerText`, or read from a recent `browser_snapshot` |

Names vary slightly between Chrome MCP implementations. Check the tool catalog the host advertises; the semantics line up.

## D. Built-in browse / open_url

Some hosts (Claude.ai web, Cowork, certain MCP wrappers) provide a single `browse_url(url) → text+screenshot` tool with no fine-grained click/fill. This is the hardest case — you can `navigate`, but `click`/`fill`/`extract` don't have direct primitives.

If this is all you have:

1. Stop at the first non-navigate action.
2. Tell the user honestly: "the host I'm running on doesn't have fine browser control. I've got you to the start page; here are the remaining steps; please drive the rest in your browser." Walk them through the steps verbatim using the `prompt`/`message` from any `ask_user`/`confirm_with_user` steps.
3. Call `/claim/complete` with `outcome="other"`, `notes="host lacks click/fill primitives; handed off to user at step N"`.

This degrades gracefully — better than silently failing or hallucinating selectors.

## When you don't have a browser at all

`/api/v1/deals` and `/api/v1/claim/start` are still useful even without a browser — the user can see what's available. Show the list, tell the user you can't auto-redeem here, suggest they re-run on a host with a browser tool. Don't call `claim/start` if you have no intention of executing the playbook; that wastes their daily slot.

## Sandbox/permission shape

Across all of these:

- Browser tools that write profile/state to `~/.config/...`, `~/.cache/...`, or `~/.claude/...` may need explicit permission grants on Claude Code. Check the host's setting documentation if first invocation fails.
- The skill's own writes are limited to one file: `~/.config/deal-today/installation_id`. That's it.

## A note on rate-limit and tool choice

The rate-limit lives server-side (deals-api). It doesn't care which browser tool you used. If the same physical user hits the same merchant on a different host with a different installation_id but the same phone number, the server's phone-hash dedup will catch it. So pick the tool that maps cleanest to the merchant's SPA, not the one that gives you the best fingerprint.
