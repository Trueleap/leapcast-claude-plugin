---
name: leapcast-connector
description: This skill should be used whenever the user mentions Leapcast, their meetings, a meeting summary, a transcript, a recording, who is live, meeting invitations, or scheduling, moving or cancelling a meeting, and before calling any mcp__plugin_leapcast_leapcast__* tool. It explains the connector's single-active-organization model, which id each tool expects, how transcript search chains into recording playback, which calls need the write scope and a confirmation, and what every error code means.
user-invocable: false
---

# Leapcast connector

The Leapcast plugin bundles the hosted Leapcast MCP server (`https://mcp.leapcast.ai/mcp`).
Its tools act as the signed-in user inside **one active organization**. Every tool returns
full plain text plus structured content, so nothing is lost in Claude Code; only the
interactive cards that other hosts render are absent.

Tool names in this plugin are prefixed `mcp__plugin_leapcast_leapcast__`, for example
`mcp__plugin_leapcast_leapcast__list_upcoming_meetings`. The short names below refer to
those tools.

## Connection

- If no Leapcast tool is available or a call returns `unauthorized`, tell the user to open
  `/mcp`, pick **leapcast**, and sign in. The browser shows a Leapcast consent page with two
  permissions: read (meetings, transcripts, summaries, recordings) and write (schedule and
  change meetings). Access lasts one hour and refreshes automatically.
- The `/leapcast:setup` skill walks through sign-in and troubleshooting.

## Organization model

- All tools except `list_recordings` are scoped to the active organization. When something
  the user expects is "not found", check `list_organizations` and offer `switch_organization`
  before concluding the data does not exist.
- `switch_organization` only changes which organization later calls use. It needs no
  confirmation.

## Which id a tool expects

| Id | Comes from | Used by |
| --- | --- | --- |
| `meetingId` | any meeting list, `get_meeting`, `who_is_live` | `get_meeting`, `get_meeting_summary`, `get_meeting_transcript`, `search_transcript`, `list_recordings`, `play_recording`, `invite_to_meeting`, `join_live_meeting` |
| `roomId` | the `roomId` field on a meeting result | `get_room`, `reschedule_meeting`, `cancel_meeting` |
| `transcriptId` | `list_past_meetings` → `transcripts[]` | optional on transcript, summary and playback tools; they default to the latest ready transcript |
| `participantId` | `list_meeting_invites` | `respond_to_invite` |

Never guess an id. Resolve a meeting by title with `list_past_meetings` or
`list_upcoming_meetings` first; if several match, ask which one.

## Finding a moment and playing it

1. Search: `search_meetings` for a phrase across recent transcribed meetings (default 30
   days, at most 20 meetings), or `search_transcript` inside one meeting. Matching is a
   case-insensitive substring match, so search for short distinctive words.
2. Each hit carries an `at` timestamp (`mm:ss` or `h:mm:ss`) and a `transcriptId`.
3. Play: `play_recording` with `meetingId`, that `at` and that `transcriptId`. The text
   result carries a direct link ending in `#t=<seconds>` that opens the recording at the
   moment, already corrected for the gap between transcript time and recording time. If only
   `playbackUrl` and `startAtSeconds` are visible, build the link as
   `<playbackUrl>#t=<startAtSeconds>`. Give the user that link; it stays valid for about
   two days.
4. Check coverage. A session can have several recordings, and `play_recording` defaults to
   the newest one, clamping `at` to its length when the moment lies outside it. When the
   returned `startAtSeconds` sits at the very end, or `recordingStartedAt` is later than the
   moment, call `list_recordings` with the `meetingId`, pick the recording whose start and
   duration contain the moment, and call `play_recording` again with its `recordingId`.
   `at` also accepts a plain number of seconds.

## Reading transcripts and summaries

- `get_meeting_summary` returns overview, key points, decisions and action items for the
  latest session with a ready summary. On `no_summary_yet`, offer the transcript instead.
- `get_meeting_transcript` pages by `offset`; a page holds at most 200 turns or about 8,000
  characters. Continue from the `nextOffset` in the result until "End of transcript".
  Turns marked `(?)` were transcribed with low confidence.
- `list_past_meetings` and `list_recordings` page with `cursor`.

## Writes: scope and confirmation

`create_meeting`, `reschedule_meeting`, `cancel_meeting`, `invite_to_meeting`,
`respond_to_invite` and `join_live_meeting` need the write permission. Before any of them:

- Show exactly what will happen (title, times in the user's timezone, who gets emailed) and
  ask for confirmation. `create_meeting` and `invite_to_meeting` send email invitations;
  `cancel_meeting` is destructive and can email a cancellation.
- Pass `startsAt` and `endsAt` as ISO-8601 **with a UTC offset** (`2026-09-08T10:00:00+01:00`)
  and pass the IANA `timezone` (`Europe/Lisbon`). State the timezone used.
- `recurrence` is an RRULE such as `FREQ=WEEKLY;BYDAY=MO`. Recurring meetings cannot be moved
  with `reschedule_meeting`; direct the user to the room in the browser.
- On `insufficient_scope`, the connection was granted read only. Tell the user to disconnect
  and reconnect from `/mcp` and to keep the write permission ticked on the consent page.

## Live meetings in Claude Code

`join_live_meeting` seats the user in the room as a visible participant, and its audio and
video only play inside a host that renders the live companion card. Claude Code does not.
Prefer `who_is_live` and hand over the join link. If the user still wants the tool, warn that
they will appear in the room without hearing it, and give the `joinUrl` from the result.

## Errors

Every failure is a text result of the form `<code>: <message>`. The full list with the
action to take is in `references/errors.md`. The ones met most often:

- `unauthorized`: connection expired, reconnect from `/mcp`.
- `insufficient_scope`: read-only connection, reconnect with write.
- `meeting_not_found`, `not_found`, `forbidden`: wrong id or wrong active organization.
- `no_transcript_yet`, `no_summary_yet`, `no_recording`: processing not finished or nothing
  uploaded; say so plainly and offer what exists.
- `rate_limited`: wait the number of seconds in the message before retrying.

## Limits and data handling

- Requests are rate limited per user; over the limit the server answers 429 with a
  `retry-after` header. Wait that long, and batch reads (one `search_meetings` beats many
  `search_transcript` calls).
- The connector stores only which organization is active per user and client. Its audit log
  records tool name and outcome, never arguments or transcript text.

## Additional resources

- `references/tools.md`: every tool with inputs, ranges, defaults and annotations.
- `references/errors.md`: every error code with meaning and what to tell the user.
