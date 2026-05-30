---
name: freetoday
description: Find today's free or deeply discounted food & drink deals near the user
  and auto-redeem them by driving the merchant's website to place a pickup order.
  Use this whenever the user mentions deal hunting, free coffee, free food, today's
  promos, "what's free near me", "any deals nearby", "anything free today", or asks
  to auto-order with a discount — even if they don't explicitly say "FreeToday".
  One redemption per merchant per day, enforced server-side.
---

# FreeToday

Discovers daily food & drink deals near the user (by ZIP or city) and, on confirmation, drives the merchant's website to place a pickup order on the user's behalf.

The deals catalog and per-merchant ordering **playbooks** live behind an API at `https://deal.echo365.ai`. This skill is intentionally thin: it talks to the API and faithfully executes whatever playbook the API returns, using whatever browser tool the host agent has. Merchant flows change without touching the skill.

**Current scope:** pickup-only merchants (e.g. Cotti Coffee NYC). The order is placed in the merchant's app; the user walks to the store to collect. We do not handle delivery in v1.

**Rate limit:** at most ONE redemption per merchant per day (merchant-store-timezone day). The server enforces this with two keys — a stable installation UUID stored on the user's machine, AND a hash of the phone number used at merchant login. Don't try to bypass it.

## Workflow

Follow these steps in order. Don't skip the confirmation steps — placing an order on someone else's account without their say-so is exactly the kind of thing that gets a skill uninstalled.

