# Merchant: Cotti Coffee (US)

`merchant_id: cotti` | Timezone: `America/New_York` | Day boundary: midnight ET

## Site

`https://mobile.us.cotticoffee.global/#/pages/tabs`

Vue/uniapp SPA with hash-mode routing — the `#/` fragment matters. Everything is client-rendered; you can't get useful HTML from a plain HTTP GET. Whichever browser tool you use must execute JS.

## Login

Phone + SMS OTP. There is no email/password path; if the user can't receive SMS, they can't complete the flow. Surface this honestly before starting.

The phone number the user gives at login is the same one the skill sends to the deals-api as the rate-limit key — that's by design, so trying a different installation_id won't get a second drink the same day.

## Order model — pickup only

There is no delivery option in the US Cotti app. The skill places an order at the user's selected NYC store; the user must walk in to collect. The final `confirm_with_user` step in the playbook surfaces the store address — make sure the user actually reads it.

## Voucher / coupon redemption

Booth winners arrive with a code shaped like `COTTI-Q{1-5}-XXXXXX` (issued by the `cotti-codes` sidecar at the NY booth). The playbook's `ask_user` step for `code` captures it; it's then filled into the in-app voucher screen.

The voucher applies a 100% discount on a specific SKU (designated by the operator). The user does NOT need a credit card on file for a $0 order — but if Cotti adds card-on-file requirements in future, the playbook will start failing at the final `place_order` step. That's a server-side fix; the skill itself shouldn't try to handle adding a card.

## Known quirks / gotchas

1. **Hash routing**: navigation between in-app screens often updates only the `#/...` fragment without a full page load. `wait_for` selectors on the destination view, not on a URL match.
2. **SSR shell**: the initial HTML is a near-empty `<div id="app">` until JS runs. `wait_for` an actual content node (`__TBD_app_root__` until mapped) before any other action.
3. **Lazy-loaded sub-routes**: account/voucher screens may chunk-load on first visit. `wait_for` timeout should be generous (10s+) for first-time users.
4. **Cloudflare in front?**: not confirmed. If you see a `Just a moment...` interstitial, that's CF Turnstile — surface as `outcome="captcha"`.

## Playbook lineage

| Version | Notes |
|---|---|
| v1 | Placeholder selectors (`__TBD_*__`). Skill will stop at first one, complete with `outcome="other"`. Real selectors mapped on first browser session against Cotti — operator updates server-side via `PUT /admin/playbook/cotti-pickup-v1`. |

## Cross-references

- `cotti-codes` sidecar (server-side): mints `COTTI-Qn-XXXXXX` redemption codes at the NYC booth. Codes are valid in the Cotti app's voucher screen. There is no live cross-call from deals-api to cotti-codes at the moment — the user just types in the code they were given.
- Booth Flowise chatflow: `09c3de37-5cd8-401d-9cc5-198f5404dca0` on `https://deal.echo365.ai/api/v1/prediction/...` (the upstream funnel — irrelevant to skill execution, mentioned for context).
