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
      "title": "Free drink at Cotti Coffee NYC (booth winners)",
      "description": "Won a COTTI-Qn-XXXXXX code at the OpenClaw × Cotti NYC booth? ...",
      "zip_prefixes": ["100", "101", "102", "103", "104", "110", "111", "112", "113", "114", "115", "116"],
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

Reserves a same-day claim slot for this `(installation_id, merchant)` pair (and `(phone_hash, merchant)` if phone supplied) and returns the playbook to execute.

**Request body**
```json
{
  "deal_id": "cotti-nyc-free-drink-booth-2026",
  "installation_id": "ea45a889-0edd-46e9-92c1-375663dfd04c",
  "phone": "+15551234567"
}
```

- `installation_id`: 8-64 chars. The UUID written under `~/.config/freetoday/installation_id` on first run.
- `phone`: optional. If present, server normalizes (digits only) and hashes with a server-only pepper before storage. Required to enforce the phone-based rate-limit; effectively required for any merchant whose playbook will log in by phone (i.e. all current merchants).

**200**
```json
{
  "claim_id": "GStUSGVajrycflbCww3svA",
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
    "title": "Free drink at Cotti Coffee NYC (booth winners)",
    "requires_code": true
  },
  "voucher": {                    // only present when the deal requires one
    "code": "aBcDeF1234",         // the merchant's real voucher code, allocated from the pool
    "kind": "代金券",              // free-text label from the merchant (e.g. 代金券, discount, free_item)
    "description": "$0 For Drinks"
  }
}
```

Hold `claim_id` — `claim/complete` needs it.

When `voucher` is present, **seed it into your playbook vars before stepping**:

```
vars["voucher_code"]        = response.voucher.code
vars["voucher_description"] = response.voucher.description
vars["voucher_kind"]        = response.voucher.kind
```

Playbook steps reference these via `{{voucher_code}}` etc. The skill does NOT ask the user for a voucher code — the server allocated it. The voucher is reserved server-side; either the success completion marks it `used`, or a failed completion keeps it `reserved` for operator review.

**409 — already claimed today**
```json
{
  "detail": {
    "error": "already_claimed_today",
    "merchant_id": "cotti",
    "claimed_date": "2026-05-29",
    "merchant_tz": "America/New_York",
    "existing_claim_id": "LHVdSdNFv2w-...",
    "existing_status": "succeeded",
    "message": "You already claimed a deal from cotti today (2026-05-29, America/New_York). One claim per merchant per day."
  }
}
```

When the user sees this, tell them the merchant's local date that already counts against them and roughly when midnight in that timezone is. Don't retry; the answer won't change until the merchant's clock rolls over.

**404** — unknown `deal_id` or the deal references a missing playbook.

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

Only happens for deals that require a voucher when the merchant's pool is empty. Tell the user honestly; suggest they retry later.

## `POST /api/v1/claim/complete`

Finalizes the claim with the outcome.

**Request body**
```json
{
  "claim_id": "GStUSGVajrycflbCww3svA",
  "outcome": "ok",
  "step_index": null,
  "notes": null
}
```

`outcome` values:
- `ok` — order confirmation extracted and shown to user. Maps to `status="succeeded"` server-side. This is the ONLY outcome that counts as a successful redemption.
- `selector_miss` — `wait_for` or click hit an element that wasn't there. Include `step_index`.
- `captcha` — site presented a CAPTCHA.
- `login_fail` — couldn't authenticate the user (e.g. wrong OTP twice).
- `user_aborted` — user said no at a `confirm_with_user` step or otherwise asked to stop.
- `other` — anything else; explain in `notes`.

Anything other than `ok` maps to `status="failed"` server-side, releases the rate-limit slot for a retry **only if the failure was server-side** (this nuance is enforced by the unique index — failed `started` claims don't block, succeeded ones do).

**200**
```json
{"ok": true, "claim_id": "...", "status": "succeeded"}  // or "failed"
```

If the claim was already completed (idempotent retry):
```json
{"ok": true, "noop": true, "prior_status": "succeeded"}
```

**404** — `claim_id` unknown.

## Error model

FastAPI default: errors come back as `{"detail": <message or object>}` with the appropriate HTTP status. Don't try to parse the detail as a fixed shape — log it and surface it to the user verbatim.

## Why no Bearer / API key on the public endpoints?

Read-only access (the catalog) is genuinely public — it's just a list of deals. Claim creation requires an installation_id which is per-machine and locally generated; we don't pretend that's an authentication system. The rate-limit is the only meaningful gatekeeper, and it's defense-in-depth (install_id + phone_hash), not user authentication. This keeps the skill simple to ship to customers who shouldn't have to provision credentials before they can do anything.
