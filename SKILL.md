---
name: claw-mentor-mentee
description: Safe OpenClaw evolution — get safety-checked compatibility reports from expert builders delivered directly to your agent. Apply or skip updates, with automatic rollback protection.
metadata: {"openclaw": {"emoji": "🔥", "primaryEnv": "CLAW_MENTOR_API_KEY", "homepage": "https://clawmentor.ai"}}
---

# Claw Mentor — Mentee Skill

> Bring your mentor's updates directly into your OpenClaw agent. Get notified when a new compatibility report is ready, review it in plain English, and apply or skip — all from your OpenClaw chat.

---

## Description

Claw Mentor is a mentorship platform for OpenClaw users. You subscribe to an expert mentor (like Ember 🔥) who maintains a carefully crafted OpenClaw setup. Whenever they update their configuration, you get a plain-English compatibility report explaining what changed, what it does, and whether it's safe for your setup.

This skill connects your local OpenClaw agent to Claw Mentor. Your agent checks for new reports on a schedule, notifies you, and helps you apply or skip changes — including taking a local backup before anything changes.

**Privacy note:** Your AGENTS.md, skill files, and config are NEVER sent to Claw Mentor. The server only receives your onboarding survey answers (which you provide voluntarily) and your apply/skip/rollback decisions. No raw configuration data ever leaves your machine.

---

## What It Does

- Checks `app.clawmentor.ai` every few hours for new compatibility reports from your subscribed mentors
- Notifies you via your configured channel (Telegram, etc.) when a new report is ready
- Shows a plain-English summary: what changed, risk level, what to expect
- Guides you through applying or skipping each update
- Takes a local snapshot (`cp -r`) before any changes, so you can always roll back
- Reports your decision (applied/skipped/rolled back) back to Claw Mentor — no config content shared

---

## Installation

```bash
clawhub install claw-mentor-mentee
```

Start a new OpenClaw session after installing. Your agent will guide you through the API key setup on first use.

Get your API key at: **app.clawmentor.ai → Settings → Mentee Skill**

---

## Setup (First Run)

**When this skill loads, check if `CLAW_MENTOR_API_KEY` is set in your environment.**

**If it IS set:** You're connected. First call `POST https://app.clawmentor.ai/api/mentee/bootstrap` (with `Authorization: Bearer {CLAW_MENTOR_API_KEY}`) to ensure any mentor updates you haven't seen yet are queued. Then call `GET https://app.clawmentor.ai/api/mentee/reports` and report back what's waiting. Also ensure `HEARTBEAT.md` contains the line `- Check for new Claw Mentor reports` — add it if missing.

**If it is NOT set**, tell the user:

> "To activate ClawMentor, I need your API key.
> 
> 👉 Get it at **app.clawmentor.ai → Settings → Mentee Skill → Generate API Key**
> 
> Once you have it, share it here and I'll configure it."

