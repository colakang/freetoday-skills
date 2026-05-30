---
name: freetoday
description: Find today's free or deeply discounted food & drink deals near the user
  and auto-redeem them by driving the merchant's website to place a pickup order.
  Use this whenever the user mentions deal hunting, free coffee, free food, today's
  promos, "what's free near me", "any deals nearby", "anything free today", or asks
  to auto-order with a discount — even if they don't explicitly say "FreeToday".
  One redemption per merchant per day, enforced server-side.
metadata:
  version: "2026.05.30.1"
---

# FreeToday

Discovers daily food & drink deals near the user (by ZIP or city) and, on confirmation, drives the merchant's website to place a pickup order on the user's behalf.

The deals catalog and per-merchant ordering **playbooks** live behind an API at `https://deal.echo365.ai`. This skill is intentionally thin: it talks to the API and faithfully executes whatever playbook the API returns, using whatever browser tool the host agent has. Merchant flows change without touching the skill.

**Current scope:** pickup-only merchants (e.g. Cotti Coffee NYC). The order is placed in the merchant's app; the user walks to the store to collect. We do not handle delivery in v1.

**Rate limit:** at most ONE redemption per merchant per day (merchant-store-timezone day). The server enforces this with two keys — a stable installation UUID stored on the user's machine, AND a hash of the phone number used at merchant login. Don't try to bypass it.

## Workflow

Follow these steps in order. Don't skip the confirmation steps — placing an order on someone else's account without their say-so is exactly the kind of thing that gets a skill uninstalled.

