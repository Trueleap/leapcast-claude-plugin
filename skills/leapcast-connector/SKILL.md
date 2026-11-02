---
name: leapcast-connector
description: This skill should be used whenever the user mentions Leapcast, their meetings, a meeting summary, a transcript, a recording, a clip or GIF, a meeting QR code, who is live, meeting invitations, how Leapcast works, or scheduling, moving or cancelling a meeting, and before calling any mcp__plugin_leapcast_leapcast__* tool. It explains the connector's single-active-organization model, which id each tool expects, how transcript search chains into recording playback, which calls need the write scope and a confirmation, and what every error code means.
user-invocable: false
---

# Leapcast connector

The Leapcast plugin bundles the hosted Leapcast MCP server (`https://mcp.leapcast.ai/mcp`).
Its tools act as the signed-in user inside **one active organization**. Every tool returns
full plain text plus structured content, so nothing is lost in Claude Code; only the
interactive cards that other hosts render are absent.

Tool names in this plugin are prefixed `mcp__plugin_leapcast_leapcast__`, for example
`mcp__plugin_leapcast_leapcast__list_meetings`. The short names below refer to
those tools.

## Connection

- If no Leapcast tool is available or a call returns `unauthorized`, tell the user to open
  `/mcp`, pick **leapcast**, and sign in. The browser shows a Leapcast consent page with two
  permissions: read (meetings, transcripts, summaries, recordings) and write (schedule and
  change meetings). Access lasts one hour and refreshes automatically.
- The `/leapcast:setup` skill walks through sign-in and troubleshooting.

## Organization model

- Listing meetings, searching across meetings and creating meetings use the active
  organization. An explicit meeting or room id works wherever the user has access, and
  `list_recordings` covers all of the user's organizations. When something the user expects
  is "not found", check `list_organizations` and offer `switch_organization` before
  concluding the data does not exist.
- `switch_organization` only changes which organization later calls use. It needs no
  confirmation.

## Which id a tool expects

| Id | Comes from | Used by |
| --- | --- | --- |
| `meetingId` | `list_meetings` (every view but `invites`), `get_meeting` | `get_meeting`, `get_meeting_summary`, `get_meeting_transcript`, `search_meetings`, `list_recordings`, `play_recording`, `invite_to_meeting`, `join_live_meeting`, `make_qr_code`, clip and host tools |
| `roomId` | the `roomId` field on a meeting or invitation result | `get_meeting` (room settings, invitees, plan), `update_meeting`, `cancel_meeting` |
| `transcriptId` | `list_meetings` view `past` → `transcripts[]`, search results | optional on transcript, summary, search and playback tools; they default to the latest ready transcript (the summary to the latest ready summary) |
| `participantId` | `list_meetings` view `invites` (it gives `roomId`, not `meetingId`) | `respond_to_invite` |
| `occurrenceId` | `meetingSessionId` in `get_content_studio` (a recorded date) | `cut_meeting_clip` and the other Studio tools |

Never guess an id. Resolve a meeting by title with `list_meetings` (view `past` or
`upcoming`) first; if several match, ask which one.

## Finding a moment and playing it

1. Search: `search_meetings` with `query` searches transcribed meetings in the active
   organization, newest first (all history unless `days` is set); add `meetingId` (and
   optionally `transcriptId`) to search one meeting. The default is a case-insensitive
   substring match, so search for short distinctive words. Across meetings only,
   `mode: "semantic"` ranks by meaning but is not exhaustive. Follow `nextCursor` with the
   other arguments unchanged.
2. Each hit carries an `at` timestamp (`mm:ss` or `h:mm:ss`); the result gives its
   `transcriptId`.
3. Play: `play_recording` with `meetingId`, that `at` and that `transcriptId`. The text
   result carries a direct link ending in `#t=<seconds>` that opens the recording at the
   moment, already corrected for the gap between transcript time and recording time. If only
   `playbackUrl` and `startAtSeconds` are visible, build the link as
   `<playbackUrl>#t=<startAtSeconds>`. Give the user that link; it stays valid for about
   two days.
4. Check coverage. A session can have several recordings. `play_recording` picks the one
   that covers the moment, or answers `moment_not_recorded` when none does. When it cannot
   tell (a recording without timing), it plays the newest one with `at` clamped to its
   length: if `startAtSeconds` sits at the very end, call `list_recordings` with the
   `meetingId`, pick another recording and call `play_recording` again with its
   `recordingId`. `at` may also be a number of seconds as a string (`"754"`).
5. To keep the moment as a clip, see "Help, QR codes and clips" below.

## Reading transcripts and summaries

- `get_meeting_summary` returns overview, key points, decisions and action items for the
  latest session with a ready summary. On `no_summary_yet`, offer the transcript instead.
- `get_meeting_transcript` pages by `offset`; a page holds at most 200 turns or about 8,000
  characters. Continue from the `nextOffset` in the result until "End of transcript".
  Turns marked `(?)` were transcribed with low confidence.
- `list_meetings` and `list_recordings` page with `cursor`; keep `view` and `days` unchanged
  when continuing.

