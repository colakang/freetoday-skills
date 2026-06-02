---
name: freetoday
description: Find today's free or deeply discounted food & drink deals near the user
  and either auto-redeem them by driving the merchant's website, or hand the user a
  voucher code to redeem themselves. Use this whenever the user mentions deal hunting,
  free coffee, free food, today's promos, "what's free near me", "any deals nearby",
  "anything free today", or asks to grab/redeem a discount — even if they don't
  explicitly say "FreeToday". Works in any agent that can make HTTP requests; a
  browser-automation tool is optional (only needed for hands-free auto-redeem).
  One redemption per merchant per day, enforced server-side.
metadata:
  version: "2026.06.02"
---

# FreeToday

Finds daily food & drink deals near the user (by ZIP or city) and gets the deal into
their hands one of two ways:

- **Auto-redeem** — the agent drives the merchant's website (log in, apply the
  voucher, place the pickup order) so the user does nothing. Requires a
  browser-automation tool in the host environment.
- **Code handoff** — the agent verifies the user's phone with a one-time code, then
  hands over a voucher code + step-by-step instructions for the user to redeem
  themselves. Works anywhere, no browser tool needed.

This skill is host-agnostic: it works in **any** agent that can make HTTP requests
(Claude Code, Claude Desktop, Codex, Cursor, OpenClaw, custom agents, …). The deals
catalog and per-merchant **playbooks** live behind an API at `https://freetoday.ai`;
the skill is intentionally thin — it talks to the API and executes whatever the API
returns, using whatever tools the host happens to have. Merchant flows change without
touching the skill.

**Current scope:** pickup-only merchants (e.g. Cotti Coffee NYC). The order is placed
in the merchant's app; the user walks to the store to collect. No delivery in v1.

**Rate limit:** at most ONE redemption per merchant per day (merchant-store-timezone
day), enforced server-side on three keys (installation id + phone hash + a device
fingerprint). Don't try to bypass it.

## Workflow

Follow in order. Don't skip the confirmation steps — redeeming on someone's behalf
without an explicit yes is exactly what gets a skill uninstalled.

