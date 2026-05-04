# Claude-Routine-Guideline-Update

Automated clinical guideline monitoring and publication via Claude Remote Routine.

The skill researches newly published Thai and international clinical guidelines, analyses them through a Thai-context lens, writes a structured brief, and publishes it to this repository — all without Bash, shell, or git CLI.

This repository also hosts the **Calendar-Event-Check** skill, which checks Google Calendar for tomorrow's events and delivers a LINE notification — likewise without any shell access.

---

## Repository Structure

```
.claude/
  skills/
    Claude-Routine-Guideline-Update/
      skill.md              # Guideline-update skill instructions
    Calendar-Event-Check/
      skill.md              # Calendar event check skill instructions
reference/
  sources.md               # Registry of verified guideline sources (auto-updated)
  perspectives.md          # Thai-context analysis framework (edit to customise)
articles/
  DD-MM-YYYY-brief.md      # Generated briefs (one per run)
README.md
env.example
.gitignore
```

---

## Skill 1 — Claude-Routine-Guideline-Update

Researches, analyses, and publishes Thai & international clinical guideline updates.

See the [skill instructions](.claude/skills/Claude-Routine-Guideline-Update/skill.md) for the full execution flow.

### Running in Claude Web Routine

#### Prerequisites

| Requirement | Details |
|-------------|--------|
| Claude account | claude.ai with Routine (scheduled tasks) access |
| GitHub connector | Connected in Claude settings — grants MCP read/write to this repo |
| LINE Messaging API | Optional — only needed for LINE notifications |

#### 1. Connect the GitHub Connector

1. Open **claude.ai → Settings → Connectors**.
2. Click **Add connector → GitHub**.
3. Authorise Claude to access this repository.
4. Confirm the connector shows **Connected** status.

If the connector is unavailable at run time, the skill aborts and logs an error — it will **not** fall back to shell commands.

#### 2. Set Environment Variables

For Claude Web Routine, set variables in **Settings → Routine → Environment**:

| Variable | Required | Value |
|----------|----------|-------|
| `GITHUB_REPO_OWNER` | Yes | Your GitHub username or organisation name |
| `GITHUB_REPO_NAME` | Yes | `claude-routine-guideline-update` (or your fork name) |
| `LINE_CHANNEL_ACCESS_TOKEN` | No | From LINE Developers Console → Messaging API channel |
| `LINE_TO` | No | LINE User ID (`U...`) or Group ID (`C...`) |

#### 3. Create the Routine

1. In Claude Web, open **Routines → New Routine**.
2. Set **Skill**: `Claude-Routine-Guideline-Update`
3. Set **Schedule**: `0 8 * * 1` (Monday 08:00) or `0 7 * * *` (daily 07:00)
4. Set **Timezone**: `Asia/Bangkok`
5. Save and enable.

---

## Skill 2 — Calendar-Slack-Notify

Checks Google Calendar for events today and tomorrow, then sends a Slack notification with a summary if any events are found. If no events exist, the routine exits silently.

See the [skill instructions](.claude/skills/Calendar-Event-Check/skill.md) for the full execution flow.

### Running in Claude Web Routine

#### Prerequisites

| Requirement | Details |
|-------------|--------|
| Claude account | claude.ai with Routine (scheduled tasks) access |
| Google Calendar connector | Connected — reads calendar events via MCP (no OAuth credentials needed) |
| Slack connector | Connected — sends messages to #daily-reminders channel |

#### 1. Connect the Google Calendar Connector

1. **claude.ai → Settings → Connectors → Add connector → Google Calendar**
2. Sign in with the Google account whose calendar you want to check.
3. Confirm **Connected** status.

> No Google Cloud project, no OAuth credentials, no refresh token needed.  
> Authentication is handled entirely by the connector.

#### 2. Connect the Slack Connector

1. **claude.ai → Settings → Connectors → Add connector → Slack**
2. Authorise Claude to access your Slack workspace.
3. Confirm **Connected** status.

#### 3. Set Environment Variables

For Claude Web Routine, set variables in **Settings → Routine → Environment**:

| Variable | Required | Value |
|----------|----------|-------|
| `GOOGLE_CALENDAR_ID` | No | Calendar ID — leave blank to use `primary` |

For local development, copy `env.example` → `.env` and fill in values. **Never commit `.env`.**

#### 4. Create the Routine

1. **Routines → New Routine**
2. Set **Skill**: `Calendar-Slack-Notify`
3. Set **Schedule**: `0 8 * * *` (08:00 each morning — checks today and tomorrow's events)
4. Set **Timezone**: `Asia/Bangkok`
5. Save and enable.

Suggested schedules:

| Use case | Cron | Bangkok time |
|----------|------|--------------|
| Morning reminder (recommended) | `0 8 * * *` | Every day 08:00 |
| Evening reminder | `0 21 * * *` | Every day 21:00 |
| Weekdays only | `0 8 * * 1-5` | Mon–Fri 08:00 |

#### 5. Verify a Test Run

1. Trigger a manual run from the Routine dashboard.
2. If events exist → Slack message arrives in #daily-reminders within seconds.
3. If no events → routine completes silently; check run log for `[INFO] No events found`.
4. Any `[ERROR]` lines in the run log indicate connector issues.

#### 6. Routine Guardrails

| Guardrail | Behaviour |
|-----------|-----------|
| No Bash / shell / git CLI | Google Calendar + Slack connectors only |
| Google Calendar connector unavailable | Abort immediately; log error; do not send Slack |
| Slack connector unavailable | Abort; log error |
| No events found (both days) | Exit quietly; no Slack message sent |
| Events found | Send Slack notification with summary to #daily-reminders |
| More than 10 events per day | Show first 10, append "... และอีก <K> รายการ" |

---

## LINE Notification Formats

**Skill 1 — Guideline Update:**
```
📋 Guideline Update — DD-MM-YYYY

• [TL;DR bullet 1]
• [TL;DR bullet 2]
• [TL;DR bullet 3]

🔗 https://github.com/{owner}/{repo}/blob/{SHA}/articles/DD-MM-YYYY-brief.md
```

**Skill 2 — Calendar Event Check:**
```
📅 นัดหมายพรุ่งนี้ (DD Mmm BBBB)

• {Event title} — HH:MM น. – HH:MM น.
• {All-day event} — ทั้งวัน

รวม N รายการ
```

---

## Skill 1 — Guardrails

| Guardrail | Behaviour |
|-----------|----------|
| No Bash/shell/git CLI | GitHub MCP connector + WebFetch only |
| Unverifiable guideline | Excluded from article; logged in `sources.md` |
| GitHub connector unavailable | Abort run, log error, do not publish |
| `LINE_*` env vars absent | Skip LINE step; commit proceeds normally |
| LINE API HTTP non-200 | Log error with status + body; no silent retry |
| Fabricated content | Strictly forbidden — cite or exclude |

---

## Local Development

1. Copy `env.example` → `.env` and fill in values.
2. Open a Claude Code session in this directory.
3. Run `/Calendar-Event-Check` or `/Claude-Routine-Guideline-Update [TOPIC]`.
4. Skills use your local MCP servers; LINE is called via WebFetch.

---

## Contributing

- Analysis framework: edit `reference/perspectives.md`.
- Article template: edit `.claude/skills/Claude-Routine-Guideline-Update/skill.md` (Step 4).
- Calendar notification format: edit `.claude/skills/Calendar-Event-Check/skill.md` (Step 4).
- Issues and PRs welcome.
