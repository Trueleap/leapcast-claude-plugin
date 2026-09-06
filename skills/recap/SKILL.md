---
name: recap
description: Summarize a past Leapcast meeting and, when given a phrase, find where it was said and link the recording at that moment.
argument-hint: "[meeting title or id] [\"phrase\"]"
disable-model-invocation: true
allowed-tools:
  - mcp__plugin_leapcast_leapcast__list_past_meetings
  - mcp__plugin_leapcast_leapcast__get_meeting_summary
  - mcp__plugin_leapcast_leapcast__get_meeting_transcript
  - mcp__plugin_leapcast_leapcast__search_transcript
  - mcp__plugin_leapcast_leapcast__search_meetings
  - mcp__plugin_leapcast_leapcast__play_recording
  - mcp__plugin_leapcast_leapcast__list_recordings
  - mcp__plugin_leapcast_leapcast__get_meeting
  - mcp__plugin_leapcast_leapcast__list_organizations
---

# /leapcast:recap

Recap a past meeting. Argument: `$ARGUMENTS` (a meeting title fragment or id, optionally
followed by a quoted phrase to locate).

## Steps

1. Resolve the meeting. If the argument looks like an id, use it. Otherwise call
   `list_past_meetings` (30 days, raise `days` if nothing matches) and match the title
   case-insensitively. With no argument, take the most recent meeting that has a ready
   transcript. If several titles match, ask which one before continuing.
2. Note the `transcriptId` from the `transcripts[]` entry of that meeting; pass it to the
   tools below so summary, search and playback refer to the same session.
3. Call `get_meeting_summary`. Present overview, key points, decisions and action items in
   that order, compact. On `no_summary_yet`, say the summary is not ready and offer the first
   page of `get_meeting_transcript` instead.
4. If a phrase was given, call `search_transcript` with it. Show the matching turns with
   speaker and timestamp. Then call `play_recording` with the best hit's `at` and the
   `transcriptId`, and give the user the direct link ending in `#t=<seconds>` (build it from
   `playbackUrl` and `startAtSeconds` if the text does not already contain it). If the
   returned start sits at the end of the recording or the recording started after the
   moment, the session has more than one recording: call `list_recordings` for the meeting,
   choose the one whose start and duration contain the moment, and play again with its
   `recordingId`. On `no_recording`, say the meeting has no recording and keep the hits.
5. If no phrase was given, end by offering to search the transcript for one.

Do not paginate the whole transcript unless the user asks. On `unauthorized`, tell the user
to sign in from `/mcp` and stop. On `meeting_not_found`, offer `list_organizations` in case
another organization is active.
