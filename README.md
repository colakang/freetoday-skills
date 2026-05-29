# deal-today

Public Claude skill that finds today's free or deeply discounted food & drink deals near the user and auto-redeems them at the merchant by driving the merchant's website to place a pickup order.

## What this is

A thin agent shell. The skill itself is just instructions (`SKILL.md` + `references/`). The actual deals catalog and per-merchant ordering **playbooks** live behind an API at `https://deal.echo365.ai`. Merchant flows change without touching the skill — the operator updates the playbook server-side.

## Install

### Claude Code

Clone into your skills directory:

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/colakang/deal-today-skills.git ~/.claude/skills/deal-today
```

Restart your session. The skill will trigger on prompts like "any free coffee near me?" or "redeem this COTTI-Q3-8HRMXJ code for me".

### OpenClaw

```bash
git clone https://github.com/colakang/deal-today-skills.git ~/.openclaw/workspace/skills/deal-today
```

### Other Claude-compatible agents

Anywhere a skill directory is auto-discovered, drop the cloned folder in. The skill needs nothing more than:
- A browser tool (any of: `playwright-cli`, `camoufox-cli`, Chrome MCP, built-in `browse_url`)
- HTTP capability (curl, fetch, requests, MCP HTTP — any one)
- File-write to `~/.config/deal-today/installation_id`

## What the skill does

1. Asks you for your ZIP or city.
2. Fetches the daily deals list from `https://deal.echo365.ai/api/v1/deals`.
3. On your pick, reserves a same-day claim and gets back the playbook.
4. Walks the playbook step by step in your browser, asking you for things like phone, OTP, and your booth code where needed.
5. Confirms with you before placing the order.

Rate limit: **one redemption per merchant per day** (the merchant's local day — for Cotti NYC that's midnight ET). Enforced server-side by both your installation UUID and a hash of your phone number — deleting the local UUID file won't get you a second drink.

## Repo layout

```
SKILL.md                          # entrypoint (~130 lines, < 400-line skill-creator budget)
references/
  api-contract.md                 # full API shapes
  playbook-schema.md              # the 7 step actions + variable substitution
  browser-tool-mapping.md         # playwright / camoufox / Chrome MCP / fallback adapters
  merchants/cotti.md              # Cotti-specific notes (pickup-only, NYC, booth funnel)
evals/                            # trigger eval prompts (skill-creator format)
```

## See also

- Backend repo: `colakang/deal-today-web` — deals-api, nginx, EC2 layout
- Companion campaign: Cotti × OpenClaw NYC booth chatflow (issues the redemption codes this skill spends)

## License

Private — not yet open-sourced. Future MIT, probably.
