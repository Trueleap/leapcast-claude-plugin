---
name: schedule
description: Schedule a new Leapcast meeting, or move or cancel an existing one, after confirming the details with the user.
argument-hint: "<what, when, who>"
disable-model-invocation: true
allowed-tools:
  - mcp__plugin_leapcast_leapcast__list_meetings
  - mcp__plugin_leapcast_leapcast__get_meeting
  - mcp__plugin_leapcast_leapcast__list_organizations
---

# /leapcast:schedule

Schedule, move or cancel a meeting from a natural-language request: `$ARGUMENTS`.
The write tools (`create_meeting`, `update_meeting`, `cancel_meeting`,
`invite_to_meeting`) are deliberately not pre-approved: each call prompts the user, which is
the second confirmation after the one below.

## Decide the branch

- New meeting: no existing meeting is named → **create**.
- The request names an existing meeting and a new time → **reschedule**.
- The request says cancel → **cancel**.

## Create

1. Collect title, start, end or duration (default 30 minutes), invitee emails, optional
   description and recurrence. Ask only for what is missing.
2. Determine the timezone: use the one the user states; otherwise use the system timezone
   and state it in the confirmation.
3. Build `startsAt` and `endsAt` as ISO-8601 with the UTC offset of that timezone
   (`2026-09-08T10:00:00+01:00`), and pass the IANA `timezone`.
4. Show the proposed meeting on one line each: title, date, start–end with timezone,
   invitees (they will receive an email), recurrence. Ask for a yes.
5. On yes, call `create_meeting` and return the join link and meeting id.

## Reschedule

1. Resolve the meeting with `list_meetings` view `upcoming` (match the title; ask if
   ambiguous), then `get_meeting` with its `roomId` to read the series start, timezone and
   recurrence.
2. Show old and new times with timezone and ask for a yes; then call `update_meeting` with
   `roomId` and one change:
   - one-off meeting: `move: { startsAt, endsAt }` and nothing else. Attendees are always
     notified.
   - one date of a recurring meeting: `occurrence: { originalStartAt, status: "rescheduled",
     startsAt, endsAt }`, where `originalStartAt` is that date's original series time (from
     the series start and recurrence), not a moved time. If you cannot work it out with
     certainty, give the room link instead.
   - the whole series: top-level `startsAt`, `endsAt` and `timezone`, only when the user
     says every date moves.
3. On `update_uncertain`, read the meeting again before any retry.

## Cancel

1. Resolve the meeting and its `roomId` as above. Never cancel without an explicit meeting.
2. Ask whether attendees should be emailed and for an optional reason. Show title, time and
   the notification choice, and ask for a yes.
3. Call `cancel_meeting` with `roomId`, `notifyAttendees` and `reason`; on `meeting_live`,
   say the meeting has to end first. To cancel only one date of a recurring meeting, call
   `update_meeting` with `occurrence: { originalStartAt, status: "cancelled", reason }` and
   `notifyAttendees` instead.

On `insufficient_scope`, the connection is read only: ask the user to disconnect and reconnect
from `/mcp` with the write permission. On `unauthorized`, ask them to sign in from `/mcp`.
