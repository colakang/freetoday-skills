# deals-api contract

Base URL: `https://deal.echo365.ai/api/v1`. JSON in, JSON out. Public endpoints have no auth; admin endpoints require `X-Admin-Key` and are not invoked by the skill.

## `GET /api/v1/deals?zip=10001` or `?city=NYC`

Lists active deals matching the user's location.

- One of `zip` or `city` is required. ZIP is matched by prefix against each deal's `zip_prefixes`; `city` is matched case-insensitive against `city_keywords` (substring either way).
- A deal with `zip_prefixes` containing `*` is matched anywhere.

**200**
```json
{
  "count": 1,
  "deals": [
    {
      "deal_id": "cotti-nyc-free-drink-booth-2026",
      "merchant_id": "cotti",
      "title": "Free drink at Cotti Coffee NYC",
      "description": "...",
      "zip_prefixes": ["100", "101", "102", "..."],
      "city_keywords": ["new york", "nyc", "manhattan", "brooklyn", "queens"],
      "expires_at": null,
      "playbook_id": "cotti-pickup-v1",
      "requires_code": true,
      "active": true,
      "redeem_modes": ["auto", "code"],
      "stores": [
        {"name": "Cotti Coffee – Penn Station", "address": "345 7th Ave Retail B, New York, NY 10001"}
      ],
      "created_at": "2026-05-29 20:57:42"
    }
  ]
}
```

- **`redeem_modes`** — which redemption paths this deal supports: `"auto"` (Mode A, the
  skill drives the merchant site) and/or `"code"` (Mode B, code-handoff). A code-only
  deal (e.g. an in-store partner with no app automation) returns `["code"]`; offer only
  that path and don't ask "auto or code?".
- **`stores`** — physical redemption location(s). Show these to the user; for a
  code-only/in-store deal this is where they bring the code.

**400** — neither `zip` nor `city` given.

## `GET /api/v1/deals/{deal_id}`

Single-deal lookup, same shape as one element of the list above. **404** if unknown.

## `POST /api/v1/claim/start`

Opens an auto-redeem attempt. This is **non-committal** — starting (and even abandoning) an auto-redeem does NOT consume your one-per-merchant-per-day slot. The slot is only taken later, at `voucher/lock` (the redeem step), so a stalled or abandoned attempt never blocks a retry or the code-handoff fallback.

Computes three rate-limit keys server-side: `installation_id`, `phone_hash`, and `agent_uid_hash` = sha256(server_pepper, mac, phone). It returns **409 only if a *real* claim already exists today** — i.e. one that has already locked a voucher or succeeded. A prior voucher-less attempt from the same installation is silently superseded.

**Request body**
```json
{
  "deal_id": "cotti-nyc-free-drink-booth-2026",
  "installation_id": "ea45a889-0edd-46e9-92c1-375663dfd04c",
  "phone": "+15551234567",
  "mac":   "aa:bb:cc:11:22:33"
}
```

- `installation_id`: 8-64 chars. The UUID written under `~/.config/freetoday/installation_id` on first run.
- `phone`: optional but strongly recommended — feeds both `phone_hash` and `agent_uid_hash`. Effectively required for any merchant whose playbook logs in by phone (all current merchants).
- `mac`: optional. Host's first non-loopback interface MAC. Combined with `phone` + server pepper into `agent_uid_hash`. If the skill can't detect a MAC (docker, sandbox, hardened OS), pass `null` or omit — server falls back to phone-only, defense degrades gracefully.

**200**
```json
{
  "claim_id": "Q1b2FQP9MwGz2xgKIfCHAg",
  "claimed_date": "2026-05-29",
  "merchant_tz": "America/New_York",
  "playbook": {
    "playbook_id": "cotti-pickup-v1",
    "merchant_id": "cotti",
    "version": 2,
    "steps": [ /* see playbook-schema.md */ ],
    "notes": "..."
  },
  "deal_meta": {
    "deal_id": "cotti-nyc-free-drink-booth-2026",
    "title": "Free drink at Cotti Coffee NYC",
    "requires_voucher": true,
    "redeem_modes": ["auto", "code"],
    "stores": [{"name": "Cotti Coffee – Penn Station", "address": "345 7th Ave Retail B, New York, NY 10001"}]
  }
}
```

**409 — mode_not_supported** (you sent `mode:"auto"` for a deal whose `redeem_modes` is `["code"]`)
```json
{"detail": {"error": "mode_not_supported", "requested_mode": "auto", "supported_modes": ["code"], "deal_id": "...", "message": "...Use the code-handoff path (voucher/dispense)."}}
```
Switch to a path in `supported_modes`. For `["code"]`, use `voucher/dispense`.

**The response does NOT include a voucher.** Voucher allocation is lazy: call `POST /api/v1/voucher/lock` only when the playbook reaches the `acquire_voucher` step. This keeps the merchant pool intact for users who abort before the redemption screen.

Hold `claim_id` — every subsequent endpoint needs it.

