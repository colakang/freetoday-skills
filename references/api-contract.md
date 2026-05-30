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
      "created_at": "2026-05-29 20:57:42"
    }
  ]
}
```

**400** — neither `zip` nor `city` given.

## `GET /api/v1/deals/{deal_id}`

Single-deal lookup, same shape as one element of the list above. **404** if unknown.

## `POST /api/v1/claim/start`

Reserves a same-day claim slot. Computes three rate-limit keys server-side: `installation_id`, `phone_hash`, and `agent_uid_hash` = sha256(server_pepper, mac, phone). Any same-merchant-same-day match in `('started','succeeded')` blocks.

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
    "requires_voucher": true
  }
}
```

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
  "swept_stale_locks": 0
}
```

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
