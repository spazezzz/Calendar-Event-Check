# Calendar-Event-Check

> **Runtime target:** Claude Remote Routine (web sandbox)  
> **Constraint:** No Bash / shell / git CLI — all I/O via MCP connectors and WebFetch only.

## Invocation

```
/Calendar-Event-Check
```

No arguments. Runs once per invocation: checks tomorrow's Google Calendar events and optionally sends a LINE notification.

## Required Connectors

| Connector | Purpose |
|-----------|---------|
| **GitHub** | Liveness check — abort if unavailable |
| **Google Calendar** | Read tomorrow's events |

Both connectors must be connected in **claude.ai → Settings → Connectors** before the Routine runs.

---

## Execution Flow

### Step 0 — Verify connectors

**GitHub connector:** Attempt to read `README.md` via the GitHub MCP connector.
- Succeeds → continue.
- Error / unavailable → **abort** and log:
  ```
  [ERROR] GitHub connector unavailable at <ISO-8601 timestamp UTC+7>. Skill aborted.
  ```

**Google Calendar connector:** Attempt a minimal call (e.g. list calendars or list events with a narrow window) to confirm the connector is live.
- Succeeds → continue.
- Error / unavailable → **abort** and log:
  ```
  [ERROR] Google Calendar connector unavailable at <ISO-8601 timestamp UTC+7>. Skill aborted.
  ```

### Step 1 — Compute tomorrow's date (Asia/Bangkok)

1. Obtain current UTC time from the Claude runtime context.
2. Add **7 hours** to convert to Asia/Bangkok (UTC+7).
3. Add **1 day** to obtain tomorrow.
4. Derive:
   - `TOMORROW_DATE` — `YYYY-MM-DD`
   - `TIME_MIN` — `YYYY-MM-DDT00:00:00+07:00` (start of tomorrow Bangkok)
   - `TIME_MAX` — `YYYY-MM-DDT00:00:00+07:00` (start of the day after tomorrow)
   - `DISPLAY_DATE` — Thai short format `DD Mmm BBBB` (Buddhist Era = Gregorian + 543)
     - Thai month abbreviations: ม.ค. ก.พ. มี.ค. เม.ย. พ.ค. มิ.ย. ก.ค. ส.ค. ก.ย. ต.ค. พ.ย. ธ.ค.
     - Example: `04 พ.ค. 2569`

### Step 2 — Fetch tomorrow's events via Google Calendar connector

Call the **Google Calendar MCP connector** to list events:

- **Calendar:** use `GOOGLE_CALENDAR_ID` env var if set; otherwise use the primary calendar of the connected account.
- **Time range:** `TIME_MIN` → `TIME_MAX` (full day in Asia/Bangkok).
- **Options:** expand recurring events (`singleEvents=true`), order by start time.

Response handling:
- Events returned → proceed to **Step 3b** (events found).
- Empty list → proceed to **Step 3a** (no events).
- Connector error → **abort** and log:
  ```
  [ERROR] Google Calendar connector returned an error at <timestamp>: {error detail}
  ```

### Step 3a — No events tomorrow

- Log: `[INFO] No events tomorrow (TOMORROW_DATE). Skill completed with no action.`
- **Stop. Do not send any LINE notification.**

### Step 3b — Events found — format event list

For each event, extract:

| Field | Source | Fallback |
|-------|--------|----------|
| Title | `summary` | `"(ไม่มีชื่อ)"` |
| Start | `start.dateTime` | `start.date` (all-day) |
| End | `end.dateTime` | `end.date` (all-day) |

Format each event as one line:
- Timed event: `• {title} — {HH:MM} น. – {HH:MM} น.`
- All-day event: `• {title} — ทั้งวัน`

All times must be displayed in **Asia/Bangkok (UTC+7)**. Convert from UTC or any other offset before display.

### Step 4 — Send LINE notification

**Skip this entire step** if `LINE_CHANNEL_ACCESS_TOKEN` or `LINE_TO` is absent. Skill still completes normally.

Compose the message text:

```
📅 นัดหมายพรุ่งนี้ (DISPLAY_DATE)

{event line 1}
{event line 2}
…

รวม N รายการ
```

Example:
```
📅 นัดหมายพรุ่งนี้ (04 พ.ค. 2569)

• ประชุมทีม — 09:00 น. – 10:00 น.
• Review PR — 14:00 น. – 15:30 น.
• วันหยุดพักผ่อน — ทั้งวัน

รวม 3 รายการ
```

Send via **WebFetch** (POST):

```
POST https://api.line.me/v2/bot/message/push
Content-Type: application/json
Authorization: Bearer {LINE_CHANNEL_ACCESS_TOKEN}

{
  "to": "{LINE_TO}",
  "messages": [{"type": "text", "text": "{message}"}]
}
```

Handling:
- HTTP 200 → log `[INFO] LINE notification sent successfully at <timestamp>.`
- HTTP non-200 → log `[ERROR] LINE API returned {status}: {body}`. **Do not retry.**
- WebFetch error → log `[ERROR] LINE WebFetch failed: {error}`. Do not block completion.

---

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `GITHUB_REPO_OWNER` | Yes | GitHub username / org owning this repository |
| `GITHUB_REPO_NAME` | Yes | Repository name |
| `GOOGLE_CALENDAR_ID` | No | Calendar ID — defaults to `primary` if absent |
| `LINE_CHANNEL_ACCESS_TOKEN` | No | LINE Messaging API long-lived channel access token |
| `LINE_TO` | No | LINE User ID (`U…`) or Group ID (`C…`) |

> Google OAuth credentials (CLIENT_ID / CLIENT_SECRET / REFRESH_TOKEN) are **not needed** — authentication is handled entirely by the Google Calendar connector.

---

## Timezone

All date/time values use **Asia/Bangkok (UTC+7)**.

1. Obtain current UTC epoch from the Claude runtime context.
2. Add **25 200 seconds** (7 × 3 600) to get Bangkok epoch.
3. Always append `+07:00` when constructing RFC 3339 strings.

---

## Guardrails Summary

| Rule | Behaviour |
|------|-----------|
| No Bash / shell / git CLI | MCP connectors + WebFetch only |
| GitHub connector unavailable | Abort; log timestamped error |
| Google Calendar connector unavailable | Abort; log timestamped error |
| Calendar connector returns error | Abort; log error detail |
| No events tomorrow | Exit quietly — no LINE message sent |
| `LINE_*` env vars absent | Skip LINE step; skill completes normally |
| LINE API HTTP non-200 | Log error (status + body); no silent retry |