## Help, QR codes and clips

- `leapcast_help` with `query` answers how Leapcast works (features, limits, prices,
  permissions) from reviewed help articles. Use it instead of guessing. When nothing
  answers, it returns `askOriUrl`; give the user that link.
- `make_qr_code` with `meetingId` (`link`: `join` by default, or `watch` for the Leapcast
  Live page), or with a leapcast.ai or leapcast.live `url`, returns the link and a QR code.
  It changes nothing.
- `list_meeting_clips` reads a meeting's clips. `cut_meeting_clip` cuts 30–60 whole seconds
  of one recorded date into a draft clip: `meetingId`, `occurrenceId`, `startSeconds` and
  `endSeconds` on the meeting clock (transcript `at` times), `title`. `make_gif` loops
  3–6 seconds of a ready clip: `meetingId`, `clip: { runId, clipId, atSeconds,
  durationSeconds }`; its link lasts ten minutes. `get_clip_download_link` gives a
  ten-minute MP4 link. These are writes; none of them approves or publishes anything.

## Writes: scope and confirmation

These tools need the write permission (`switch_organization` does not): `create_meeting`,
`update_meeting`, `cancel_meeting`, `invite_to_meeting`, `respond_to_invite`,
`join_live_meeting`, `cut_meeting_clip`, `make_gif`, `get_clip_download_link`, the Studio
generate, edit and review tools, the host actions below, and room chat and polls
(`read_room_activity` too). Before any of them:

- Show exactly what will happen (title, times in the user's timezone, who gets emailed) and
  ask for confirmation. `create_meeting` and `invite_to_meeting` send email invitations;
  `cancel_meeting` is destructive and can email a cancellation.
- Pass every time as ISO-8601 **with a UTC offset** (`2026-09-08T10:00:00+01:00`). With a
  scheduled `create_meeting` or a settings edit, also pass the IANA `timezone`
  (`Europe/Lisbon`). State the timezone used.
- `recurrence` is an RRULE such as `FREQ=WEEKLY;BYDAY=MO`. `create_meeting` with
  `instant: true` makes an unscheduled room and takes no `startsAt`, `endsAt`, `timezone` or
  `recurrence`; it does not start a session.
- `update_meeting` takes `roomId` and exactly one kind of change, with no other fields
  beside it except `notifyAttendees` (`false` keeps only a cancellation quiet; never with
  `move`):
  - `move: { startsAt, endsAt }` moves a one-off meeting; attendees are always notified.
  - `occurrence` changes one date of a recurring meeting by `originalStartAt`, that date's
    original series time (not a moved one): `status: "rescheduled"` with `startsAt` and
    `endsAt`, `status: "cancelled"` with an optional `reason`, or `status: "active"` to
    restore it.
  - top-level fields (`title`, `startsAt`, `endsAt`, `timezone`, `recurrence`, settings) edit
    the room or the whole series.
- On `update_uncertain`, the change may already be saved: read the meeting with
  `get_meeting` before any retry.
- On `insufficient_scope`, the connection was granted read only. Tell the user to disconnect
  and reconnect from `/mcp` and to keep the write permission ticked on the consent page.

## Live meetings in Claude Code

`join_live_meeting` seats the user in the room as a visible participant (it starts the
meeting when they host it, or puts them in the waiting room), and its audio and video only
play inside a host that renders the live companion card. Claude Code does not. Prefer
`list_meetings` with view `live` and hand over the join link. If the user still wants the
tool, warn that they will appear in the room without hearing it, and give the `joinUrl`
from the result.

Hosts can run a live meeting from here. `get_meeting_host_state` reads the waiting room,
roster, ids and recording state; read it first and never guess a participant id. The
actions (`moderate_participant`, `remove_meeting_participant`, `set_meeting_role`,
`set_meeting_recording` with `recording` `on` or `off`, `end_meeting_for_everyone`) are
writes and follow the confirmation rule above.

## Errors

Every failure is a text result of the form `<code>: <message>`. The full list with the
action to take is in `references/errors.md`. The ones met most often:

- `unauthorized`: connection expired, reconnect from `/mcp`.
- `insufficient_scope`: read-only connection, reconnect with write.
- `meeting_not_found`, `not_found`, `forbidden`: wrong id or wrong active organization.
- `no_transcript_yet`, `no_summary_yet`, `no_recording`, `moment_not_recorded`: processing not
  finished, nothing uploaded, or the moment lies outside every recording; say so plainly and
  offer what exists.
- `rate_limited`: wait the number of seconds in the message before retrying.

## Limits and data handling

- Requests are rate limited per user; over the limit the server answers 429 with a
  `retry-after` header. Wait that long, and batch reads (one `search_meetings` across
  meetings beats one call per meeting).
- The connector stores only which organization is active per user and client. Its audit log
  records tool name and outcome, never arguments or transcript text.

## Additional resources

- `references/tools.md`: every tool with inputs, ranges, defaults and annotations.
- `references/errors.md`: every error code with meaning and what to tell the user.
