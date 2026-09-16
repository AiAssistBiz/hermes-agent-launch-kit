---
name: automated-workflow
description: Runs a scan-draft-approve-act loop against a chosen data source — checks it on a schedule, drafts a response on a match, and only sends/submits/posts after explicit human approval in chat. Use whenever the user wants a recurring "watch X and draft Y" automation with a human-in-the-loop gate before anything goes out.
version: 1.0.0
author: automated-workforce-kit
tags: [automation, cron, approval-workflow, monitoring]
---

# Automated Workflow: Scan → Draft → Approve → Act

## When to Use

Trigger this skill when the user asks to:
- monitor a source (job board, RSS feed, inbox, listings site, API) on a recurring basis
- draft a response, application, or summary automatically when something matches their criteria
- require a human approval step before anything is sent, submitted, or posted
- keep a running log of what was drafted, approved, or rejected

Do not use this skill for one-off tasks — it is specifically for **recurring, scheduled** monitoring with a **persistent approval gate**.

## Setup

Before running this skill for a new source, confirm with the user:
1. **The source** — URL, API, or feed to check, and how to access it (public API, requires an API key, or needs scraping)
2. **The match criteria** — what makes something worth drafting a response to (be specific; vague criteria produce noisy digests)
3. **The draft type** — reply, summary, application, or other output format
4. **The delivery channel** — which connected messaging platform (Slack, Discord, Telegram, etc.) and which channel/DM should receive drafts for approval
5. **The action on approval** — what "act" actually means for this source (send an email, submit a form, post a reply, file a record) and whether that action itself needs a script/tool or can be done manually after approval

## Procedure

1. **Scan.** Set up a scheduled cron job (`hermes cron add`) that checks the source on the agreed interval. For sources needing JS rendering or heavier scraping, use an external tool (e.g. Firecrawl) rather than a raw fetch.
2. **Filter.** Apply the match criteria before drafting anything — the goal is a short, high-signal list, not a firehose. If nothing matches on a run, skip the draft/deliver steps entirely; don't send empty digests.
3. **Draft.** For each match, draft the output using the configured model provider. Keep drafts self-contained — include enough context (source link, key details) that the human can approve/reject without leaving the chat.
4. **Deliver for approval.** Post each draft to the configured channel using `send_message`. Never skip this step — the agent must not act on a match without a corresponding approval message existing first.
5. **Gate on approval.** Wait for an explicit approval reply (e.g. "approve", a ✅ reaction if `reaction_triggers` is enabled, or a button tap in a `clarify` prompt) before executing the action. A non-response is not an approval — do not act on silence or timeout.
6. **Act.** On approval, execute the action (send, submit, post, file). On rejection, log it and stop — do not re-draft the same match automatically.
7. **Log.** Record source, match, draft, decision (approved/rejected/no response), and timestamp somewhere durable (a file, a sheet, a database row) so the filter criteria can be tightened over time based on real outcomes.

## Pitfalls

- **Filter too broad → approval fatigue.** If the human starts rubber-stamping everything without reading, the criteria need tightening, not the approval step removing.
- **Acting without a delivered approval message.** Always confirm step 4 actually posted before treating a match as "pending approval" — a failed send should never silently fall through to auto-act.
- **Re-drafting rejected matches.** Once a match is rejected, exclude it from future scans (by ID, URL, or content hash) so it doesn't reappear on the next run.
- **Cron path mismatches.** Scheduled jobs run in a different environment than an interactive session — always use absolute paths and test the exact cron command over SSH before trusting the schedule.
- **Treating "gateway looks up" as "gateway is working."** After any host sleep/restart, verify the gateway actually round-trips a message, not just that `status` reports ready.

## Verification

After setting up a new source, confirm:
- [ ] A test run correctly finds zero matches on a source with nothing new (no false positives)
- [ ] A known matching item produces a draft delivered to the correct channel
- [ ] An explicit approval reply triggers the action, and the action actually completes
- [ ] An explicit rejection stops the action and is logged
- [ ] Re-running the scan does not re-surface an already-decided match
