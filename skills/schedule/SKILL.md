---
name: schedule
description: Schedule a new Leapcast meeting, or move or cancel an existing one, after confirming the details with the user.
argument-hint: "<what, when, who>"
disable-model-invocation: true
allowed-tools:
  - mcp__plugin_leapcast_leapcast__list_upcoming_meetings
  - mcp__plugin_leapcast_leapcast__get_meeting
  - mcp__plugin_leapcast_leapcast__list_organizations
---

# /leapcast:schedule

Schedule, move or cancel a meeting from a natural-language request: `$ARGUMENTS`.
The write tools (`create_meeting`, `reschedule_meeting`, `cancel_meeting`,
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

1. Resolve the meeting with `list_upcoming_meetings` (match the title; ask if ambiguous),
   then `get_meeting` to read its `roomId` and recurrence.
2. If the meeting is recurring, explain it cannot be moved by the connector and give the room
   link instead.
3. Show old and new times with timezone and ask for a yes; then call `reschedule_meeting`
   with `roomId`, `startsAt`, `endsAt`. Attendees are notified.

## Cancel

1. Resolve the meeting and its `roomId` as above. Never cancel without an explicit meeting.
2. Ask whether attendees should be emailed and for an optional reason. Show title, time and
   the notification choice, and ask for a yes.
3. Call `cancel_meeting`. On `meeting_live`, say the meeting has to end first.

On `insufficient_scope`, the connection is read only: ask the user to disconnect and reconnect
from `/mcp` with the write permission. On `unauthorized`, ask them to sign in from `/mcp`.