1. **Make sure the installation_id exists** (one-time per machine, see [installation identity](#installation-identity)).
2. **Ask the user for their location** — ZIP code preferred, city name acceptable. Don't guess; ask.
3. **Fetch deals** — `GET https://deal.echo365.ai/api/v1/deals?zip={zip}` or `?city={city}`. Show the user the list with deal IDs and titles, and let them pick.
4. **Reserve a same-day claim + fetch the playbook** — `POST /api/v1/claim/start` with `{deal_id, installation_id, phone?, mac?}`. The server uses these to compute three rate-limit keys: `installation_id`, `phone_hash`, and `agent_uid_hash` = sha256(mac + phone + server_pepper). Same-merchant-same-day claims that match ANY of those are rejected.
   - **409 already_claimed_today** → tell the user when their next attempt is allowed (`claimed_date` + `merchant_tz` are in the response) and stop.
   - The response does NOT include a voucher — voucher allocation is lazy, see step 6.
5. **Execute the playbook** — see [Executing a playbook](#executing-a-playbook). Variables collected via `ask_user` steps go into a local `vars` dict; `{{var}}` substitutions in subsequent steps draw from it.
6. **At the `acquire_voucher` step**, call `POST /api/v1/voucher/lock` with `{claim_id}`. The server allocates a real merchant voucher from the pool (lazily — only when the playbook actually reaches the redemption step). Populate:
   - `vars["voucher_code"]        = response.code`
   - `vars["voucher_description"] = response.description`
   - `vars["voucher_kind"]        = response.kind`
   - **503 voucher_pool_empty** → abort. Call `claim/complete` with `outcome="voucher_pool_empty"` and tell the user honestly.
7. **Report the outcome** — when the playbook ends (success OR failure), `POST /api/v1/claim/complete` with `{claim_id, outcome, step_index?, notes?}`. `outcome="ok"` is the only path that counts as success and burns the voucher. Aborting before `acquire_voucher` is free of voucher-pool impact (the daily slot for that user IS consumed by the started claim). Aborting AT or AFTER `acquire_voucher` releases the voucher back to the pool for outcomes the server can confidently call "not consumed merchant-side" (`user_aborted`, `captcha`, `voucher_pool_empty`); ambiguous outcomes leave it `locked` until the 30-minute stale-lock sweep recycles it.

## Calling the backend

Base URL: `https://deal.echo365.ai/api/v1`. No auth on `/deals` or `/claim/*`. See `references/api-contract.md` for the full request/response shapes.

```
# List
curl -sS "https://deal.echo365.ai/api/v1/deals?zip=10001"

# Start a claim (returns playbook + claim_id, or 409)
curl -sS -X POST https://deal.echo365.ai/api/v1/claim/start \
  -H 'Content-Type: application/json' \
  -d '{"deal_id":"cotti-nyc-free-drink-booth-2026",
       "installation_id":"<from-local-file>",
       "phone":"+15551234567"}'

# Complete it (after the playbook finishes, success or fail)
curl -sS -X POST https://deal.echo365.ai/api/v1/claim/complete \
  -H 'Content-Type: application/json' \
  -d '{"claim_id":"...","outcome":"ok"}'
```

If `curl` isn't in the host's tool palette, use whatever HTTP capability is: native `fetch`, a Python `requests` snippet, an MCP HTTP tool, etc. Don't fixate on shell.

## Installation identity

The server needs a stable per-installation key so it can enforce "one redemption per day per machine." On first run:

1. Check whether the file `~/.config/freetoday/installation_id` exists (`$XDG_CONFIG_HOME/freetoday/installation_id` if `XDG_CONFIG_HOME` is set).
2. If yes, read its 32-char UUID and use it for every API call as the `installation_id` field.
3. If no, generate a UUIDv4 (`uuidgen` / `python3 -c 'import uuid; print(uuid.uuid4())'` / language-equivalent), `mkdir -p` the parent, write it `chmod 600`.

The file is the only persistent state the skill writes. Never overwrite an existing one — that's the user trying to reset, and it's their right, but the server will catch the second redemption today via the phone hash + agent_uid_hash combo.

## MAC detection (for agent_uid_hash)

Along with phone, the server uses MAC to compute a stronger per-user fingerprint than installation_id alone. Detect the first non-loopback interface MAC on the host:

| OS | Command |
|---|---|
| macOS | `ifconfig en0 \| awk '/ether/{print $2}'` |
| Linux | `cat /sys/class/net/$(ls /sys/class/net \| grep -E '^(en\|eth)' \| head -1)/address` |
| Linux/macOS fallback | `ip link show 2>/dev/null \| awk '/link\/ether/{print $2; exit}'` |
| Windows | `getmac /v /fo csv \| findstr -v 'Disconnected' \| head -1` |

If you can't detect a MAC (running in docker without `--network=host`, a hardened sandbox, WSL with virtualized networking, or anywhere `ifconfig`/`ip` aren't installed), pass `mac: null` (or omit) in the `claim/start` request. The server gracefully degrades to phone-only fingerprinting — same level of protection as the v1 phone-hash rate-limit.

Don't fabricate or hash a MAC client-side. Just send what you have (raw) and let the server's pepper handle the hashing.

## Executing a playbook

The playbook returned by `claim/start` looks like this (truncated):

```json
{
  "playbook_id": "cotti-pickup-v1",
  "merchant_id": "cotti",
  "version": 1,
  "steps": [
    {"action": "navigate", "url": "https://mobile.us.cotticoffee.global/#/pages/tabs"},
    {"action": "wait_for", "selector": "...", "timeout_ms": 10000},
    {"action": "ask_user", "prompt": "Enter your phone", "var": "phone"},
    {"action": "fill",     "selector": "...", "value": "{{phone}}"},
    ...
  ]
}
```

Loop over `steps` in order. Maintain a local `vars` dict (string→string). For each step:

- **`navigate`** — open `step.url` in the browser.
- **`wait_for`** — wait for `step.selector` to be present, up to `step.timeout_ms` (default 8000).
- **`click`** — click `step.selector`.
- **`fill`** — type `step.value` into `step.selector`. Before typing, substitute every `{{var}}` in `step.value` with the matching key from `vars`.
- **`extract`** — read the text content of `step.selector` and store it under `vars[step.var]`.
- **`ask_user`** — pause execution. Ask the user IN THE CHAT, verbatim, the contents of `step.prompt`. When they reply, store it under `vars[step.var]`. If `step.mask` is true, tell the user that what they type won't be shown back to them — but you cannot actually hide chat history; do not lie about masking.
- **`confirm_with_user`** — render `step.message` with `{{var}}` substitution and wait for an explicit yes/no. On no, abort the playbook and call `/claim/complete` with `outcome="user_aborted"`.
- **`acquire_voucher`** — call `POST /api/v1/voucher/lock` with `{claim_id: <current>}`. On 200, populate `vars["voucher_code"]`, `vars["voucher_description"]`, `vars["voucher_kind"]` from the response. On 503 voucher_pool_empty, abort with `claim/complete outcome="voucher_pool_empty"`.

If `step.selector` starts with `__TBD_` it's a placeholder for an un-mapped flow — stop, tell the user "this playbook is still being mapped; here's what I'd do next: ..." and call `/claim/complete` with `outcome="other"`, notes describing where you stopped.

The browser tool you use depends on what the host gives you — see `references/browser-tool-mapping.md` for concrete adapter patterns (playwright-cli, camoufox-cli, Chrome MCP, and built-in browse fallback).

## Handling sensitive input

`ask_user` steps for phone numbers and OTP codes are inherently sensitive. Treat their values in `vars` as ephemeral: don't write them to disk, don't log them, and after `claim/complete` succeeds, discard them. If the host agent's transcript will be archived (Claude Code transcripts often are), you can't prevent that, but at least don't pile on by writing them anywhere else. The phone gets sent to the deals-api once — that's intentional, for rate-limit deduplication — but never quote it back to the user when you don't need to.

## Confirming before placing the order

The merchant playbook will include at least one `confirm_with_user` step right before the final "place order" click. **Honor it.** Do not click through it without an explicit user yes. If the user says no, abort and call `claim/complete` with `outcome="user_aborted"`.

## Error recovery

Common failure modes and how to handle them:

| Failure | Action |
|---|---|
| `wait_for` times out | Retry the step once. If still failing, `claim/complete` with `outcome="selector_miss"` and `step_index` set. Tell the user the merchant site appears to have changed; the operator will refresh the playbook. |
| CAPTCHA appears | Most merchants don't have one for redemption flows, but if you see one: stop, tell the user honestly, complete with `outcome="captcha"`. |
| Login fails (wrong OTP) | Ask the user to re-enter. After two failures, complete with `outcome="login_fail"`. |
| Network/API 5xx | Retry with backoff (2s, 5s). After 3 failures, complete with `outcome="other"`. |

When you call `/claim/complete` with anything other than `outcome="ok"`, include a brief `notes` describing what went wrong. The operator uses these to fix the playbook server-side — short specific notes save round-trips.

## References

- [`references/api-contract.md`](references/api-contract.md) — exact endpoint shapes, status codes, response examples
- [`references/playbook-schema.md`](references/playbook-schema.md) — full step-action semantics + variable substitution rules
- [`references/browser-tool-mapping.md`](references/browser-tool-mapping.md) — how to map step actions to playwright-cli / camoufox-cli / Chrome MCP / built-in browse
- [`references/merchants/cotti.md`](references/merchants/cotti.md) — Cotti-specific notes (pickup-only, NYC store, booth-code funnel)
