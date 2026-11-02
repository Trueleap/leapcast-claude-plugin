---
name: meetings
description: Show the user's upcoming Leapcast meetings, who is live right now, or pending meeting invitations.
argument-hint: "[days | live | invites]"
disable-model-invocation: true
allowed-tools:
  - mcp__plugin_leapcast_leapcast__list_meetings
  - mcp__plugin_leapcast_leapcast__get_meeting
  - mcp__plugin_leapcast_leapcast__list_organizations
  - mcp__plugin_leapcast_leapcast__make_qr_code
---

# /leapcast:meetings

Show the user's Leapcast schedule. Argument: `$ARGUMENTS`.

## Steps

1. Interpret the argument:
   - empty: upcoming meetings for the next 14 days, then pending invitations;
   - a number: upcoming meetings for that many days (1–62);
   - `live`: only meetings live now or inside their scheduled window;
   - `invites`: only pending invitations.
2. Call `list_meetings` once per list with `view` `upcoming` (and `days`), `live` or
   `invites`. On `unauthorized`, tell the user to sign in from `/mcp` (pick **leapcast**)
   and stop.
3. Present results as a short list, one line per meeting: title, day and local time,
   status, and the join link. Say which organization the list belongs to (from the result)
   and mention `/leapcast:meetings` again if they want another window.
4. If the list is empty, say so and offer `list_organizations` in case the wrong
   organization is active.
5. For pending invitations, list them with the meeting title and time and ask whether to
   accept, tentatively accept or decline any. Only respond after an explicit yes; that is
   a write and belongs to the connector's confirmation rule.
6. If the user wants a QR code for a meeting, call `make_qr_code` with its `meetingId`.

Do not join meetings from this skill. Keep the answer under the length of the list itself.
