# Leapcast connector error codes

Every failed call returns text `<code>: <message>` with `isError` set. Read the code, then act
as below. Never retry a write blindly.

## Connection and permission

| Code | Meaning | What to do |
| --- | --- | --- |
| `unauthorized` | Access token expired or revoked | Ask the user to open `/mcp`, choose leapcast and sign in again |
| `insufficient_scope` | Connection was granted read only; a write or `join_live_meeting` was attempted | Ask the user to disconnect and reconnect from `/mcp`, keeping the write permission on the consent page |
| `forbidden` | The user cannot access that item in the active organization | Check the id; offer `list_organizations` and `switch_organization` |
| `not_a_member` | `switch_organization` got an id the user does not belong to | Call `list_organizations` and use one of its ids |

## Lookups

| Code | Meaning | What to do |
| --- | --- | --- |
| `meeting_not_found` | No visible meeting with that id | Confirm the id from a list tool; check the active organization |
| `not_found` | Item missing in the active organization, or a `transcriptId` / `recordingId` that does not belong to the meeting | Re-resolve the id from `list_past_meetings` or `list_recordings` |
| `no_transcript_yet` | No session of the meeting has a ready transcript | Say processing is not finished; suggest trying after the meeting ends |
| `no_summary_yet` | Transcript exists but the summary is not ready, or none ready at all | Offer `get_meeting_transcript` |
| `no_recording` | No uploaded recording for the meeting | Say so; offer the transcript or summary |

## Writes and live

| Code | Meaning | What to do |
| --- | --- | --- |
| `invalid_request` | Bad input: `at` not parseable, `endsAt` before `startsAt`, or the server rejected a field | Fix the input and confirm again before retrying |
| `meeting_live` | `cancel_meeting` on a room with a session in progress | End the meeting first; if empty, wait a minute and retry |
| `not_live` | `join_live_meeting` on a meeting with nobody in it | Use `who_is_live`; try again once it starts |
| `unsupported_provider` | The meeting runs on a provider the in-chat companion cannot join | Give the join link from the message |

## Service

| Code | Meaning | What to do |
| --- | --- | --- |
| `rate_limited` | Leapcast is throttling; message says how many seconds to wait | Wait that long, then retry once |
| `upstream_unavailable` | Leapcast API temporarily down | Retry after a few seconds; report if it persists |
| `invalid_response` | Unexpected response from Leapcast | Retry once; ask the user to report it if it repeats |

## HTTP-level responses (seen by the client, not as tool results)

- 401 with `WWW-Authenticate`: no or bad token; Claude Code starts sign-in.
- 403 `insufficient_scope`: token lacks the read permission; reconnect.
- 429 `rate_limited` with `retry-after: 60`.
