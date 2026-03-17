# Ticket Pipeline: ACP Session Instructions

When dispatching an ACP session to handle a ticket, include these instructions:

## ACP Session Task Template

Use this as the `task` field in `sessions_spawn`:

```
You are handling ticket <TICKET-ID> for <AGENT_NAME>.

**Issue:** <GITHUB_ISSUE_URL>
**Repo:** Clone from GitHub as needed (use GITHUB_TOKEN from workspace .env for auth)
**Git identity:** <BOT_NAME> <<BOT_EMAIL>>

## Step 1: Understand the ticket
- Fetch the full issue from GitHub API
- If there are screenshot URLs in the issue body, analyze them
- Read the relevant files in the codebase to understand the context

## Step 2: Post analysis comment
Post a comment on the GitHub issue with:
- What the bug/feature is (from screenshots + description)
- Which files/components are affected
- Your proposed approach

## Step 3: Implement
- For TEST tickets (body contains "test" or no real bug described): make a trivial commit (add a comment line to a relevant file)
- For real tickets: implement the fix with tests
- Branch name: fix/<ticket-id>-<short-description> or feat/<ticket-id>-<short-description>
- Commit message: "fix: <description> (<TICKET-ID>)" or "feat: ..."

## Step 4: Open PR
- Push branch
- Open PR via GitHub API referencing the issue
- PR title: "<type>: <description> (<TICKET-ID>)"
- PR body: "Closes #<issue-number>\n\n<brief description of changes>"

## Step 5: Notify
After opening PR, use sessions_send tool:
- target: label "main"
- message: "🏭 <TICKET-ID> PR ready: <PR_URL> — <one-line summary>"
```

## Spawning an ACP Session

Use `sessions_spawn` with these parameters:

```json
{
  "runtime": "acp",
  "agentId": "claude",
  "mode": "run",
  "task": "<task template from above>"
}
```

**Key points:**
- `runtime: "acp"` — uses the ACP backend (acpx)
- `agentId: "claude"` — spawns a Claude Code session
- `mode: "run"` — one-shot session (fire-and-forget)
- No need to wait — ACP sessions run asynchronously

## Repo Management

ACP sessions have full access to the filesystem. They can clone repos on demand:

```bash
cd workspace/repos
git clone https://<BOT_USERNAME>:${GITHUB_TOKEN}@github.com/<ORG>/<REPO>.git
cd <REPO>
git config user.name "<BOT_NAME>"
git config user.email "<BOT_EMAIL>"
```

## State Tracking

Update `memory/support-tickets.json` after each dispatch (from the cron, before spawning):

```json
[
  {
    "ticket_id": "TD-0015",
    "email_from": "user@example.com",
    "issue_url": "https://github.com/org/repo/issues/2",
    "pr_url": null,
    "status": "dispatched",
    "created_at": "2026-03-14T23:00:00Z"
  }
]
```

Status values: `dispatched` → `pr_open` → `merged`

The ACP session will update `pr_url` when it opens the PR.

## Notification Flow

1. Cron announces dispatch to Telegram (reliable)
2. ACP session runs Claude Code to handle the ticket
3. ACP session uses `sessions_send` to notify when PR is ready (best-effort)

TinyDesk (or equivalent) handles user-facing notifications on merge. No need to email the submitter.
