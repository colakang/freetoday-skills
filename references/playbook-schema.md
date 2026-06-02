# Playbook schema

A playbook is an ordered list of `steps` returned by `POST /api/v1/claim/start`. Each step is a JSON object with an `action` string and action-specific fields. The skill executes them in order, maintaining a local `vars: dict[str, str]` for state passed between steps.

There are exactly **8 action types**. The set is intentionally small so any browser tool can implement it (7 of them are browser actions; the 8th, `acquire_voucher`, is an API call the skill makes mid-flow).

## Variable substitution

Any string field whose value contains `{{name}}` has those tokens replaced from `vars` before the step runs. If `name` isn't in `vars`, leave the literal `{{name}}` in place and surface the bug at `claim/complete` with `outcome="other"`.

Substitution happens on `fill.value`, `confirm_with_user.message`, and (defensively) on `navigate.url`. It does NOT happen on selectors — those are taken literally.

## Voucher vars (populated by `acquire_voucher`)

When the playbook reaches an `acquire_voucher` step, the skill calls `POST /api/v1/voucher/lock` and seeds these into `vars`:

| `vars` key | Source |
|---|---|
| `voucher_code` | `response.code` |
| `voucher_description` | `response.description` |
| `voucher_kind` | `response.kind` |

Playbook authors use `{{voucher_code}}` etc. in subsequent `fill` / `confirm_with_user` steps. The skill does NOT ask the user for a voucher code — the server pool allocates it lazily, only when the playbook actually reaches the redemption step.

## Selectors

By default, selectors are CSS. The skill also passes Playwright-style accessibility selectors verbatim — `role=button[name="…"]`, `role=textbox[name="…"]`, `text="…"` — because they're the most portable cross-tool way to point at modern apps' a11y trees (Flutter Web, React with Material, etc.).

Picking which to use is up to the playbook author; the executor is expected to pass whatever it gets to its underlying tool. Chrome MCP, camoufox-cli, and playwright-cli all accept this syntax. For unsupported syntaxes, the tool falls back to its own a11y APIs.

For Flutter Web apps specifically (Cotti, see `merchants/cotti.md`), CSS will not work — the page renders to a canvas, and `role=…` is the only viable form. The browser tool may need to click the `<flt-semantics-placeholder>` element to populate the a11y tree first; most tools do this implicitly when they request an accessibility snapshot.

### Localized selectors (`selector_en`, `intent`)

Many merchant apps render in the **browser's language**, so an accessibility `name` is locale-dependent — a US/English-locale user sees different label text than the playbook author captured. To stay robust, a step may carry, alongside `selector`:

- **`selector_en`** — the same selector with the element's **English** accessible name.
- **`intent`** — a short human description of the target element.

Resolution order when present:
1. Try `selector` (the primary, often non-English name).
2. If it doesn't match, try `selector_en`.
3. If neither name is present in the live a11y tree, locate the element by **`intent`** (role + meaning) — match the element that plays that role, whatever its displayed language.

When a step has none of these, use `selector` literally.

**Speak the user's language.** `confirm_with_user` / `ask_user` prompts may quote on-screen text in more than one language (e.g. `'Continue' / '继续'`). Show the user the wording that matches the language you actually observe on screen; don't read them a label in a language their UI isn't using.

## Actions

### `navigate`
```json
{"action": "navigate", "url": "https://..."}
```
Open the URL in the current browser tab. The skill assumes there is exactly one tab in play; don't open a new one unless the host's tool only supports new tabs.

### `wait_for`
```json
{"action": "wait_for", "selector": "...", "timeout_ms": 8000}
```
Block until `selector` matches at least one DOM element. Default timeout 8000ms. On timeout, raise → `outcome="selector_miss"`.

Selectors are CSS by default. Some browser tools (Playwright) extend this with `>>`-chained pseudo-selectors like `button:has-text("Login")` — those are fine to pass through verbatim if the tool supports them.

### `click`
```json
{"action": "click", "selector": "..."}
```
Click the matched element. If multiple match, click the first. On miss → `outcome="selector_miss"`.

### `fill`
```json
{"action": "fill", "selector": "...", "value": "{{phone}}"}
```
Type `value` (after `{{var}}` substitution) into the element. For input/textarea, set value and dispatch `input` + `change`. For contenteditable, type-as-keys. Tool-specific guidance in `browser-tool-mapping.md`.

### `extract`
```json
{"action": "extract", "selector": "...", "var": "drink_name"}
```
Read `.innerText` (or equivalent) of the matched element, strip it, and store into `vars[var]`. On miss → `outcome="selector_miss"`.

