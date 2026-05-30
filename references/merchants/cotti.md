# Merchant: Cotti Coffee (US)

`merchant_id: cotti` | Timezone: `America/New_York` | Day boundary: midnight ET

## Site

`https://mobile.us.cotticoffee.global/#/pages/tabs`

**This is a Flutter Web app, not Vue/uniapp.** That changes how you drive it:

- The page is rendered into a `<canvas>` element. **CSS selectors on inner elements don't work** — there are no inner elements as far as the DOM is concerned.
- Flutter exposes a separate accessibility (a11y) tree that mirrors the rendered UI. Your browser tool MUST use that tree (Playwright's `getByRole`, Chrome MCP's accessibility snapshot, etc.) to identify widgets — not CSS.
- The a11y tree is gated behind a `<flt-semantics-placeholder role="button" aria-label="Enable accessibility">` element. Most browser automation tools auto-fire the click that enables it when they take an accessibility snapshot; if your tool doesn't, you need to `.click()` that placeholder element via JS before the tree populates. Once populated, ref-based clicks work.

## Selector strategy for the playbook

All `selector` fields in the Cotti playbook use **Playwright-style role+name syntax**:

```
role=button[name="继续"]
role=textbox[name="邮箱或手机号码"]
role=button[name="🇺🇸 United States +1 美国"]
```

These map cleanly to:
- Playwright: native `getByRole(...)` / accepts the string verbatim
- Chrome MCP: equivalent `aria-role` + `aria-label` matcher in the snapshot
- camoufox-cli: same Playwright syntax (camoufox is Playwright-flavored)

For unnamed widgets (the agreement toggle, for example), the playbook falls back to coordinate-relative descriptions in `confirm_with_user` steps that hand off to the user.

## Login

Phone + SMS OTP only. No email/password, no Apple/Google social path that works without app context. Cotti's flow:

1. **Country picker** — first visit only; we pre-set USA. Skill encodes this as steps that no-op on repeat visits (Cotti remembers in localStorage).
2. **Legal accept** — first visit only; same caveat.
3. **Tabs page → 我的 → "Sign up/Login"** — the button is unnamed in the a11y tree, but it's the only tap target near the "未登录" text.
4. **Country code selector** — defaults to **🇨🇳 +86**. We MUST change to 🇺🇸 +1 or Cotti rejects the number.
5. **Phone + agreement toggle**, then **继续**.
6. **CAPTCHA** — see next section.
7. **SMS OTP** — 6 digits, valid for ~60s.
8. **Auto-registration if unknown phone** — Cotti shows "手机号未注册 / 是否重新注册为新账号？" Tap **注册新账号** to complete the flow. This is the registration path, not just login — there's no separate sign-up form.

Once logged in, session persists in browser storage. Subsequent skill runs on the same browser profile skip the entire login flow.

## CAPTCHA

After tapping 继续 on the phone form, Cotti shows a **slider puzzle**: "拖动滑块使曲线匹配" — drag a slider to align two curves. This is a Tencent-style image-match captcha.

- It cannot be solved automatically by the skill. It needs computer vision + drag physics that no general-purpose browser tool ships with.
- The playbook handles this with a `confirm_with_user` step right after the 继续 click: "Solve the slider captcha in the browser window. Press OK once the SMS has been sent." The user does it manually.
- Cotti sometimes skips the captcha on repeat sends — if the playbook's `wait_for` for the OTP input succeeds quickly, the user got lucky and the `confirm_with_user` becomes a one-tap acknowledgment.

This is the single biggest UX wart in the Cotti playbook. There's no good fix; it's a deliberate Cotti design choice and we work around it.

## Voucher code format

**Important:** Cotti's voucher screen accepts **Cotti's own 10-character mixed-case codes** (e.g. `aBcDeF1234`). It does NOT accept the `COTTI-Qn-XXXXXX` codes that the OpenClaw booth issues via the `cotti-codes` sidecar.

What this means for the booth funnel:

- The booth chatflow issues a `COTTI-Q3-8HRMXJ` code as proof-of-win. That's an OpenClaw-side tracking token.
- The actual free drink is delivered via a Cotti-issued voucher code (provided by Cotti as part of the partnership). The operator distributes those vouchers via a separate channel (e.g. emailed to booth winners, or shown alongside the COTTI-Qn code).
- The freetoday skill no longer asks the user for a voucher code — the server pre-allocates one from the `merchant_vouchers` pool (loaded by the operator from Cotti's promo system) and passes it in the `claim/start` response. The skill fills `{{voucher_code}}` automatically.

If you're updating the operator-side process so that COTTI-Qn codes are pre-loaded into Cotti's promo system as aliases (which would let users redeem with either format), update this section and the playbook's `ask_user` prompt accordingly.

## Order model — pickup only

There is no delivery option in the US Cotti app. Voucher redemption credits the account; the user then taps "Order Now" to attach the voucher to an actual order and pick it up at a chosen store. v1 of the playbook stops at "voucher in account" — the order-placement step is a separate v2 playbook (TBD).

## Known quirks

1. **Hash routing**: navigation between in-app screens updates only the `#/...` fragment. `wait_for` on destination widgets, not on URL match.
2. **localStorage state**: `~33` keys after one full login flow. Persisting browser profile means most steps no-op on repeat runs.
3. **The "Enable accessibility" placeholder** is the literal entry point to making Flutter introspectable. If the playbook's first `wait_for` hangs, check that the a11y tree has actually been populated.
4. **Voucher count display lag**: immediately after a successful redemption, "代金券 N" on 我的 page doesn't always refresh. A page reload (or navigating away and back) shows the correct count. Don't treat the stale "0" as a redemption failure — the modal "您正在兑换以下商品" → "确认兑换" → confirmation toast IS the source of truth.

## Playbook lineage

| Version | Notes |
|---|---|
| v1 | Placeholder skeleton with `__TBD_*__` selectors (never lived in production). |
| v2 | First real selectors mapped from a manual flow on 2026-05-29. Uses Playwright-style role+name selectors, hybrid auto+manual (skill drives what it can, hands off to user at CAPTCHA). Voucher pre-allocated at claim/start. |
| v3 | Adds the `acquire_voucher` step (lazy allocation via `POST /api/v1/voucher/lock`). Voucher is locked only when the playbook actually reaches the Cotti 券码兑换 screen — users who abort during login don't burn vouchers. Stops at "voucher credited" — no auto-order. |

## Cross-references

- `cotti-codes` sidecar (server-side at `127.0.0.1:8090` on ai-relay): mints `COTTI-Qn-XXXXXX` codes at the NYC booth. Separate from Cotti's voucher system; only used for booth-attribution / counter handover.
- Booth Flowise chatflow: `09c3de37-5cd8-401d-9cc5-198f5404dca0` — issues the COTTI-Qn codes during the quiz.
