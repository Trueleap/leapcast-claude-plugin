# Leapcast connector tools

All tools need the read permission; the ones marked **write** also need the write permission.
Prefix every name with `mcp__plugin_leapcast_leapcast__` when calling it from this plugin.
Annotations: ro = read-only, d = destructive, i = idempotent, ow = open world (reaches people
outside Leapcast: email, chat, polls).

## Organizations

| Tool | Purpose | Inputs | Annotations |
| --- | --- | --- | --- |
| `list_organizations` | Organizations the user belongs to and which one is active | none | ro, i |
| `switch_organization` | Make one organization active for later calls | `organizationId` | i |

## Meetings

| Tool | Purpose | Inputs | Annotations |
| --- | --- | --- | --- |
| `list_meetings` | Meetings in the active organization, with join links. `view`: `upcoming` (scheduled from now), `instant` (unscheduled rooms), `past` (attended or hosted, newest first, with `transcripts[]`: `meetingId`, `transcriptId`, `transcriptStatus`, `summaryStatus`), `live` (live now or in their scheduled window, with participant counts), `invites` (unanswered invitations, with `participantId`) | `view` (default `upcoming`), `days` (upcoming 1–62, default 14; past 1–365, default 30), `limit` (default 20, at most 50; invites default 50, at most 100), `cursor` | ro, i |
| `get_meeting` | Exactly one of: `meetingId` → schedule, host, description, live state, latest session, the user's invite status, `roomId`; `roomId` → settings, agenda, invitees and roles, recurrence, transcript settings, plan with topic notes, join and Leapcast Live watch links | `meetingId` or `roomId` | ro, i |

## Transcripts and summaries

| Tool | Purpose | Inputs | Annotations |
| --- | --- | --- | --- |
| `get_meeting_summary` | Overview, key points, decisions, action items | `meetingId`, `transcriptId` (optional) | ro, i |
| `get_meeting_transcript` | Timestamped speaker turns in pages, or with `format: "text"` the redacted TXT export (8,000 characters per page); continue with `offset` = `nextOffset` | `meetingId`, `transcriptId` (optional), `format` `turns` (default) or `text`, `offset` ≥ 0, `limit` 1–200 (default 200, turns only) | ro, i |
| `search_meetings` | Transcript turns containing a phrase. With `meetingId`: one meeting's transcript. Without: meeting history in the active organization, newest first | `query` 1–200, `meetingId`, `transcriptId` (with `meetingId`), `mode` `substring` (default, case-insensitive) or `semantic` (ranked, not exhaustive; not with `meetingId`), `days` (omit for all history), `meetings` 1–20 (default 10), `limit` 1–100 (default 25), `cursor`, `skipFailedRun` (history only) | ro, i |

## Recordings

| Tool | Purpose | Inputs | Annotations |
| --- | --- | --- | --- |
| `list_recordings` | Recordings across all the user's organizations, newest first, with playback links | `meetingId`, `search` (title text), `limit` 1–50 (default 10), `cursor` | ro, i |
| `play_recording` | Playback link cued to a moment; returns `playbackUrl`, `startAtSeconds`, `startedAt`, `durationSeconds` and a text link ending in `#t=<seconds>`. Picks the session recording whose window contains the moment, or answers `moment_not_recorded`; pass `recordingId` from `list_recordings` to force one | `meetingId`, `at` (string: `12:34`, `1:02:03` or seconds such as `"754"`), `transcriptId`, `recordingId` | ro, i |

## Content Studio and clips

| Tool | Purpose | Inputs | Annotations |
| --- | --- | --- | --- |
| `get_content_studio` | Studio rights, recorded dates (`meetingSessionId` is the `occurrenceId`) and latest run counts, or a run's post drafts | `meetingId`, `view` `status` (default) or `posts`, `occurrenceId`, `runId` (posts only) | ro, i |
| `list_meeting_clips` | A meeting's clips, meeting-clock ranges, states and clip posts | `meetingId`, `occurrenceId`, `runId` | ro, i |
| `cut_meeting_clip` **write** | Cut 30–60 whole seconds of one recorded date into a draft clip; the same request returns the same clip | `meetingId`, `occurrenceId`, `startSeconds`, `endSeconds` (meeting clock, as in transcripts), `title` ≤ 120 | i |
| `make_gif` **write** | Silent 3–6 s GIF from a ready clip, behind a private ten-minute link | `meetingId`, `clip`: `runId`, `clipId`, `atSeconds` (default 0), `durationSeconds` 3–6 (default 5) | i |
| `get_clip_download_link` **write** | Private ten-minute MP4 link for a ready clip | `meetingId`, `runId`, `clipId` | i |
| `generate_studio_content` **write** | AI clip suggestions, quote cards or stat tiles, or post drafts for one recorded date; never publishes | `meetingId`, `kind` `clips`, `cards` or `posts`; clips and posts need `occurrenceId`; cards need exactly one of `occurrenceId` or `runId`; `more` (clips only), `cardKind` `quote_card` or `stat_tile` (cards only) | none |
| `edit_studio_post` **write** | Replace a post's full text; resets approval to draft | `meetingId`, `runId`, `postId`, `newText`, `subject` (email posts) | d, i |
| `review_studio_post` **write** | Keep, drop or reset a post | `meetingId`, `postId`, `approval` `approved`, `rejected` or `draft` | i |

## Live meetings and host controls

