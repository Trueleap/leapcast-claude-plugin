---
name: meetings
description: Show the user's upcoming Leapcast meetings, who is live right now, or pending meeting invitations.
argument-hint: "[days | live | invites]"
disable-model-invocation: true
allowed-tools:
  - mcp__plugin_leapcast_leapcast__list_upcoming_meetings
  - mcp__plugin_leapcast_leapcast__list_meeting_invites
  - mcp__plugin_leapcast_leapcast__who_is_live
  - mcp__plugin_leapcast_leapcast__list_organizations
---

# /leapcast:meetings

Show the user's Leapcast schedule. Argument: `$ARGUMENTS`.

## Steps

1. Interpret the argument:
   - empty: upcoming meetings for the next 14 days, then pending invitations;
   - a number: upcoming meetings for that many days (1–62);
   - `live`: only `who_is_live`;
   - `invites`: only `list_meeting_invites`.
2. Call the matching tools. On `unauthorized`, tell the user to sign in from `/mcp`
   (pick **leapcast**) and stop.
3. Present results as a short list, one line per meeting: title, day and local time,
   status, and the join link. Say which organization the list belongs to (from the result)
   and mention `/leapcast:meetings` again if they want another window.
4. If the list is empty, say so and offer `list_organizations` in case the wrong
   organization is active.
5. For pending invitations, list them with the meeting title and time and ask whether to
   accept, tentatively accept or decline any. Only respond after an explicit yes; that is
   a write and belongs to the connector's confirmation rule.

Do not join meetings from this skill. Keep the answer under the length of the list itself.