**When the user provides their API key:**
1. Store it: `openclaw config set 'skills.entries["claw-mentor-mentee"].apiKey' THEIR_KEY`
2. Confirm: "✅ ClawMentor is connected. Fetching your first report now..."
3. **Bootstrap on first install** — call `POST https://app.clawmentor.ai/api/mentee/bootstrap` with header `Authorization: Bearer {CLAW_MENTOR_API_KEY}`. This fetches the latest update from each of your subscribed mentors and treats it as new — because for your setup, it IS new. Wait for the response.
4. After bootstrap, call `GET https://app.clawmentor.ai/api/mentee/reports` and show the user what's waiting.
5. **Schedule periodic checks** — append this line to `HEARTBEAT.md` in your workspace (create the file if it doesn't exist):
   ```
   - Check for new Claw Mentor reports
   ```
   This tells your OpenClaw agent to run the mentee skill check on every heartbeat cycle. Confirm to the user: "✅ Heartbeat check scheduled — I'll ping you automatically when a new report is ready."

---

## Configuration

| Variable | Where it comes from | Default |
|---|---|---|
| `CLAW_MENTOR_API_KEY` | app.clawmentor.ai → Settings → Mentee Skill | Required |
| `CLAW_MENTOR_CHECK_INTERVAL_HOURS` | Optional — set in your OpenClaw environment | `6` |

OpenClaw stores your API key in `~/.openclaw/openclaw.json` under `skills.entries["claw-mentor-mentee"].apiKey` and automatically injects it as `CLAW_MENTOR_API_KEY` each session.

---

## Permissions

| Permission | Why |
|---|---|
| `READ: ~/.openclaw/` | To take snapshots before applying changes |
| `WRITE: ~/.openclaw/claw-mentor/snapshots/` | To store local backup snapshots |
| `WRITE: ~/.openclaw/claw-mentor/state.json` | To track last check time and notified reports |
| `NETWORK: app.clawmentor.ai` | To fetch reports and send status updates |
| `NOTIFY: configured channel` | To alert you when a new report is ready |
| `EXEC: cp, mkdir` | Shell commands for taking snapshots |

No other files are read. No config content is ever uploaded.

---

## Agent Instructions

When this skill is installed, your agent should follow these instructions:

### Heartbeat Check (every `CLAW_MENTOR_CHECK_INTERVAL_HOURS` hours)

1. Read `~/.openclaw/claw-mentor/state.json` to get `last_check` and `notified_report_ids` (create file if absent)
2. If time since `last_check` < `CLAW_MENTOR_CHECK_INTERVAL_HOURS` hours → skip, return `HEARTBEAT_OK`
3. Call `GET https://app.clawmentor.ai/api/mentee/reports` with header `Authorization: Bearer {CLAW_MENTOR_API_KEY}`
4. Update `state.json` with `last_check: now`
5. For each report in the response where `status == 'pending'` AND `id` NOT in `notified_report_ids`:
   - Send a notification message (see format below)
   - Add the report ID to `notified_report_ids` in state
6. If no pending reports → call `POST https://app.clawmentor.ai/api/mentee/bootstrap` to check for any mentor updates not yet queued for this user. If bootstrap returns `bootstrapped > 0`, go back to step 3 and surface the new reports. Otherwise → return `HEARTBEAT_OK`

**Notification message format:**
```
🔥 New mentor update ready!

**{mentor_name}** · {risk_emoji} {risk_level_display} risk

{plain_english_summary}

Say **"show my mentor report"** to see the full details.
Or: **"apply mentor report"** / **"skip mentor report"**
```

Risk emoji: LOW → 🟢, MEDIUM → 🟡, HIGH → 🔴

### Command: "show my mentor report" / "my mentor reports" / "check my reports"

1. Call `GET https://app.clawmentor.ai/api/mentee/reports`
2. If no pending reports: "No new mentor reports. You're up to date! ✅"
3. For each pending report, show (use bullet lists — NOT markdown tables, they render poorly on Telegram and most channels):
   ```
   📋 Report from {mentor_name} — {date}
   Risk: {risk_level} {risk_emoji}

   {plain_english_summary}

   Skills to add ({N}):
   • {skill_name} {risk_emoji} — {what_it_does}
   • ...

   Skills to modify: {names or "none"}
   Skills to remove: {names or "none"}
   Permissions being added: {comma-separated list or "none"}

   {recommendation if non-empty}

   Say "apply mentor report" to apply or "skip mentor report" to skip.
   ```

### Command: "apply mentor report" / "apply [mentor name]'s update"

This is the most important command. Follow all steps carefully.

1. Call `GET https://app.clawmentor.ai/api/mentee/reports` to get the latest pending report
2. If no pending reports: "Nothing to apply — no pending reports."
3. Show a clean checklist of skills to add (bullet list, NOT a markdown table — tables render poorly on most channels):
   ```
   This report adds {N} skills:
   • skill-name 🟢 — what it does
   • skill-name 🟡 — what it does
   • skill-name 🔴 — what it does (high risk — needs your permission access)

   Which ones do you want? Say "all of them", name specific ones, or skip any you don't need.
   ```
4. **Take a snapshot before anything changes:**
   ```bash
   SNAPSHOT_DATE=$(date +%Y-%m-%d-%H-%M)
   SNAPSHOT_PATH="$HOME/.openclaw/claw-mentor/snapshots/$SNAPSHOT_DATE/"
   mkdir -p "$SNAPSHOT_PATH"
   # Use rsync to exclude the snapshots dir (avoids recursion); fall back to cp+cleanup
   if command -v rsync &>/dev/null; then
     rsync -a --exclude='claw-mentor/snapshots/' "$HOME/.openclaw/" "$SNAPSHOT_PATH"
   else
     cp -r "$HOME/.openclaw/" "$SNAPSHOT_PATH"
     rm -rf "$SNAPSHOT_PATH/claw-mentor/snapshots/"
   fi
   ```
5. Confirm: "✅ Backup saved to `~/.openclaw/claw-mentor/snapshots/{date}/`"
6. **Install each skill the user approved:**
   - First try: `clawhub install {skill-name}` — many skills are bundled with OpenClaw and won't need this; that's expected
   - For bundled skills (browser, telegram-notify, etc.): confirm they're already active, no install needed
   - For skills with CLI dependencies (github → `gh`, session-logs → `ripgrep`, coding-agent → `claude`, etc.): install the dependency automatically using the appropriate package manager
   - Note any skills that need an API key — flag them but don't block; handle in post-apply
7. When all installs are done, call `POST https://app.clawmentor.ai/api/mentee/status` with:
   ```json
   { "reportId": "{id}", "status": "applied", "snapshotPath": "{SNAPSHOT_PATH}" }
   ```
8. **Post-Apply Close** — send a clean summary (bullet list, not a table):
   ```
   ✅ Ember's report applied! Here's what went in:

   Installed & ready:
   • skill-name — status
   • ...

   Needs your input:
   • skill-name — [specific thing needed, e.g. "Notion API key — get it at notionintegrations.com"]
   • ...

   Your backup is at ~/.openclaw/claw-mentor/snapshots/{date}/ if you ever need to roll back.
   ```
9. **Check `~/.openclaw/claw-mentor/state.json` for `first_apply_done`.** If it is NOT set → run the **First-Time Welcome** flow below. Then set `first_apply_done: true` in state.json.

---

### First-Time Welcome (runs once, after first ever apply)

This is NOT a status report. It's a human conversation. Keep each message short. Don't send it all at once — send one message, wait for response or a few seconds, then continue.

**Message 1 — What's different now** (write this in plain English based on what was actually installed, don't just list skill names):
> "Here's what you can do now that you couldn't before:
> [list 3-5 natural language examples based on installed skills, e.g.]
> • 'Search for recent news on X' — I'll pull live web results
> • 'Summarize this URL/video/podcast' — I'll give you the key points
> • 'What's the weather today?' — quick answer via heartbeat
> • 'Check my GitHub issues' — I'll list and help triage them
> • I'll now send you a morning and evening brief automatically
>
> [If anything still needs setup]: To finish: [1] [specific action] takes [time estimate]. Want to do that now?"