1. **Make sure the installation_id exists** (one-time per machine, see [installation identity](#installation-identity)).
2. **Pick a mode** — see [Mode picker](#mode-picker) below. Decide between Mode A (you drive the merchant browser end-to-end) and Mode B (just hand the user a voucher code; they redeem manually).
3. **Ask the user for their location** — ZIP code preferred, city name acceptable. Don't guess; ask.
4. **Fetch deals** — `GET https://deal.echo365.ai/api/v1/deals?zip={zip}` or `?city={city}`. Show the user the list with deal IDs and titles, and let them pick.
5. **Mode A** — Reserve a same-day claim + fetch the playbook: `POST /api/v1/claim/start` with `{deal_id, installation_id, phone?, mac?, mode:"auto", agent_framework?, os?, locale?, email?}`. Then execute the playbook (step 6-8).
   **Mode B** — Skip to step 9 (verify-then-dispense).
6. **(Mode A only)** **Execute the playbook** — see [Executing a playbook](#executing-a-playbook). Variables collected via `ask_user` steps go into a local `vars` dict; `{{var}}` substitutions in subsequent steps draw from it.
7. **(Mode A only)** **At the `acquire_voucher` step**, call `POST /api/v1/voucher/lock` with `{claim_id}`. Populate `vars["voucher_code/description/kind"]` from the response.
8. **(Mode A only)** **Report the outcome** — `POST /api/v1/claim/complete` with `{claim_id, outcome, step_index?, notes?}`.

9. **(Mode B only)** **Verify the user's phone**:
   - Ask user for their phone (US format, with country code).
   - `POST /api/v1/verify/send-otp {phone, installation_id}` — server sends 6-digit code (Twilio in prod; dev mode echoes the code in the response for testing).
   - Ask user for the SMS code they received.
   - `POST /api/v1/verify/check-otp {phone, code, installation_id}` → on 200, capture `verification_token`. On 400/410/429, surface the message and try again or restart.
10. **(Mode B only)** **Dispense the voucher**:
    - `POST /api/v1/voucher/dispense {deal_id, installation_id, phone, verification_token, mac?, agent_framework?, os?, locale?, email?}`
    - Response includes `voucher.code`, `voucher.description`, `voucher.kind`, `voucher.discount_type`, and `instructions` (an array of redemption steps).
    - **DISPLAY THE VOUCHER CODE AND INSTRUCTIONS VERBATIM** to the user — they will manually paste the code into the merchant app.

Common error handling (both modes):
- **409 already_claimed_today** → tell the user when the merchant clock rolls over (`claimed_date` + `merchant_tz`). Don't retry.
- **503 voucher_pool_empty** → operator hasn't loaded a fresh batch. Tell user honestly.
- **503 daily_quota_exhausted** → merchant has hit its daily cap (e.g. 20/day). User must wait until tomorrow.

## Mode picker

The skill has two execution modes:

| | Mode A (auto-redeem) | Mode B (code-only) |
|---|---|---|
| What the skill does | Drives the merchant's site to log in, fill voucher, place order | Verifies user's phone via OTP, hands over the voucher code |
| Browser tool needed | YES (playwright-cli / camoufox-cli / Chrome MCP / browse_url) | NO |
| User interaction | OTP from merchant's app + CAPTCHA + confirmations | OTP from us (Twilio) only |
| Where the user goes next | Just pick up at the store | Open merchant app manually, paste code |
| Daily-slot consumed | Yes | Yes |

**How to pick:**

1. On first invocation in a session, detect whether you have a browser automation tool. Check (in order):
   - `which playwright-cli camoufox-cli 2>/dev/null` returning a path
   - Any MCP tool whose name matches `mcp__*browser*` / `mcp__*chrome*`
   - A built-in `browse_url` / `web_browser` capability advertised in your tool catalog
2. **If NO browser tool** → use Mode B. Tell the user "I can't drive your browser here, so I'll just hand you the code after phone verification — you'll redeem it yourself in the merchant app." Don't ask; just proceed.
3. **If browser tool present** → ASK the user once: "Want me to drive the whole flow in a browser (Mode A), or just give you the code and you'll redeem yourself (Mode B)? Mode A handles login + CAPTCHA + checkout for you; Mode B is faster but you'll do the redemption manually."
4. Remember the user's choice for this session — don't re-ask on subsequent claims in the same conversation.
5. If the user later types something like "just give me the code" or "I'll do it myself", switch to Mode B mid-conversation.

**Both modes share the same rate-limit** (one claim per merchant per day per user, enforced by installation_id + phone_hash + agent_uid_hash). Switching modes doesn't reset it.

## Calling the backend

Base URL: `https://deal.echo365.ai/api/v1`. No auth on `/deals`, `/claim/*`, `/voucher/*`. See `references/api-contract.md` for the full request/response shapes.

**Every API call MUST include the `X-Skill-Version` header.** Read it from the `VERSION` file alongside this `SKILL.md`:

```bash
SKILL_DIR="$(dirname "$0")"           # or wherever this skill is installed
SKILL_VER=$(cat "$SKILL_DIR/VERSION")  # e.g. "2026.05.30"
```

```bash
# Probe (cheap, no claim allocated)
curl -sS -H "X-Skill-Version: $SKILL_VER" \
  https://deal.echo365.ai/api/v1/skill-version/freetoday

# List deals
curl -sS -H "X-Skill-Version: $SKILL_VER" \
  "https://deal.echo365.ai/api/v1/deals?zip=10001"
```

### Mode A — auto-redeem via browser

```bash
# claim/start (mode=auto)
curl -sS -X POST https://deal.echo365.ai/api/v1/claim/start \
  -H "X-Skill-Version: $SKILL_VER" -H 'Content-Type: application/json' \
  -d '{"deal_id":"cotti-nyc-free-drink-booth-2026",
       "installation_id":"<from-local-file>","phone":"+15551234567",
       "mac":"<MAC>", "mode":"auto",
       "agent_framework":"claude-code","os":"macos","locale":"en-US"}'

# At the acquire_voucher step in the playbook
curl -sS -X POST https://deal.echo365.ai/api/v1/voucher/lock \
  -H "X-Skill-Version: $SKILL_VER" -H 'Content-Type: application/json' \
  -d '{"claim_id":"..."}'

# After the playbook ends
curl -sS -X POST https://deal.echo365.ai/api/v1/claim/complete \
  -H "X-Skill-Version: $SKILL_VER" -H 'Content-Type: application/json' \
  -d '{"claim_id":"...","outcome":"ok"}'
```

### Mode B — code-only with phone OTP verification

```bash
# 1. Send OTP (server uses Twilio if configured; dev mode logs to stdout)
curl -sS -X POST https://deal.echo365.ai/api/v1/verify/send-otp \
  -H "X-Skill-Version: $SKILL_VER" -H 'Content-Type: application/json' \
  -d '{"phone":"+15551234567","installation_id":"<from-local-file>"}'

# 2. After user types the SMS code back to you
curl -sS -X POST https://deal.echo365.ai/api/v1/verify/check-otp \
  -H "Content-Type: application/json" \
  -d '{"phone":"+15551234567","code":"924294","installation_id":"<...>"}'
# → captures verification_token from the response

# 3. Dispense the voucher (one round trip; server claims+allocates+completes)
curl -sS -X POST https://deal.echo365.ai/api/v1/voucher/dispense \
  -H "X-Skill-Version: $SKILL_VER" -H 'Content-Type: application/json' \
  -d '{"deal_id":"cotti-nyc-free-drink-booth-2026",
       "installation_id":"<from-local-file>","phone":"+15551234567",
       "verification_token":"<from-check-otp>",
       "agent_framework":"claude-desktop","os":"macos"}'
# → response includes voucher.code + instructions array; show user verbatim
```

If `curl` isn't in the host's tool palette, use whatever HTTP capability is: native `fetch`, a Python `requests` snippet, an MCP HTTP tool, etc. Don't fixate on shell — but always send the header.

### Handling the version response

Every successful response body has a `skill_meta` field:

```json
{
  "skill_meta": {
    "your_version": "2026.05.30",
    "latest": "2026.06.15",
    "min_required": "2026.05.30",
    "update_available": true,
    "update_required": false,
    "notes_url": "https://github.com/colakang/freetoday-skills/releases"
  }
}
```

Behaviour:
- `update_required: true` → server already returned **426 Upgrade Required** before reaching here, with a clear upgrade command in `detail.message`. Surface that to the user verbatim and stop.
- `update_available: true` → soft notice. Tell the user once per session: "📦 freetoday update available ({{latest}}). Run `git -C ~/.claude/skills/freetoday pull && git checkout {{latest}}` when convenient." Then continue normally.
- `update_available: false` → silent; carry on.

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
