# Leapcast plugin for Claude Code

Meetings, summaries, transcripts, recordings and live rooms from [Leapcast](https://leapcast.ai),
inside Claude Code. The plugin connects to the hosted Leapcast MCP server; you sign in with your
Leapcast account in the browser. No API keys, no local services, no telemetry.

## Install

```text
/plugin marketplace add Trueleap/leapcast-claude-plugin
/plugin install leapcast@leapcast
```

Then start a new session, run `/mcp`, choose **leapcast** and sign in. The consent page asks for
two permissions: read (meetings, transcripts, summaries, recordings) and write (schedule and
change meetings). `/leapcast:setup` walks through this and checks the connection.

## What you can do

- `/leapcast:meetings` — upcoming meetings, who is live, pending invitations, a meeting's QR code.
- `/leapcast:recap "design review" "budget"` — summary of a past meeting, the turns where a phrase
  was said, and a recording link that opens at that moment.
- `/leapcast:schedule 30 min onboarding sync next Tuesday 10:00 with ana@example.com` — creates,
  moves or cancels meetings after showing you the details and asking first.
- Or just ask: "What meetings do I have this week?", "Summarize yesterday's standup",
  "Play the moment we talked about the launch date", "Cut a clip of that moment",
  "How do Leapcast recordings work?", "Which of my organizations is active?".

The connector exposes 38 tools; the skill `leapcast-connector` teaches Claude which id each tool
needs, how transcript search chains into playback and clips, and what every error means.

## How it works

- MCP server: `https://mcp.leapcast.ai/mcp` (Streamable HTTP), the same connector submitted to
  the Claude and ChatGPT directories.
- Authentication: OAuth 2.1 with dynamic client registration and PKCE, handled by Claude Code.
  Access tokens last one hour and refresh automatically. Disconnect any time from `/mcp`.
- Data: tools act as you, inside one active Leapcast organization. The connector stores only
  which organization is active for this connection. Writes (scheduling, invitations,
  cancellations, clips, host controls) always ask you first.

## Other clients

Claude.ai, ChatGPT and any other MCP client can use the same server: see
[leapcast.ai/connect](https://leapcast.ai/connect).

## Support and legal

- Support: support@leapcast.ai
- Privacy policy: https://trueleapinc.com/privacy-policy
- Terms: https://trueleapinc.com/terms-and-conditions

## Development

```bash
claude plugin validate --strict .
claude --plugin-dir .
```

MIT © TrueLeap Inc