1. **Ensure the installation id exists** (one-time per machine — see
   [Installation identity](#installation-identity)).
2. **Ask the user for their location** — ZIP preferred, city acceptable. Don't guess.
3. **Fetch deals** — `GET /api/v1/deals?zip={zip}` (or `?city={city}`). Show the list
   with titles and let the user pick one. Each deal carries `redeem_modes` (which paths
   it supports) and `stores` (where the voucher is redeemed) — read both.
4. **Pick the path** (see [Choosing how to redeem](#choosing-how-to-redeem)). It's
   bounded by the deal's `redeem_modes` AND whether this host has a browser tool:
   - If the deal supports only `["code"]`, or this host has no browser tool → go
     straight to **Code handoff**; don't offer auto-redeem.
   - If the deal supports both `["auto","code"]` and a browser tool exists → ask:
     > "Want me to redeem it automatically for you, or would you rather I just get you
     > the code and you redeem it yourself?"
   - **Auto-redeem** → [Auto-redeem path](#auto-redeem-path) · **Code handoff** →
     [Code-handoff path](#code-handoff-path)

Common errors (either path):
- **409 mode_not_supported** → this deal doesn't support the path you tried (e.g. you
  called `claim/start mode=auto` on a code-only deal). Switch to the path in
  `supported_modes` from the response.
- **409 already_claimed_today** → tell the user when the merchant clock rolls over
  (`claimed_date` + `merchant_tz` in the response). Don't retry.
- **503 voucher_pool_empty** → no codes loaded right now. Tell the user honestly.
- **503 daily_quota_exhausted** → the merchant hit its daily cap. Try again tomorrow.
- **426 upgrade_required** → this skill is too old; show the upgrade command from the
  response and stop. See [Version handling](#version-handling).

## Choosing how to redeem

Two paths. The choice is the user's, but it's bounded by what the host can do.

| | Auto-redeem | Code handoff |
|---|---|---|
| What the agent does | Drives the merchant site: login, voucher, order | Verifies phone, hands over the code |
| Browser tool needed | **Yes** | No |
| User effort | Approve a couple of prompts; pick up at store | Verify a code once; redeem in the app themselves |

How to decide, in order:

0. **Check the deal's `redeem_modes`.** If it's `["code"]` only, skip straight to the
   code-handoff path — don't detect a browser or ask the question. (`claim/start` with
   `mode:"auto"` on such a deal returns **409 `mode_not_supported`** anyway.) Only when
   `redeem_modes` includes `"auto"` do steps 1–3 below apply.
1. **Detect a browser-automation tool.** Check, in order:
   - `which playwright-cli camoufox-cli 2>/dev/null` returns a path
   - the host advertises an MCP browser tool (name matches `*browser*` / `*chrome*`)
   - the host has a built-in `browse_url` / `web_browser` capability
2. **No browser tool** → only code handoff is possible. Tell the user plainly:
   "This environment can't drive a browser, so I'll verify your phone and hand you the
   code to redeem yourself — sound good?" Then take the code-handoff path.
3. **Browser tool present AND deal supports `auto`** → ask the question in step 4 of the
   workflow and let the user choose. Auto-redeem if they want hands-free; code handoff
   if they'd rather do it themselves.
4. **Remember the choice for the session.** Don't re-ask on later deals in the same
   conversation. If the user later says "just give me the code" / "I'll do it myself",
   switch to code handoff.

Both paths share the same one-per-merchant-per-day limit; switching doesn't reset it.

## Auto-redeem path

1. **Reserve a claim + fetch the playbook** — `POST /api/v1/claim/start` with
   `{deal_id, installation_id, phone?, mac?, mode:"auto", agent_framework?, os?, locale?}`.
   (The phone here is the one the merchant login will use; the playbook will ask for it
   if you don't pass it.)
2. **Execute the playbook** — see [Executing a playbook](#executing-a-playbook).
3. **At the `acquire_voucher` step**, `POST /api/v1/voucher/lock {claim_id}` and load
   `vars["voucher_code" / "voucher_description" / "voucher_kind"]` from the response.
4. **Report the outcome** — `POST /api/v1/claim/complete {claim_id, outcome, step_index?, notes?}`.
   On `outcome:"ok"`, the server binds the merchant-login phone to this installation
   automatically (the merchant's own login proved phone ownership) — so a later
   code-handoff redemption on the same device won't need a separate OTP.

## Code-handoff path

The user redeems themselves; the agent fetches a code. Phone verification is a
**one-time** step — the first time this installation ever redeems, it verifies a phone
to establish identity. After that, no OTP is needed; the one-per-merchant-per-day limit
is the gate.

1. **Try to dispense directly** — `POST /api/v1/voucher/dispense {deal_id,
   installation_id, agent_framework?, os?, locale?}`.
   - **200** → this installation is already known. Jump to step 4.
   - **403 phone_not_bound** → first redemption ever on this device; do the one-time
     phone check in steps 2-3, then retry dispense.
2. **Verify a phone (first time only)** — ask for the user's mobile
   ("What's your mobile number? US format with country code, e.g. +1 555 123 4567"),
   then `POST /api/v1/verify/send-otp {phone, installation_id}` (server texts a 6-digit
   code), ask for the code, and `POST /api/v1/verify/check-otp {phone, code,
   installation_id}`. On 200, capture `verification_token`. On 400/410/429 surface the
   message and resend/retry.
3. **Dispense with the binding** — retry `POST /api/v1/voucher/dispense` adding
   `phone` + `verification_token`. This binds the phone to the installation and returns
   the voucher.
4. **Show the `voucher.code`, the `instructions` array, and `stores` verbatim.** The
   steps are merchant-specific — they may tell the user to enter the code in the app
   (e.g. Cotti) or simply show it at the counter (e.g. TEAPULSE). Always include the
   redemption store address(es); don't paraphrase.

> The phone binding is set once (here, or — for the auto-redeem path — automatically on
> the first successful merchant login). It identifies the installation; it is NOT
> re-verified on later redemptions.

## Calling the backend

Base URL: `https://freetoday.ai/api/v1`. No API key needed. See
`references/api-contract.md` for full request/response shapes.

**Every API call MUST send the `X-Skill-Version` header**, read from the `VERSION`
file next to this `SKILL.md`:

```bash
SKILL_DIR="$(dirname "$0")"            # wherever this skill is installed
SKILL_VER=$(cat "$SKILL_DIR/VERSION")  # e.g. "2026.05.31"
```

Use `curl` if available, otherwise any HTTP capability the host has — native `fetch`,
a Python `requests` snippet, an MCP HTTP tool. The transport doesn't matter; the
header does.

```bash
# Discover (cheap, allocates nothing)
curl -sS -H "X-Skill-Version: $SKILL_VER" "$BASE/api/v1/deals?zip=10001"
```

### Auto-redeem calls

```bash
curl -sS -X POST "$BASE/api/v1/claim/start" \
  -H "X-Skill-Version: $SKILL_VER" -H 'Content-Type: application/json' \
  -d '{"deal_id":"…","installation_id":"…","phone":"+15551234567",
       "mac":"…","mode":"auto","agent_framework":"…","os":"…","locale":"…"}'

curl -sS -X POST "$BASE/api/v1/voucher/lock" \
  -H "X-Skill-Version: $SKILL_VER" -H 'Content-Type: application/json' \
  -d '{"claim_id":"…"}'

curl -sS -X POST "$BASE/api/v1/claim/complete" \
  -H "X-Skill-Version: $SKILL_VER" -H 'Content-Type: application/json' \
  -d '{"claim_id":"…","outcome":"ok"}'
```

### Code-handoff calls

```bash
# 1. Optimistic dispense — works directly if this installation is already bound
curl -sS -X POST "$BASE/api/v1/voucher/dispense" \
  -H "X-Skill-Version: $SKILL_VER" -H 'Content-Type: application/json' \
  -d '{"deal_id":"…","installation_id":"…","agent_framework":"…","os":"…"}'
#  → 200 voucher.code + instructions[]   (done)
#  → 403 phone_not_bound  → first time on this device, do the OTP steps below

# 2. (first time only) one-time phone verification
curl -sS -X POST "$BASE/api/v1/verify/send-otp" \
  -H "X-Skill-Version: $SKILL_VER" -H 'Content-Type: application/json' \
  -d '{"phone":"+15551234567","installation_id":"…"}'
curl -sS -X POST "$BASE/api/v1/verify/check-otp" \
  -H 'Content-Type: application/json' \
  -d '{"phone":"+15551234567","code":"123456","installation_id":"…"}'   # → verification_token

# 3. (first time only) retry dispense with phone + token to bind and get the code
curl -sS -X POST "$BASE/api/v1/voucher/dispense" \
  -H "X-Skill-Version: $SKILL_VER" -H 'Content-Type: application/json' \
  -d '{"deal_id":"…","installation_id":"…","phone":"+15551234567",
       "verification_token":"…","agent_framework":"…","os":"…"}'
#  → voucher.code + instructions[]  (show the user verbatim)
```

`$BASE` = `https://freetoday.ai`.

### Version handling

Every successful response body carries `skill_meta`:

```json
{"skill_meta": {"your_version":"2026.05.31","latest":"2026.06.15",
  "min_required":"2026.05.31","update_available":true,"update_required":false,
  "notes_url":"https://github.com/colakang/freetoday-skills/releases"}}
```

- `update_required: true` → the server already returned **426** with an upgrade command
  in `detail.message`. Show it verbatim and stop.
- `update_available: true` → soft notice. Tell the user once per session: "📦 freetoday
  update available ({{latest}}) — run `git -C <skill-dir> pull` when convenient", where
  `<skill-dir>` is wherever this skill is installed. Then continue.
- otherwise → silent.

## Installation identity

The server needs a stable per-install key to enforce one-redemption-per-day. On first
run:

1. Look for `installation_id` under the host's config dir — `$XDG_CONFIG_HOME/freetoday/`
   if set, else `~/.config/freetoday/` (POSIX) or `%APPDATA%\freetoday\` (Windows).
2. If present, read the UUID and send it as `installation_id` on every call.
3. If absent, generate a UUIDv4 (`uuidgen` / `python3 -c 'import uuid;print(uuid.uuid4())'`
   / language equivalent), create the parent dir, write the file `chmod 600`.

This file is the only persistent state the skill writes. Never overwrite an existing
one — that's the user's to reset, and the server still catches a second same-day
redemption via the phone + fingerprint keys anyway.

## MAC detection (optional fingerprint)

Alongside phone, the server can use a device MAC for a stronger per-user fingerprint.
Best-effort detect the first non-loopback MAC:

| OS | Command |
|---|---|
| macOS | `ifconfig en0 \| awk '/ether/{print $2}'` |
| Linux | `cat /sys/class/net/$(ls /sys/class/net \| grep -E '^(en\|eth)' \| head -1)/address` |
| Linux/macOS fallback | `ip link show 2>/dev/null \| awk '/link\/ether/{print $2; exit}'` |
| Windows | `getmac /v /fo csv` (first connected adapter) |

If you can't get one (docker, sandbox, locked-down host), pass `mac: null` or omit it —
the server degrades gracefully to phone-only fingerprinting. Don't fabricate or
pre-hash a MAC; send the raw value and let the server hash it.

## Executing a playbook

`claim/start` returns a playbook — an ordered list of steps the agent runs against the
merchant site:

```json
{"playbook_id":"…","merchant_id":"…","version":1,
 "steps":[
   {"action":"navigate","url":"…"},
   {"action":"wait_for","selector":"…","timeout_ms":10000},
   {"action":"ask_user","prompt":"Enter your phone","var":"phone"},
   {"action":"fill","selector":"…","value":"{{phone}}"}
 ]}
```

Loop over `steps`, keeping a local `vars` dict. Per action:

- **`navigate`** — open `step.url`.
- **`wait_for`** — wait for `step.selector` (default 8000ms); on timeout → `selector_miss`.
- **`click`** — click `step.selector`.
- **`fill`** — substitute `{{var}}` in `step.value`, then type it into `step.selector`.
- **`extract`** — read `step.selector`'s text into `vars[step.var]`.
- **`ask_user`** — pause; ask `step.prompt` verbatim in chat; store the reply in
  `vars[step.var]`. If `step.mask`, warn the user their input may be visible in
  transcript — don't claim you can hide it.
- **`confirm_with_user`** — render `step.message` (with substitution); wait for an
  explicit yes/no. No → abort + `claim/complete outcome="user_aborted"`.
- **`acquire_voucher`** — `POST /api/v1/voucher/lock {claim_id}`; load
  `vars["voucher_code"/"voucher_description"/"voucher_kind"]`. On 503 →
  `claim/complete outcome="voucher_pool_empty"` and stop.

A `__TBD_…` selector means the flow isn't mapped yet — stop, tell the user, and
`claim/complete outcome="other"` with notes on where you stopped.

Map each action to your host's browser tool per
[`references/browser-tool-mapping.md`](references/browser-tool-mapping.md).

## Handling sensitive input

Phone and OTP values are sensitive. Keep them in `vars` only as long as needed; don't
write them to disk or logs, and discard after the redemption completes. If the host
archives transcripts you can't prevent that, but don't add to it. The phone goes to the
deals-api for verification + rate-limit dedup; never quote it back to the user
needlessly.

## Confirming before placing the order

Auto-redeem playbooks include a `confirm_with_user` right before the final order click.
**Honor it** — never click through without an explicit yes. On no, abort and
`claim/complete outcome="user_aborted"`.

## Error recovery (auto-redeem)

| Failure | Action |
|---|---|
| `wait_for` times out | Retry once; still failing → `claim/complete outcome="selector_miss"` + `step_index`. Tell the user the site likely changed; operator will refresh the playbook. |
| CAPTCHA appears | Stop, tell the user honestly, `outcome="captcha"`. |
| Login fails (wrong OTP) | Re-ask; after two fails → `outcome="login_fail"`. |
| Network/API 5xx | Backoff retry (2s, 5s); after 3 → `outcome="other"`. |

Always include a short `notes` on non-`ok` outcomes — the operator uses them to fix the
playbook server-side.

## References

- [`references/api-contract.md`](references/api-contract.md) — endpoint shapes, statuses, examples
- [`references/playbook-schema.md`](references/playbook-schema.md) — step-action semantics + variable rules
- [`references/browser-tool-mapping.md`](references/browser-tool-mapping.md) — map actions to playwright-cli / camoufox-cli / Chrome MCP / built-in browse
- [`references/merchants/cotti.md`](references/merchants/cotti.md) — Cotti specifics (pickup-only, NYC)