### `ask_user`
```json
{"action": "ask_user", "prompt": "Enter your phone number", "var": "phone", "mask": false}
```
Pause and ask the user in the chat using `prompt` verbatim (don't paraphrase or "clean it up" — the prompt is authored to be unambiguous). Store the reply into `vars[var]`.

`mask: true` is a hint — the host agent should warn the user that what they type may be visible in transcript. Don't lie; you cannot guarantee masking in chat.

If the user supplies obvious garbage (empty string, "n/a", "skip"), abort with `outcome="user_aborted"`.

### `confirm_with_user`
```json
{"action": "confirm_with_user", "message": "Order will be placed for {{drink_name}}. Confirm?"}
```
Render `message` (with substitution) to the user and wait for explicit yes/no. Yes → continue. No → abort with `outcome="user_aborted"`.

This action MUST appear at least once before the final order-placing click; merchant playbooks are authored with that in mind. Don't skip it even if it feels redundant.

### `acquire_voucher`
```json
{"action": "acquire_voucher"}
```
The only non-browser action. The skill calls `POST /api/v1/voucher/lock` with the current `claim_id`. The server allocates a merchant voucher from the pool, locks it, and returns `{code, kind, description, voucher_id}`. The skill populates `vars["voucher_code"]`, `vars["voucher_description"]`, `vars["voucher_kind"]`.

The action takes no parameters — server uses the claim's stored merchant_id to pick from the right pool.

On error:
- **503 voucher_pool_empty** → abort with `claim/complete outcome="voucher_pool_empty"`.
- **409 wrong_state** → claim is already completed; abort with `outcome="other", notes="lock-after-complete bug"`.

Idempotent: calling twice on the same `claim_id` returns the same voucher (response includes `noop: true`). Useful if the skill retries the step after a transient network hiccup.

Place this step in the playbook RIGHT BEFORE the first `fill` that uses `{{voucher_code}}`. Don't put it at the start of the playbook — that would allocate vouchers for users who abort before the redemption screen.

## Placeholder selectors (`__TBD_*__`)

If the playbook contains selectors starting with `__TBD_` (literal placeholder, e.g. `"__TBD_login_button__"`), it means the merchant flow isn't fully mapped yet. Don't try to guess what they refer to. Stop at the first `__TBD_` selector, complete the claim with `outcome="other"` and notes describing the user-visible step you stopped at, and tell the user the playbook is still in progress.

## Error handling at the step level

- One retry per step is acceptable for `wait_for` and `click` (transient SPA hydration).
- More than one retry is a smell — the selector probably moved. Fail loud.
- A `wait_for` that succeeded once doesn't need to be re-asserted before a subsequent `click` to the same selector — but if the page navigated in between, do re-assert.

## A walk-through

```
vars = {}
for i, step in enumerate(playbook.steps):
    a = step["action"]
    if a == "navigate":   tool.goto(sub(step["url"]))
    elif a == "wait_for": tool.wait(step["selector"], step.get("timeout_ms", 8000))
    elif a == "click":    tool.click(step["selector"])
    elif a == "fill":     tool.fill(step["selector"], sub(step["value"]))
    elif a == "extract":  vars[step["var"]] = tool.text(step["selector"]).strip()
    elif a == "ask_user": vars[step["var"]] = ask(step["prompt"])
    elif a == "confirm_with_user":
        if not yes_no(sub(step["message"])):
            return complete(claim_id, "user_aborted", step_index=i)
    elif a == "acquire_voucher":
        r = api_post(f"/api/v1/voucher/lock", {"claim_id": claim_id})
        if r.status == 503:
            return complete(claim_id, "voucher_pool_empty", step_index=i, notes=r.json()["detail"]["message"])
        v = r.json()
        vars["voucher_code"]        = v["code"]
        vars["voucher_description"] = v["description"]
        vars["voucher_kind"]        = v["kind"]
    else:
        return complete(claim_id, "other", step_index=i, notes=f"unknown action {a}")
return complete(claim_id, "ok")
```

## What's intentionally NOT here

- No `eval_js` / `evaluate` — keeps playbooks portable and auditable.
- No `screenshot` — host tools have their own (use them for debugging but don't model them as playbook state).
- No `iframe` / `frame_switch` — if a merchant uses an iframe, the playbook author should put the iframe selector in front (e.g. `iframe[name=checkout] >> button.pay`).
- No `select_option` — most modern SPAs use clickable list items; use `click` chains.
- No `upload_file` — out of scope.
- No `solve_captcha` — out of scope; surface to user.

These omissions are deliberate. Adding more action types is the operator's call (server-side) — keep the skill simple.
