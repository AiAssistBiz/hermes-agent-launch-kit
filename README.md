# Automated Workforce Starter Kit
### Deploy your own Hermes Agent — a persistent AI agent that watches, drafts, and acts for you

This is a practical setup guide for standing up [Hermes Agent](https://www.nousresearch.com/) (built by Nous Research) as a **24/7 automated workforce agent** — something that runs on a cloud server instead of your laptop, connects to the messaging apps you already use, and can monitor sources, draft outputs, and act on them through a human-approval gate.

This kit documents the exact deploy → configure → connect → automate path, including the gotchas that aren't in the official docs.

---

## What you're building

A four-stage pipeline:

**Scan → Draft → Approve → Act**

1. **Scan** — Hermes watches a data source on a schedule (a job board, an RSS feed, a listings site, an inbox — anything reachable via API or scraping)
2. **Draft** — when it finds something matching your criteria, it drafts a response, summary, or action using your model provider of choice
3. **Approve** — the draft lands in your messaging app (Slack, Discord, Telegram, email) for a human yes/no before anything goes out
4. **Act** — on approval, Hermes executes the next step (send, submit, file, post)

The human-approval gate is the whole point — this isn't a "set it and forget it" bot, it's a force multiplier that still keeps you in the loop on anything that leaves the building.

---

## Prerequisites

- A [DigitalOcean](https://www.digitalocean.com/) account (or any VPS — this guide uses DO's 1-click app because it's the fastest path)
- An API key for a model provider (Nous Portal, OpenAI, Anthropic, or any provider Hermes supports — a free tier is enough to start)
- A messaging platform account you're willing to connect: Slack, Discord, Telegram, WhatsApp, Signal, or email
- Basic comfort with SSH — you'll need it for one-time configuration

**Why a cloud server instead of a spare laptop:** it's tempting to run this on hardware you already own, but a laptop that sleeps or loses power kills the gateway connection silently — it'll still *look* like it's running. A small droplet ($12–24/mo, 2vCPU/2GB is plenty to start) stays up and avoids that entire failure mode.

---

## Step 1 — Deploy the droplet

1. Go to the DigitalOcean Marketplace and find **Hermes Agent** (search "hermesagent")
2. Click "Create Hermes Agent Droplet," pick a region close to you, and choose at least a 2GB RAM plan
3. Wait for provisioning to finish — the dashboard often says "ready" before SSH is actually available, so give it 2–3 extra minutes if your first connection attempt fails

Prefer the CLI/API? One command does it:
```bash
curl -X POST -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer '$TOKEN'' -d \
  '{"name":"your-agent-name","region":"nyc3","size":"s-2vcpu-4gb","image":"hermesagent"}' \
  "https://api.digitalocean.com/v2/droplets"
```

---

## Step 2 — Configure the agent

SSH into the droplet. Hermes runs under a dedicated `hermes` user, not root:

```bash
ssh hermes@your-droplet-ip
```

Config lives at:
```
/home/hermes/.hermes/
```
— **not** `/root/.hermes/`. This trips people up because most self-hosted tooling defaults to root; Hermes doesn't.

Inside that directory:
- Set your model provider in the config (a free-tier provider is fine to start — you can upgrade later without re-deploying)
- Set your terminal/execution backend — Docker is the simplest default and is pre-installed on the marketplace image
- Drop your credentials (API keys, bot tokens) into `.env` in the same directory

---

## Step 3 — Connect a messaging gateway (Slack, in full)

This is how you'll actually talk to the agent and receive drafts for approval. Slack takes the most setup of any platform, so here's the complete backend wiring — not just "create an app."

**Fastest path — let Hermes generate the manifest for you:**
```bash
hermes slack manifest --agent-view --write
```
This writes `~/.hermes/slack-manifest.json` with every required OAuth scope, event subscription, and slash command already declared. Go to [api.slack.com/apps](https://api.slack.com/apps) → **Create New App** → **From an app manifest**, pick your workspace, paste the file contents, and click through to create. That's scopes and events done in one step — skip to "Install & tokens" below.

**Manual path, if you want to see every piece:**

*Bot Token Scopes* (Features → OAuth & Permissions → Bot Token Scopes):
| Scope | Why it's required |
|---|---|
| `chat:write` | Send messages as the bot |
| `app_mentions:read` | Detect @mentions in channels |
| `channels:history`, `groups:history` | **Without these the bot only works in DMs** — this is the #1 setup miss |
| `im:history`, `im:read`, `im:write` | Direct messages |
| `mpim:history`, `mpim:read` | Group DMs |
| `users:read` | Look up user info |
| `files:read`, `files:write` | Read/send attachments |

*Event Subscriptions* (Features → Event Subscriptions → Subscribe to bot events) — all required:
`message.im`, `message.mpim`, `message.channels`, `message.groups`, `app_mention`

*Socket Mode* (Settings → Socket Mode → Enable ON): generates an App-Level Token with the `connections:write` scope — this is your `SLACK_APP_TOKEN` (starts with `xapp-`). Socket Mode means no public URL is needed; the connection is outbound WebSocket only.

*App Home* (Features → App Home → Show Tabs): turn **Messages Tab** ON, or Slack blocks all DMs to the bot regardless of scopes/events being correct.

**Install & tokens:**
1. Settings → Install App → Install to Workspace → copy the **Bot User OAuth Token** (`xoxb-...`) — this is `SLACK_BOT_TOKEN`
2. Get your own Slack Member ID (click your name → View full profile → ⋮ → Copy member ID) — looks like `U01ABC2DEF3`

**Wire it into Hermes** — add to `/home/hermes/.hermes/.env`:
```bash
SLACK_BOT_TOKEN=xoxb-your-bot-token-here
SLACK_APP_TOKEN=xapp-your-app-token-here
SLACK_ALLOWED_USERS=U01ABC2DEF3          # comma-separated Member IDs — without this, the gateway denies everyone by default
SLACK_HOME_CHANNEL=C01234567890          # optional: where cron/scheduled results land
```
Then start the gateway:
```bash
hermes gateway install       # runs as a persistent user service
hermes gateway status        # verify it connected
```
Finally, invite the bot to any channel you want it active in — it does not auto-join:
```
/invite @Hermes Agent
```

**If scopes/events change later:** you must reinstall the app from the Install App page for the change to take effect — this is the single most common "I fixed it but it's still broken" trap.

Discord, Telegram, WhatsApp, Signal, and email gateways follow the same shape — platform-specific token(s) into `.env`, then `hermes gateway install` — but with far less setup surface than Slack.

---

## Step 4 — Known gotchas (save yourself the debugging time)

**"Gateway shows Ready but doesn't respond in chat"**
This happens after the host machine sleeps or is power-cycled — the gateway holds a stale socket connection that doesn't auto-recover on its own. `hermes gateway status` will lie to you and say it's fine.
Fix:
```bash
hermes gateway stop
hermes gateway start
```
This is the single most common issue reported with self-hosted setups — and it's a big part of why Step 1 recommends a cloud droplet over a laptop, since droplets don't sleep.

**Cron jobs that work locally but silently fail over SSH**
If you're scheduling scan jobs with cron, local and SSH environments often resolve different paths (especially for virtual environments or non-standard installs). Always use absolute paths in cron entries, and test the exact cron command manually over SSH before trusting the schedule.

---

## Step 5 — Install the workflow skill

Hermes uses **Skills** — reusable, on-demand instruction files (the open `agentskills.io` standard, portable to Claude Code, Cursor, and Codex too) — instead of stuffing everything into one system prompt. Skills live at `~/.hermes/skills/<category>/<skill-name>/SKILL.md` and load progressively, only when relevant.

This kit ships a ready-made skill for the scan → draft → approve → act pattern: **`skills/automated-workflow/SKILL.md`** (in the same repo as this README). To install it:

```bash
mkdir -p ~/.hermes/skills/automation/automated-workflow
cp skills/automated-workflow/SKILL.md ~/.hermes/skills/automation/automated-workflow/
hermes chat -q "/new"    # start a fresh session so the skill index picks it up
```

Or point Hermes at the repo directly as a **tap** (a shared skill source others can pull too):
```bash
hermes skills tap add your-github-username/automated-workforce-kit
```

Verify it loads:
```bash
hermes chat --toolsets skills -q "What skills do you have?"
```

## Step 6 — Build your first workflow

With the gateway live and the skill installed, the pattern is the same regardless of what you're automating — the skill file encodes exactly this:

1. **Pick one source** to scan — start narrow. One job board, one RSS feed, one inbox label. Don't try to cover everything on day one.
2. **Write the scan job** as a scheduled task inside Hermes (`hermes cron add`, or via an external scraping tool like Firecrawl if the source needs JS rendering) that checks the source on an interval and filters for what actually matters to you.
3. **Draft on a match** — when the scan finds something, have Hermes draft the output (a reply, a summary, an application) using your model provider and post it into your connected chat.
4. **Gate it** — nothing sends, submits, or posts without an explicit approval reply in chat. This is what keeps a 2AM false positive from becoming a problem.
5. **Log outcomes** — keep a simple record of what got drafted, approved, or rejected. This is what lets you tighten the filter over time instead of re-litigating the same bad match every week.

Start with something low-stakes (a summary digest, a draft-only workflow with no auto-send) before wiring up anything that acts on your behalf without a person confirming first.

---

## Resources

- Hermes Agent docs: https://docs.digitalocean.com/products/marketplace/catalog/hermes-agent/
- DigitalOcean tutorial — full walkthrough including MCP server setup: https://www.digitalocean.com/community/tutorials/how-to-run-hermes-agent
- Nous Research: https://www.nousresearch.com/

---

*Put together from a real deployment — the gotchas above cost real debugging time so you don't have to spend yours on them.*
