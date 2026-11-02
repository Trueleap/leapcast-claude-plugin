# Leapcast connector error codes

Every failed call returns text `<code>: <message>` with `isError` set. Read the code, then act
as below. Never retry a write blindly.

## Connection and permission

| Code | Meaning | What to do |
| --- | --- | --- |
| `unauthorized` | Access token expired or revoked | Ask the user to open `/mcp`, choose leapcast and sign in again |
| `insufficient_scope` | Connection was granted read only; a write, `join_live_meeting` or `read_room_activity` was attempted | Ask the user to disconnect and reconnect from `/mcp`, keeping the write permission on the consent page |
| `forbidden` | The user cannot access that item in the active organization | Check the id; offer `list_organizations` and `switch_organization` |
| `not_a_member` | `switch_organization` got an id the user does not belong to | Call `list_organizations` and use one of its ids |

## Lookups

| Code | Meaning | What to do |
| --- | --- | --- |
| `meeting_not_found` | No visible meeting with that id | Confirm the id from a list tool; check the active organization |
| `not_found` | Item missing in the active organization, or a `transcriptId` / `recordingId` that does not belong to the meeting | Re-resolve the id from `list_meetings` view `past` or `list_recordings` |
| `no_transcript_yet` | No session of the meeting has a ready transcript | Say processing is not finished; suggest trying after the meeting ends |
| `no_summary_yet` | Transcript exists but the summary is not ready, or none ready at all | Offer `get_meeting_transcript` |
| `no_recording` | No uploaded recording for the meeting | Say so; offer the transcript or summary |
| `moment_not_recorded` | The session's recordings all started later or ended earlier than the requested moment | Say the moment was not recorded; offer `list_recordings` to show what exists, or another moment |

## Writes and live

| Code | Meaning | What to do |
| --- | --- | --- |
| `invalid_request` | Bad input: `at` not parseable, `endsAt` before `startsAt`, more than one kind of change in `update_meeting`, or the server rejected a field | Fix the input and confirm again before retrying |
| `update_uncertain` | `update_meeting` may already have saved the change, but the result is unclear | Read the meeting with `get_meeting` before any retry; never repeat the edit blindly |
| `meeting_live` | `cancel_meeting` on a room with a session in progress | End the meeting first; if empty, wait a minute and retry |
| `not_live` | `set_meeting_recording` or `end_meeting_for_everyone` on a meeting with no live session | Check `list_meetings` view `live`; try again once it starts |

## Clips and host controls

Their messages say what to do. Clip and Studio tools answer codes such as `studio_off`,
`not_ready`, `clip_exists`, `range_not_recorded`, `recording_pending`, `clip_not_ready` and
`gif_outside_clip`; list the clips or pick other seconds as the message says. Host controls
answer `not_allowed` when the user is not the host or a co-host or the meeting is not live,
and `already_recording` when a recording is running; read `get_meeting_host_state` first.

## Service

| Code | Meaning | What to do |
| --- | --- | --- |
| `rate_limited` | Leapcast is throttling; message says how many seconds to wait | Wait that long, then retry once |
| `upstream_unavailable` | Leapcast API temporarily down | Retry after a few seconds; report if it persists |
| `outcome_unknown` | A write may already have completed; Leapcast's answer did not arrive | Check the current state before retrying; never repeat it blindly |
| `invalid_response` | Unexpected response from Leapcast; a write may still have gone through | Check the current state before retrying; ask the user to report it if it repeats |
| `internal_error` | The tool could not complete | Check the current state before retrying |

## HTTP-level responses (seen by the client, not as tool results)

- 401 with `WWW-Authenticate`: no or bad token; Claude Code starts sign-in.
- 403 `insufficient_scope`: token lacks the read permission; reconnect.
- 429 `rate_limited` with `retry-after: 60`.
