# freetoday

[![Star on GitHub](https://img.shields.io/github/stars/colakang/freetoday-skills?style=social)](https://github.com/colakang/freetoday-skills/stargazers)
[![License: AGPL-3.0](https://img.shields.io/badge/license-AGPL--3.0-blue)](LICENSE)
![Version](https://img.shields.io/badge/skill-2026.06.01.1-brightgreen)

An agent skill that finds today's free or deeply-discounted food & drinks near you and
either **redeems them for you automatically** (driving the merchant's site) or **hands
you a voucher code** to redeem yourself after a quick phone verification.

> **⭐ Found a free coffee through this? Star the repo and send it to a friend.**
> Every star helps more partners come on board — which means more free stuff for
> everyone. Two seconds: [**star it**](https://github.com/colakang/freetoday-skills) and
> share `https://deal.echo365.ai` with someone who'd use it.

Works in **any agent that can make HTTP requests** — Claude Code, Claude Desktop,
Codex, Cursor, OpenClaw, or your own. A browser-automation tool is *optional*: it's
only needed for hands-free auto-redeem. Without one, the skill falls back to the
code-handoff flow.

## What this is

A thin instruction package (`SKILL.md` + `references/`). The deals catalog and the
per-merchant redemption **playbooks** live behind an API at `https://deal.echo365.ai`,
so merchant flows can change without you re-installing anything.

## Install

The skill is just a folder of Markdown. "Installing" means putting that folder where
your agent looks for skills. The one-liner auto-detects the right place:

```bash
curl -fsSL https://deal.echo365.ai/install.sh | sh
```

Or clone it yourself into the location your agent uses:

### Claude Code

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/colakang/freetoday-skills.git ~/.claude/skills/freetoday
```
Restart the session; trigger with e.g. "any free coffee near me?".

### Codex (OpenAI Codex CLI / IDE)

Codex discovers project instructions through `AGENTS.md`. Clone the skill anywhere,
then point Codex at it by referencing the path in your `AGENTS.md` (project root or
`~/.codex/AGENTS.md` for global):

```bash
git clone https://github.com/colakang/freetoday-skills.git ~/.codex/skills/freetoday
```
```markdown
<!-- in AGENTS.md -->
## freetoday
When the user asks about free/discounted food or drink deals, follow the instructions
in ~/.codex/skills/freetoday/SKILL.md.
```

### Cursor

```bash
git clone https://github.com/colakang/freetoday-skills.git ~/.cursor/freetoday
```
Add a rule under `.cursor/rules/` that points at `~/.cursor/freetoday/SKILL.md`, or
paste the SKILL.md path into your project rules.

### OpenClaw

```bash
git clone https://github.com/colakang/freetoday-skills.git ~/.openclaw/workspace/skills/freetoday
```

### Any other agent

Drop the cloned folder wherever your agent auto-discovers skills, or just tell your
agent to "read and follow `<path>/SKILL.md`". The skill only needs:
- **HTTP capability** (curl, fetch, requests, an MCP HTTP tool — any one). Required.
- **A browser-automation tool** (`playwright-cli`, `camoufox-cli`, Chrome MCP, or a
  built-in `browse_url`). Optional — only for auto-redeem.
- **File write** under `~/.config/freetoday/` (one small UUID file).

## What the skill does

1. Asks for your ZIP or city, fetches matching deals.
2. Asks how you want it: **auto-redeem** (it drives the merchant site) or **just the
   code** (you redeem yourself).
3. Auto-redeem → walks a server-provided playbook in your browser, confirming with you
   before placing the order.
   Code handoff → verifies your phone with a one-time SMS code, then hands you the
   voucher code + steps.

Rate limit: **one redemption per merchant per day**, enforced server-side on your
installation id + a hash of your phone + a device fingerprint. Deleting the local id
file won't get you a second one.

## Repo layout

```
SKILL.md                  entrypoint — workflow, the two redemption paths, executor
VERSION                   date-style version, sent as X-Skill-Version on every call
references/
  api-contract.md         every endpoint's request/response shape + status codes
  playbook-schema.md      step-action semantics + variable substitution
  browser-tool-mapping.md map actions → playwright-cli / camoufox-cli / Chrome MCP / browse_url
  merchants/cotti.md      Cotti-specific notes (Flutter Web app, CAPTCHA handoff)
evals/evals.json          trigger eval prompts
```

## See also

- Companion campaign: Cotti × OpenClaw NY Tech Week 2026 booth.

## Spread the word

freetoday grows by word of mouth — no ads, no growth hacks. If it got you something free:

- ⭐ **Star this repo** — it's the signal partners look at before they sign on.
- 📣 **Share it** — send `https://deal.echo365.ai` to a friend, or post the deal you found.
- 🛠️ **Bring a merchant** — if you run (or know) a spot that wants to give a little away,
  open an issue. New cities and merchants are how this gets useful outside NYC.

## License

AGPL-3.0 — see [LICENSE](LICENSE). Copyright (c) 2026 freetoday.ai.
If you fork it or run a modified version as a network service, AGPL requires you to
offer your corresponding source to that service's users.