**Message 2 — One clear action if anything needs setup** (only if there are pending API keys or setup steps):
> "The one thing left: [skill] needs a [key type]. Here's how:
> [Simple 1-2 line instruction — no jargon]
> Once you do that, [skill] will [what it does]. Takes about [X] minutes."

Wait for their response before continuing.

**Message 3 — Get to know you** (conversational, not a form):
> "Quick question — what's the main thing you want me to help with day-to-day? Work stuff, personal projects, research, staying on top of things...? Just a sentence or two is fine."

When they respond, follow up with one more:
> "Got it. And is there anything specific you're working on right now — a project, a goal, something you're trying to figure out?"

Save both answers to `~/.openclaw/claw-mentor/state.json` under `user_profile.goals` and `user_profile.context`. This personalizes future reports.

**Message 4 — Close** (short, energizing, done):
> "You're all set. 🔥 Ember will ping you when there's a new update — each report will get more useful as I learn what matters to you. Just talk to me like normal and I'll use everything we just set up."

### Command: "skip mentor report" / "skip [mentor]'s update"

1. Get the latest pending report (same API call)
2. If none: "Nothing to skip."
3. Call `POST https://app.clawmentor.ai/api/mentee/status` with `{ "reportId": "{id}", "status": "skipped" }`
4. Confirm: "Skipped. You can still view it at app.clawmentor.ai/dashboard whenever you're ready."

