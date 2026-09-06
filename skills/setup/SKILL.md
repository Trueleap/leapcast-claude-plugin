---
name: setup
description: Connect Claude Code to Leapcast, verify the connection, and fix common sign-in problems.
argument-hint: ""
disable-model-invocation: true
allowed-tools:
  - mcp__plugin_leapcast_leapcast__list_organizations
  - mcp__plugin_leapcast_leapcast__switch_organization
---

# /leapcast:setup

Get the Leapcast connection working and confirm it.

## Steps

1. Explain in two sentences: the plugin talks to the hosted Leapcast MCP server over OAuth;
   the user signs in with their normal Leapcast account in the browser and nothing else is
   configured. There are no API keys.
2. Ask the user to run `/mcp`, select **leapcast**, and choose sign in. The browser opens the
   Leapcast consent page listing two permissions: read (meetings, transcripts, summaries,
   recordings) and write (schedule and change meetings). Recommend allowing both; without
   write, `/leapcast:schedule` and invitation responses will fail with `insufficient_scope`.
3. After they report back, call `list_organizations`. Show the organizations and which one
   is active. If the user wants another one, call `switch_organization` with its id.
4. Suggest a first command: `/leapcast:meetings` or `/leapcast:recap`.

## Troubleshooting

- Consent page says "This app" instead of a client name: the client registration failed.
  Ask the user to retry sign-in from `/mcp`.
- `unauthorized` on every call: the connection expired or was revoked. Reconnect from `/mcp`.
- `insufficient_scope`: reconnect and keep the write permission ticked.
- Meetings missing: the wrong organization is active; use `list_organizations` and
  `switch_organization`.
- Sign-in never opens a browser: the user can copy the URL Claude Code prints. Loopback
  redirects on any local port are accepted by Leapcast.
- The server only answers `POST` on `/mcp`; a browser visit to the URL showing "method not
  allowed" is expected.

Leapcast support: support@leapcast.ai. Privacy policy: https://trueleapinc.com/privacy-policy.
