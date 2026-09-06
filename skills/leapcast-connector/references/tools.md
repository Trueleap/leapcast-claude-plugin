# Leapcast connector tools

All tools need the read permission; the ones marked **write** also need the write permission.
Prefix every name with `mcp__plugin_leapcast_leapcast__` when calling it from this plugin.
Annotations: ro = read-only, d = destructive, i = idempotent, ow = open world (emails people).

## Organizations

| Tool | Purpose | Inputs | Annotations |
| --- | --- | --- | --- |
| `list_organizations` | Organizations the user belongs to and which one is active | none | ro, i |
| `switch_organization` | Make one organization active for later calls | `organizationId` | i |

## Meetings

| Tool | Purpose | Inputs | Annotations |
| --- | --- | --- | --- |
| `list_upcoming_meetings` | Scheduled meetings from now, with join links | `days` 1–62 (default 14), `limit` 1–50 (default 20) | ro, i |
| `list_past_meetings` | Meetings attended or hosted, newest first, with `transcripts[]` (`meetingId`, `transcriptId`, `transcriptStatus`, `summaryStatus`) | `days` 1–365 (default 30), `limit` 1–50 (default 20), `cursor` | ro, i |
| `who_is_live` | Meetings live now or inside their scheduled window, with participant counts | none | ro, i |
| `get_meeting` | One meeting: schedule, host, description, live state, latest session, the user's invite status, `roomId` | `meetingId` | ro, i |
| `list_meeting_invites` | Invitations the user has not answered; gives `participantId` | `limit` 1–100 (default 50) | ro, i |
| `get_room` | Room configuration: agenda, invitees and roles, recurrence, transcript settings | `roomId` | ro, i |

## Transcripts and summaries

| Tool | Purpose | Inputs | Annotations |
| --- | --- | --- | --- |
| `get_meeting_summary` | Overview, key points, decisions, action items | `meetingId`, `transcriptId` (optional) | ro, i |
| `get_meeting_transcript` | Timestamped speaker turns in pages; continue with `offset` = `nextOffset` | `meetingId`, `transcriptId` (optional), `offset` ≥ 0, `limit` 1–200 (default 200) | ro, i |
| `search_transcript` | Turns in one transcript containing a phrase (case-insensitive substring) | `meetingId`, `query` 1–200 chars, `transcriptId` (optional) | ro, i |
| `search_meetings` | Same search across recent transcribed meetings in the active organization | `query` 1–200, `days` 1–365 (default 30), `meetings` 1–20 (default 10), `limit` 1–100 (default 25) | ro, i |

## Recordings

| Tool | Purpose | Inputs | Annotations |
| --- | --- | --- | --- |
| `list_recordings` | Recordings across all the user's organizations, newest first, with playback links | `meetingId`, `search` (title text), `limit` 1–50 (default 10), `cursor` | ro, i |
| `play_recording` | Playback link cued to a moment; returns `playbackUrl`, `startAtSeconds`, `recordingStartedAt`, `durationSeconds` and a text link ending in `#t=<seconds>`. Defaults to the newest recording and clamps `at` to its length; pass `recordingId` from `list_recordings` when another recording covers the moment | `meetingId`, `at` (`12:34`, `1:02:03` or seconds), `transcriptId`, `recordingId` | ro, i |

## Live

| Tool | Purpose | Inputs | Annotations |
| --- | --- | --- | --- |
| `join_live_meeting` **write** | Joins a live meeting as the user; playback needs a host that renders cards (not Claude Code) | `meetingId` | i |

## Writes

| Tool | Purpose | Inputs | Annotations |
| --- | --- | --- | --- |
| `create_meeting` **write** | Schedule a meeting; invitees receive email | `title` 1–200, `description` ≤ 4000, `startsAt` and `endsAt` ISO-8601 with offset, `timezone` IANA, `recurrence` RRULE ≤ 500, `invitees` ≤ 50 emails | ow |
| `reschedule_meeting` **write** | Move a one-off meeting; attendees notified | `roomId`, `startsAt`, `endsAt` | i |
| `cancel_meeting` **write** | Cancel by room id; live meetings must end first | `roomId`, `notifyAttendees` (default true), `reason` ≤ 500 | d, i |
| `invite_to_meeting` **write** | Email invitations; already-invited people skipped | `meetingId`, `emails` 1–50 | i, ow |
| `respond_to_invite` **write** | Accept, tentatively accept or decline | `participantId`, `response` = `confirmed`, `tentative` or `declined` | i |

## Result shapes worth knowing

- Meeting lists: one line per meeting, `<title> [scheduled|live|ended|cancelled] <start> → <end> (meeting <id>) — join: <url>`, then `More available: pass cursor "…"` when paged.
- Transcript pages: `[mm:ss] Speaker: text`, `(?)` marks low confidence, ending with `Continue with offset N.` or `End of transcript.`
- `play_recording`: `Playing "<title>" recorded <date>, <n> min from <h:mm:ss>. Direct link: <url>#t=<seconds>`.