**409 — already claimed today**
```json
{
  "detail": {
    "error": "already_claimed_today",
    "merchant_id": "cotti",
    "claimed_date": "2026-05-29",
    "merchant_tz": "America/New_York",
    "existing_claim_id": "...",
    "existing_status": "succeeded",
    "message": "You already claimed a deal from cotti today (2026-05-29, America/New_York). One claim per merchant per day."
  }
}
```

Tell the user when the merchant clock rolls over; don't retry.

**404** — unknown `deal_id` or its playbook is missing.

## `POST /api/v1/voucher/lock`

Lazy voucher allocation. Call this AT the `acquire_voucher` step in the playbook — not before.

**Request body**
```json
{"claim_id": "Q1b2FQP9MwGz2xgKIfCHAg"}
```

Server flow:
1. Looks up the claim (404 if missing, 409 if not in `started` state).
2. Sweeps stale `locked` vouchers (>30 min old) for this merchant back to `created`.
3. Allocates the lowest-id `created` voucher for the merchant.
4. Marks it `locked`, links to this `claim_id`.
5. Returns the voucher details.

**200**
```json
{
  "voucher_id": 1,
  "code": "aBcDeF1234",
  "kind": "代金券",
  "description": "$0 For Drinks",
  "usage_limit": 1,
  "usage_count": 0,
  "discount_type": "full",
  "discount_value": null,
  "swept_stale_locks": 0
}
```

For shared vouchers (`usage_limit > 1`), this still returns the voucher even when multiple parallel users got the same code — the server doesn't enforce exclusivity for shared codes, just tracks usage_count which is incremented at claim/complete time.

Idempotent: calling again with the same `claim_id` returns the same voucher with `noop: true, prior: true`.

After getting the response, the skill seeds:
```
vars["voucher_code"]        = response.code
vars["voucher_description"] = response.description
vars["voucher_kind"]        = response.kind
```

**503 — voucher pool empty**
```json
{
  "detail": {
    "error": "voucher_pool_empty",
    "merchant_id": "cotti",
    "message": "Sorry — cotti vouchers are sold out for now. The operator needs to load a fresh batch. Try again later."
  }
}
```

Abort the playbook. Call `claim/complete` with `outcome="voucher_pool_empty"` so the server can keep a clean audit trail. Tell the user honestly that the pool is empty.

**404** — claim not found. The skill messed up (lost claim_id) or sent a wrong one.

**409 — wrong state**
```json
{"detail": {"error": "wrong_state", "claim_status": "succeeded", "message": "claim is not in 'started' state"}}
```

The claim is already completed — too late to lock a voucher for it.

**409 — already_claimed_today**
```json
{"detail": {"error": "already_claimed_today", "merchant_id": "cotti", "claimed_date": "2026-06-01", "existing_claim_id": "...", "existing_status": "succeeded", "message": "..."}}
```

This is the real daily-slot gate. It fires here (not at `claim/start`) because the slot is only consumed when a voucher is actually allocated. Means another redemption — e.g. a parallel code-handoff — already took today's slot for this user. Don't retry; tell the user when the merchant clock rolls over and stop.

## `POST /api/v1/verify/send-otp`  (Mode B only)

Generate a 6-digit OTP for the phone, send via Twilio (or stdout-stub when Twilio env vars unset).

**Request body**: `{phone, installation_id}`

**200**
```json
{
  "ok": true,
  "expires_in_seconds": 300,
  "sms_mode": "twilio",
  "dev_mode_otp": "924294",
  "warning": "Twilio not configured; OTP returned in response for dev mode only. ..."
}
```

`dev_mode_otp` and `warning` are ONLY present when the server is running without Twilio credentials. Production deployments will omit both — the user must read the OTP from their actual SMS inbox.

## `POST /api/v1/verify/check-otp`  (Mode B only)

Validate the OTP. On success returns a single-use `verification_token` valid for ~5 minutes; the skill passes it to `/voucher/dispense`.

**Request body**: `{phone, code, installation_id}`

**200**: `{ok: true, verification_token: "<opaque>", ttl_seconds: 300}`

**400** wrong code (returns `attempts_used` / `max_attempts`).
**404** no pending OTP — call /send-otp first.
**410** OTP expired.
**429** too many attempts; restart from /send-otp.

## `POST /api/v1/voucher/dispense`  (code-handoff path)

Single-call combo: claim + voucher allocation + completion. Used when the user redeems
themselves. `phone` + `verification_token` are **only needed the first time** an
installation dispenses (to bind a phone). Once bound, send just `installation_id` —
the per-installation/merchant/day limit is the gate; no OTP is re-required.

**Recommended call pattern:** try with `installation_id` alone first; on **403
phone_not_bound**, run the one-time OTP flow and retry with `phone` + `verification_token`.

