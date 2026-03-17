# Cron Job Prompt Template

Use this as the `--message` for the `email-check` cron job. Customize the placeholders.

**CRITICAL: Fire-and-forget pattern.** The cron must spawn ACP sessions and exit immediately. Do NOT wait for sessions to complete. Announce dispatches BEFORE spawning — the cron's reply is what gets delivered to Telegram.

```
You are <AGENT_NAME>. Check support email AND poll GitHub for missed ticket assignments. FIRE-AND-FORGET: spawn ACP sessions and exit immediately. Do NOT wait for results.

## Step 1: Check email
Connect to <IMAP_HOST>:993 as <EMAIL_USER> (password: EMAIL_PASSWORD from workspace .env).
Fetch all UNSEEN emails. Mark them all read immediately.
For each ticket-assignment email from the bot account (subject contains "assigned"): extract the GitHub issue URL and add to a "to_process" list.

## Step 2: Poll GitHub for missed assignments (safety net)
Using GITHUB_TOKEN from workspace .env, call:
  GET https://api.github.com/search/issues?q=assignee:<BOT_GITHUB_USERNAME>+state:open+is:issue&per_page=50

For each result, check if already actioned (ALL must be true to skip):
- Present in memory/support-tickets.json (matched by issue_url)?
- Does <BOT_GITHUB_USERNAME> already have a comment on the issue?
- Is there an open or closed PR referencing the issue?

If NOT actioned → add to "to_process" list.

## Step 3: ANNOUNCE FIRST (before spawning)
In YOUR reply, announce what you found and will dispatch:
- "📧 Email: N tickets found"
- "🔍 GitHub: N issues found"
- "🎫 Dispatching: <issue URLs>"
This announcement is what gets delivered to Telegram.

## Step 4: Spawn ACP sessions (fire-and-forget)
For EACH issue in "to_process":
- Use sessions_spawn with:
  - runtime: acp
  - agentId: claude
  - mode: run
  - task: Full ticket handling instructions (analyze issue, explore codebase, post analysis comment, create branch, implement fix, push, open PR, notify human)
- DO NOT wait for results. Just spawn and move on.
- Add issue to memory/support-tickets.json with status "dispatched"

## Step 5: Log and exit
Append to memory/heartbeat-log.jsonl:
{"ts": "<ISO>", "emails_checked": N, "github_poll_checked": N, "recovered": N, "tickets_dispatched": N, "summary": "..."}

Exit immediately. Do not wait for ACP sessions.

If nothing found anywhere: reply HEARTBEAT_OK
```

## Placeholders to Replace

| Placeholder | Example |
|---|---|
| `<AGENT_NAME>` | `alpha-4` |
| `<IMAP_HOST>` | `imap.ionos.com` |
| `<EMAIL_USER>` | `teamalpha@z1bc.io` |
| `<BOT_GITHUB_USERNAME>` | `z-team-alpha` |

## Cron Add Command

```bash
openclaw cron add \
  --name "email-check" \
  --every 15m \
  --session isolated \
  --announce \
  --to <TELEGRAM_CHAT_ID> \
  --timeout-seconds 120 \
  --description "Check email + GitHub for ticket assignments" \
  --message "$(cat /path/to/cron-prompt.txt)"
```

**Never use `--channel last`** — it doesn't resolve in isolated sessions.
