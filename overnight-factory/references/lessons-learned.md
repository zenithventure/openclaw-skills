# Lessons Learned in Production

Real failure modes encountered and their fixes.

## ACP vs Direct Claude Code

**Recommendation:** Use ACP (via acpx) instead of spawning Claude Code directly.

**Why ACP is better:**
- No need to manage Claude Code installation path
- acpx handles the Claude Code adapter automatically (`@zed-industries/claude-agent-acp`)
- Consistent interface for multiple coding agents (claude, codex, gemini)
- Built-in session management and state tracking

**How to use:**
```json
{
  "runtime": "acp",
  "agentId": "claude",
  "mode": "run",
  "task": "..."
}
```

**Not:**
```bash
claude -p --dangerously-skip-permissions --output-format text "..."
```

## WebSocket CLI↔Gateway Connection Failure

**Problem:** `openclaw cron add` and other CLI commands fail with "gateway closed (1000)" or "handshake timeout".
**Symptom:** Gateway is running (HTTP works, Telegram works), but CLI commands fail to connect via WebSocket.
**Cause:** WebSocket handshake between CLI and gateway times out after ~3 seconds. Root cause unclear — may be resource/binding issue on some hosts.
**Fix:** Edit cron jobs directly:
1. Stop gateway: `systemctl --user stop openclaw-gateway`
2. Edit `~/.openclaw/cron/jobs.json` manually
3. Restart gateway: `systemctl --user start openclaw-gateway`
4. Verify in logs: `tail -f /tmp/openclaw/openclaw-*.log | grep cron`
**Note:** This only affects CLI↔Gateway communication. Cron jobs, Telegram, and gateway core functions work fine.

## Cron jobs.json Parse Errors

**Problem:** Cron fails to start with "Failed to parse cron store" error.
**Symptom:** `SyntaxError: JSON5: invalid character '\n' at line X` in gateway logs.
**Cause:** Literal newlines in the `message` string field. OpenClaw uses JSON5 parser which doesn't accept actual newlines in string values.
**Fix:** Use `\n` escape sequences in JSON strings, not literal newlines:
```json
"message": "Line 1\nLine 2\nLine 3"
```
**Not:**
```json
"message": "Line 1
Line 2
Line 3"
```

## Manual Cron Job State

**Problem:** Cron job timer keeps re-arming but never fires.
**Symptom:** Logs show `cron: timer armed` every minute but `nextAt` never changes, job never executes.
**Cause:** Manually setting `state.nextRunAtMs` in jobs.json can confuse the scheduler.
**Fix:** When creating jobs manually, omit the `state` object entirely. OpenClaw will initialize it correctly on first load.

## Cron Delivery

**Problem:** Cron runs successfully but notifications never reach Telegram.
**Symptom:** `lastDelivered: false`, `lastDeliveryStatus: "not-delivered"` on every run.
**Cause:** `--channel last` doesn't resolve in isolated sessions — no "last" channel exists.
**Fix:** Use `--to <explicit-telegram-chat-id>` when creating/editing the cron.
**Note:** `lastDelivered: false` on empty-inbox runs is EXPECTED (HEARTBEAT_OK is intentionally silent).

## Cron Timeouts

**Problem:** Cron errors out on every run with `consecutiveErrors: 5+`.
**Symptom:** `lastError: "cron: job execution timed out"`, `lastDurationMs` equals your timeout exactly.
**Cause:** Cron prompt instructs it to wait for ACP sessions to complete, which takes 10-30+ minutes.
**Fix:** Fire-and-forget design — spawn ACP sessions without waiting, exit immediately. 120s timeout is sufficient.

## Error Backoff

**Problem:** After fixing a cron issue, it doesn't run for 30-60 minutes.
**Cause:** Exponential backoff after consecutive errors. Edit doesn't reset error count.
**Fix:** Delete and recreate: `openclaw cron delete <id> && openclaw cron add ...`

## ACP Session Notification

**Problem:** ACP session completes and opens PR, but no Telegram notification arrives.
**Cause:** ACP sessions spawned from isolated cron sessions may not reliably call back to main session.
**Fix:** Have the cron itself announce dispatches in its own reply (before spawning). The cron reply is what reliably delivers to Telegram. ACP session notification via `sessions_send` is best-effort bonus.

## Missed Tickets (Email Loss)

**Problem:** TinyDesk creates the GitHub issue and assigns it, but the assignment email is never received.
**Cause:** Email delivery is a single point of failure.
**Fix:** Add GitHub poll as safety net in the cron — `GET /search/issues?q=assignee:<bot>+state:open` every 15min. Cross-check against support-tickets.json + existing comments + existing PRs to avoid re-dispatching.

## ACP Permission Errors

**Problem:** ACP session fails with "Permission prompt unavailable in non-interactive mode".
**Cause:** ACP sessions run non-interactively. If `permissionMode` blocks writes/exec, the session fails.
**Fix:** Configure acpx plugin permissions:
```bash
openclaw config set plugins.entries.acpx.config.permissionMode approve-all
openclaw config set plugins.entries.acpx.config.nonInteractivePermissions deny
```
Then restart the gateway.

## ACP Backend Not Ready

**Problem:** `sessions_spawn` with `runtime: "acp"` fails with "ACP runtime backend is not configured".
**Cause:** acpx plugin not installed or not enabled.
**Fix:**
```bash
openclaw plugins install acpx
openclaw config set plugins.entries.acpx.enabled true
openclaw config set acp.enabled true
openclaw config set acp.backend acpx
systemctl --user restart openclaw-gateway
```

## Heartbeat vs Cron Confusion

**Problem:** Email checks appear to work (logs show activity) but tickets are missed.
**Cause:** Heartbeat (main session) was doing email checks sporadically; the isolated cron job was either broken or never running.
**Diagnosis:** Check `openclaw cron list --json` — if `lastRunAtMs: null` or `consecutiveErrors > 0`, the cron is the problem.
**Fix:** Separate concerns cleanly: heartbeat handles main-session tasks only, cron handles email + GitHub polling.

## Duplicate Dispatch

**Problem:** Same ticket dispatched multiple times.
**Cause:** GitHub poll marks a ticket as unactioned if the comment or PR check doesn't match exactly.
**Fix:** Add `issue_url` to support-tickets.json immediately on dispatch (before ACP session runs). Check this file first in the GitHub poll before checking comments/PRs.