**Request body** (`phone` + `verification_token` optional once bound)
```json
{
  "deal_id": "cotti-nyc-free-drink-booth-2026",
  "installation_id": "<uuid>",
  "phone": "+15551234567",              // first-time only
  "verification_token": "<from /verify/check-otp>",  // first-time only
  "mac": "<optional>",
  "agent_framework": "claude-desktop",
  "os": "macos",
  "locale": "en-US",
  "email": "<optional, consent-based>"
}
```

**200**
```json
{
  "claim_id": "...",
  "claimed_date": "2026-05-30",
  "merchant_tz": "America/New_York",
  "deal_meta": {"deal_id": "...", "title": "...", "merchant_id": "cotti"},
  "voucher": {
    "code": "aBcDeF1234",
    "kind": "代金券",
    "description": "$0 For Drinks",
    "usage_limit": 1,
    "discount_type": "full",
    "discount_value": null
  },
  "stores": [
    {"name": "Cotti Coffee – Penn Station", "address": "345 7th Ave Retail B, New York, NY 10001"}
  ],
  "instructions": [
    "Open the Cotti app (https://mobile.us.cotticoffee.global) and log in with the phone you just verified",
    "Go to 我的 → 券码兑换 and enter the code: aBcDeF1234",
    "Tap 兑换 → 确认兑换 — the voucher lands in your 代金券 pocket",
    "Redeemable in-store at: Cotti Coffee – Penn Station — 345 7th Ave Retail B, New York, NY 10001"
  ]
}
```

**Show the user `voucher.code`, the `instructions` array, and `stores` verbatim.** Don't paraphrase — the steps are merchant-specific and may be in-app (Cotti) or "show the code at the counter" (an in-store partner like TEAPULSE). The instructions always end with the redemption store(s).

**401** bad verification_token (first-time only — token unrecognized or doesn't match the phone). Restart from send-otp.
**403** `phone_not_bound` — first redemption on this installation; supply `phone` + `verification_token` (run send-otp + check-otp first). Or `phone_binding_mismatch` — a `phone` was supplied that differs from the one this installation is already bound to; omit `phone` to use the bound one, or this is a different user.
**409** already_claimed_today (same shape as claim/start).
**409** mode_not_supported — this deal is auto-only (`redeem_modes` lacks `"code"`); drive the playbook via claim/start instead.
**503** voucher_pool_empty or daily_quota_exhausted.
**426** upgrade_required.

> Binding is established once — here (code-handoff first time) or automatically on a
> successful auto-redeem (`claim/complete outcome="ok"`, where the merchant's own phone
> login proved ownership). It is never re-verified on later redemptions; the
> one-per-merchant-per-day rate-limit is the abuse control.

## `POST /api/v1/claim/complete`

Finalizes the claim with the outcome. Drives voucher state-machine transitions.

**Request body**
```json
{
  "claim_id": "Q1b2FQP9MwGz2xgKIfCHAg",
  "outcome": "ok",
  "step_index": null,
  "notes": null
}
```

`outcome` values:
- `ok` — order/voucher confirmation extracted and shown to user. Maps to `status="succeeded"`. The voucher (if locked) is marked `used`.
- `selector_miss` — `wait_for`/`click` element wasn't there. Include `step_index`. Voucher (if locked) stays `locked` — operator reviews via `/admin/vouchers/{id}/release`.
- `captcha` — site presented a CAPTCHA the skill can't solve. Voucher releases back to `created` (we know nothing was redeemed merchant-side).
- `login_fail` — couldn't authenticate (e.g. OTP failed twice). Voucher stays `locked` (ambiguous: maybe Cotti partially recorded the attempt).
- `user_aborted` — user said no at a `confirm_with_user` step. Voucher releases back to `created`.
- `voucher_pool_empty` — server told us no voucher available. Voucher state unchanged (none was locked).
- `other` — anything else; explain in `notes`. Voucher stays `locked` for operator review.

Voucher state outcomes (assuming a voucher was locked):

| outcome | voucher status after |
|---|---|
| `ok` | `used` |
| `user_aborted`, `captcha`, `voucher_pool_empty` | `created` (released) |
| `selector_miss`, `login_fail`, `other` | `locked` (sweep recycles after 30 min, or operator manual release) |

**200**
```json
{"ok": true, "claim_id": "...", "status": "succeeded"}  // or "failed"
```

Idempotent retry:
```json
{"ok": true, "noop": true, "prior_status": "succeeded"}
```

**404** — `claim_id` unknown.

## Error model

FastAPI default: errors come back as `{"detail": <message or object>}` with the appropriate HTTP status. Don't try to parse the detail as a fixed shape — log it and surface it to the user verbatim.

## Why no Bearer / API key on the public endpoints?

Read-only access (the catalog) is genuinely public — just a list of deals. Claim creation requires `installation_id` which is per-machine and locally generated; we don't pretend that's an authentication system. The rate-limit is the only meaningful gatekeeper — three keys (installation_id, phone_hash, agent_uid_hash). This keeps the skill simple to ship to customers who shouldn't have to provision credentials before they can do anything.
