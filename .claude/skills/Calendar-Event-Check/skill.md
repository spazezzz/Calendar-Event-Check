# Calendar-Slack-Notify

> **Runtime target:** Claude Remote Routine (web sandbox)  
> **Constraint:** No Bash / shell / git CLI — all I/O via Google Calendar connector and Slack connector only.

## Invocation

```
/Calendar-Slack-Notify
```

No arguments. Runs once per invocation: checks Google Calendar for events today and tomorrow, then sends a summary notification via Slack if any events are found.

## Required Connectors

| Connector | Purpose |
|-----------|---------|
| **Google Calendar** | Read events for today and tomorrow |
| **Slack** | Send event summary notification |

Both connectors must be connected in **claude.ai → Settings → Connectors** before the Routine runs.

---

## Execution Flow

### Step 1 — Calculate date/time ranges (Asia/Bangkok)

1. Obtain current UTC time from the Claude runtime context.
2. Add **7 hours** to convert to Asia/Bangkok (UTC+7).
3. Compute:
   - **Today:** `DATE_TODAY` (DD/MM/YYYY in Asia/Bangkok)
   - `timeMin_today` = 00:00:00 of today (UTC, ISO 8601)
   - `timeMax_today` = 23:59:59 of today (UTC, ISO 8601)
   - **Tomorrow:** `DATE_TOMORROW` (DD/MM/YYYY in Asia/Bangkok)
   - `timeMin_tomorrow` = 00:00:00 of tomorrow (UTC, ISO 8601)
   - `timeMax_tomorrow` = 23:59:59 of tomorrow (UTC, ISO 8601)

### Step 2 — Fetch events via Google Calendar connector

Call the **Google Calendar MCP connector** twice:

**2a. Today's events:**
- **Calendar:** use `GOOGLE_CALENDAR_ID` env var if set; otherwise use primary calendar.
- **Parameters:** `timeMin=timeMin_today`, `timeMax=timeMax_today`, `singleEvents=true`, `orderBy=startTime`

**2b. Tomorrow's events:**
- **Calendar:** use `GOOGLE_CALENDAR_ID` env var if set; otherwise use primary calendar.
- **Parameters:** `timeMin=timeMin_tomorrow`, `timeMax=timeMax_tomorrow`, `singleEvents=true`, `orderBy=startTime`

**Error handling:**
- Connector unavailable → **abort immediately** and log:
  ```
  [ERROR] Google Calendar connector unavailable at <ISO-8601 timestamp UTC+7>. Routine aborted.
  ```
- Do not proceed to send Slack notification.

### Step 3 — Check for events

**If no events exist for both today and tomorrow:**
- Log: `[INFO] No events found for <DATE_TODAY> or <DATE_TOMORROW>. Routine completed with no action.`
- **Stop. Do not send Slack notification.**

**If at least 1 event exists in either day:**
- Proceed to Step 4 (compose and send notification).

### Step 4 — Format notification message

Compose a Slack message with two sections (today / tomorrow):

```
🗓️ สรุปกำหนดการ 🗻

⭐ วันนี้ — <DATE_TODAY> ⭐
................................................................
<event lines for today, or "(ไม่มีกำหนดการ)">
รวม <N> กำหนดการ
━━━━━━━━━━━━━

⏳ พรุ่งนี้ — <DATE_TOMORROW> ⏳
................................................................
<event lines for tomorrow, or "(ไม่มีกำหนดการ)">
รวม <M> กำหนดการ
```

**Event formatting rules:**

For each event, extract:
| Field | Source | Fallback |
|-------|--------|----------|
| Title | `summary` | `"(ไม่มีชื่อ)"` |
| Start | `start.dateTime` | `start.date` (all-day) |
| End | `end.dateTime` | `end.date` (all-day) |
| Location | `location` | omit line if absent |
| Attendees | `attendees` count | omit line if absent |

**For each event:**
```
🕐 <START_TIME>–<END_TIME> | <TITLE>
   📍 <LOCATION or meeting link (if available)>
   👥 <attendee count> คน
```

- **Time format:** HH:MM (Asia/Bangkok)
- **All-day events:** display `ทั้งวัน` instead of time range
- **Omit 📍 line** if no location and no meeting link
- **Omit 👥 line** if no attendees
- **Max 10 events per day** — if more, append `... และอีก <K> รายการ`

**If a day has no events, display:**
```
_(ไม่มีกำหนดการ)_
```

**Example message:**
```
🗓️ สรุปกำหนดการ 🗻

⭐ วันนี้ — 03/05/2026 ⭐
................................................................
🕐 09:00–10:00 | Weekly Team Standup
   📍 https://meet.google.com/abc-defg-hij
   👥 8 คน
🕐 15:00–15:30 | Code Review Session
   👥 3 คน
รวม 2 กำหนดการ
━━━━━━━━━━━━━

⏳ พรุ่งนี้ — 04/05/2026 ⏳
................................................................
🕐 13:00–14:00 | 1-on-1 กับ Manager
   👥 2 คน
🕐 ทั้งวัน | วันหยุดธนาคาร
รวม 2 กำหนดการ
```

### Step 5 — Send Slack notification

Call the **Slack MCP connector** to send the message:

- **Channel:** `#daily-reminders`
- **Message:** text from Step 4

**Handling:**
- Success → log:
  ```
  [INFO] Slack notification sent successfully for <DATE_TODAY> and <DATE_TOMORROW>.
  ```
- Connector error → log:
  ```
  [ERROR] Slack send failed: <error message>. Do not retry silently.
  ```
- **Do not auto-retry** on failure.

---

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `GOOGLE_CALENDAR_ID` | No | Calendar ID — defaults to `primary` if absent |

> Google OAuth credentials and GitHub variables are **not needed** — authentication is handled entirely by the Google Calendar and Slack connectors.

---

## Timezone

All date/time values use **Asia/Bangkok (UTC+7)**.

1. Obtain current UTC time from the Claude runtime context.
2. Add **7 hours** to convert to Asia/Bangkok.
3. Compute today and tomorrow dates in Bangkok timezone.
4. When querying Google Calendar, convert Bangkok dates back to UTC with `+07:00` offset for RFC 3339 format.

---

## Guardrails Summary

| Rule | Behaviour |
|------|-----------|
| No Bash / shell / git CLI | Google Calendar + Slack connectors only |
| Google Calendar connector unavailable | Abort immediately; log error; do not send Slack |
| Slack connector unavailable | Abort; log error; do not proceed |
| No events found (both days) | Exit quietly — no Slack message sent |
| Events found (one or both days) | Send Slack notification with summary |
| More than 10 events per day | Show first 10, append `... และอีก <K> รายการ` |
| Any connector error | Log error detail; do not retry silently |