| Tool | Purpose | Inputs | Annotations |
| --- | --- | --- | --- |
| `join_live_meeting` **write** | Join (or, as host, start) a meeting as the user; waiting-room rules apply. Playback needs a host that renders cards (not Claude Code) | `meetingId` (or short join code) | i |
| `get_meeting_host_state` | What the user may control: waiting room, roster with ids and roles, recording state; with `sessionId` also capture policy | `meetingId`, `sessionId` (optional) | ro, i |
| `moderate_participant` **write** | Admit or reject one waiting person, or mute microphones / turn off cameras | `meetingId`, `sessionId`, `action` `admit`, `reject`, `mute_audio` or `disable_video`, `participantIds` (one for admit/reject) | none |
| `remove_meeting_participant` **write** | Remove one person from the live session | `meetingId`, `participantId` | d, i |
| `set_meeting_role` **write** | Make a participant co-host, presenter or viewer; they are emailed | `meetingId`, `userId`, `role` `co_host`, `presenter` or `viewer` | d, i, ow |
| `set_meeting_recording` **write** | Start or stop recording the live session | `meetingId`, `recording` `on` or `off` | i |
| `end_meeting_for_everyone` **write** | End the live session for everyone | `meetingId` | d, ow |
| `update_session_capture` **write** | Turn transcript and summary capture on or off for a session | `meetingId`, `sessionId`, `transcriptEnabled`, `summaryEnabled` | d, i |
| `update_meeting_plan` **write** | Replace the room's whole plan and topic notes | `roomId`, `plan` | d, i |

## Room chat and polls

| Tool | Purpose | Inputs | Annotations |
| --- | --- | --- | --- |
| `read_room_activity` **write** | Read room chat (newest first) or polls; write because it may open chat membership | `meetingId`, `view` `chat` (default) or `polls`, `before` (chat), `offset` (polls), `limit` 1–50 | i |
| `send_room_chat` **write** | Send a chat message to the room, guests included | `meetingId`, `text` ≤ 4000, `requestId` (ULID, optional) | d, ow |
| `create_room_poll` **write** | Create a poll and post it to chat | `meetingId`, `question`, `options` 2–20, `allowMultiple` | d, ow |
| `vote_room_poll` **write** | Vote as the user | `meetingId`, `pollId`, `optionId` | d, i, ow |
| `close_room_poll` **write** | Close a poll to further votes | `meetingId`, `pollId` | d, i |

## Scheduling

| Tool | Purpose | Inputs | Annotations |
| --- | --- | --- | --- |
| `create_meeting` **write** | Schedule a meeting, or with `instant: true` an unscheduled room; invitees receive email | `title` 1–200, `description` ≤ 4000, `startsAt` and `endsAt` ISO-8601 with offset (required unless `instant`; `instant` refuses them, `timezone` and `recurrence`), `timezone` IANA, `recurrence` RRULE ≤ 500, `invitees` ≤ 50 emails, `instant`, optional settings (`accessPolicy`, `audioOnly`, `recordOnStart`, `transcriptEnabled`, `summaryEnabled`, `agenda`…) | d, ow |
| `update_meeting` **write** | One change per call: `move` a one-off meeting, change one `occurrence` of a recurring meeting, or edit settings / the whole series | `roomId` plus one of `move: { startsAt, endsAt }`, `occurrence: { originalStartAt, status, startsAt, endsAt, title, reason }` (`startsAt` and `endsAt` required for `rescheduled` and refused otherwise; `reason` only for `cancelled`), or top-level `title`, `description`, `startsAt`, `endsAt`, `timezone`, `recurrence` (null clears) and settings; `notifyAttendees` (not with `move`) | d, ow |
| `cancel_meeting` **write** | Cancel by room id; live meetings must end first | `roomId`, `notifyAttendees` (default true), `reason` ≤ 500 | d, i, ow |
| `invite_to_meeting` **write** | Email invitations; already-invited people skipped | `meetingId`, `emails` 1–50 | d, i, ow |
| `respond_to_invite` **write** | Accept, tentatively accept or decline | `participantId`, `response` = `confirmed`, `tentative` or `declined` | d, i, ow |

## Help and QR codes

| Tool | Purpose | Inputs | Annotations |
| --- | --- | --- | --- |
| `leapcast_help` | How Leapcast works, from reviewed help articles and platform facts; `askOriUrl` when nothing answers; brand kits with `topic: "brand"` | `query` 1–200, `topic` `product` (default) or `brand` | ro, i |
| `make_qr_code` | QR code (PNG and SVG) for a meeting's join or Leapcast Live watch page, or a leapcast.ai / leapcast.live link | `meetingId` with `link` `join` (default) or `watch`, or `url` | ro, i |

## Result shapes worth knowing

- Meeting lists (all views but `invites`, which lists `participantId` and `roomId` per invitation): one line per meeting, `- <title> [<status>] (meetingId: <id>, roomId: <id>) — <start> – <end> UTC · [Join](<url>)`, then `nextCursor: "…"` (or `null`).
- Transcript pages: `[mm:ss] Speaker: text`, `(?)` marks low confidence, ending with `Continue with offset N.` or `End of transcript.`
- `play_recording`: `Playing "<title>" (recordingId: <id>, meetingId: <id>) recorded <date>, <n> min from <h:mm:ss>. Direct link: <url>#t=<seconds>`.