### Command: "roll back [mentor]'s update" / "undo mentor changes"

1. Find the most recently applied report from the last API call (or ask user which one)
2. Check if a snapshot was taken (look in `~/.openclaw/claw-mentor/snapshots/` for the most recent)
3. Show the restore command:
   ```bash
   cp -r ~/.openclaw/claw-mentor/snapshots/{most-recent-date}/ ~/.openclaw/
   ```
4. Remind user: "After restoring, restart your OpenClaw agent for changes to take effect."
5. When user confirms they've restored: call `POST https://app.clawmentor.ai/api/mentee/status` with `{ "reportId": "{id}", "status": "rolled_back" }`

---

## State File Format

`~/.openclaw/claw-mentor/state.json`:
```json
{
  "last_check": "2026-03-01T14:32:00Z",
  "notified_report_ids": ["uuid1", "uuid2"],
  "last_snapshot_path": "~/.openclaw/claw-mentor/snapshots/2026-03-01-14-32/"
}
```

Create this file on first use if it doesn't exist.

---

## API Reference

All endpoints at `https://app.clawmentor.ai`.

### GET /api/mentee/reports
**Auth:** `Authorization: Bearer {CLAW_MENTOR_API_KEY}`  
**Returns:**
```json
{
  "user": { "id": "...", "email": "...", "tier": "starter" },
  "reports": [
    {
      "id": "uuid",
      "created_at": "2026-03-01T10:00:00Z",
      "risk_level": "low",
      "plain_english_summary": "...",
      "skills_to_add": [{ "name": "...", "what_it_does": "...", "permissions_it_needs": [] }],
      "skills_to_modify": [],
      "skills_to_remove": [],
      "permission_changes": [],
      "status": "pending",
      "mentors": { "name": "Ember 🔥", "handle": "ember", "specialty": "..." }
    }
  ],
  "subscriptions": [...]
}
```

### POST /api/mentee/status
**Auth:** `Authorization: Bearer {CLAW_MENTOR_API_KEY}`  
**Body:** `{ "reportId": "uuid", "status": "applied|skipped|rolled_back", "snapshotPath": "~/.openclaw/..." }`  
**Returns:** `{ "success": true, "reportId": "...", "status": "applied", "updated_at": "..." }`

---

## Troubleshooting

**`clawhub install` rate limited** → ClawHub enforces per-IP download limits. Wait 2–3 minutes and retry. If the skill folder already exists from a failed attempt, run `clawhub install claw-mentor-mentee --force` to overwrite it.

**"Invalid API key"** → Go to app.clawmentor.ai → Settings → Mentee Skill → Generate a new key.

**"No reports found"** → Either no reports have been generated yet, or all are already applied/skipped. Claw Mentor runs daily — new reports appear within 24 hours of a mentor update.

**Snapshot failed** → Ensure your OpenClaw agent has filesystem access to `~/.openclaw/`. Check that `cp` and `mkdir` are available in your environment.

**Report not updating** → Check your API key is correct and you have an active subscription at app.clawmentor.ai.

---

## Source

Open source (auditable): [github.com/clawmentor/claw-mentor-mentee](https://github.com/clawmentor/claw-mentor-mentee)

Questions or issues? Open a GitHub issue or email hello@clawmentor.ai.
