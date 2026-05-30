# freetoday

Public Claude skill that finds today's free or deeply discounted food & drink deals near the user and auto-redeems them at the merchant by driving the merchant's website to place a pickup order.

## What this is

A thin agent shell. The skill itself is just instructions (`SKILL.md` + `references/`). The actual deals catalog and per-merchant ordering **playbooks** live behind an API at `https://deal.echo365.ai`. Merchant flows change without touching the skill — the operator updates the playbook server-side.

## Install

### Claude Code

Clone into your skills directory:

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/colakang/freetoday-skills.git ~/.claude/skills/freetoday
```

Restart your session. The skill will trigger on prompts like "any free coffee near me?" or "anything free today?".

### OpenClaw

```bash
git clone https://github.com/colakang/freetoday-skills.git ~/.openclaw/workspace/skills/freetoday
```

### Other Claude-compatible agents

Anywhere a skill directory is auto-discovered, drop the cloned folder in. The skill needs nothing more than:
- A browser tool (any of: `playwright-cli`, `camoufox-cli`, Chrome MCP, built-in `browse_url`)
- HTTP capability (curl, fetch, requests, MCP HTTP — any one)
- File-write to `~/.config/freetoday/installation_id`

## What the skill does

1. Asks you for your ZIP or city.
2. Fetches the daily deals list from `https://deal.echo365.ai/api/v1/deals`.
3. On your pick, reserves a same-day claim and gets back the playbook + a server-allocated voucher code from the merchant's pool.
4. Walks the playbook step by step in your browser, asking you for things like phone and OTP where needed.
5. Confirms with you before redeeming.

Rate limit: **one redemption per merchant per day** (the merchant's local day — for Cotti NYC that's midnight ET). Enforced server-side by both your installation UUID and a hash of your phone number — deleting the local UUID file won't get you a second drink.

## Repo layout

```
SKILL.md                          # entrypoint (under skill-creator's 400-line budget) — workflow, voucher seeding, playbook executor
README.md                         # this file
.gitignore
evals/
  evals.json                      # trigger eval prompts (skill-creator format)
references/
  api-contract.md                 # full deals-api shapes
  playbook-schema.md              # the 7 step actions + variable substitution + role+name selectors
  browser-tool-mapping.md         # playwright / camoufox / Chrome MCP / fallback adapters
  merchants/cotti.md              # Cotti-specific notes (Flutter Web, CAPTCHA, voucher format)
```

## See also

- Backend repo: `colakang/deal-today-web` — deals-api, nginx, EC2 layout, voucher pool, admin endpoints
- Companion campaign: Cotti × OpenClaw NYC booth chatflow (issues attribution codes that point users to this skill)

## License

MIT — see [LICENSE](LICENSE). Copyright (c) 2026 freetoday.ai.
