# linkedin-tracker

A standalone, per-BDR LinkedIn connection-request + cold-call lead tracker for the Superside BDR team.

It reconciles your **own** LinkedIn inbox (who accepted, who replied) against your **own** Notion tracking page, @-mentions you to send a same-day first message when someone connects, and on send runs builds a lead list from your downloaded Salesforce reports and sends connection requests through **Claude in Chrome** on your real LinkedIn login. It never sends connection requests through the MCP.

**Fully self-contained.** It does not depend on `linkedin-crm-sync`, `convohub-sync`, or any other plugin. It ships its own read-only LinkedIn MCP and its own permission deny-list, and keeps its own Notion page and databases — separate from Conversation Hub / All Accounts / All Meetings.

## What's in the box

| File | Purpose |
|---|---|
| `.mcp.json` | The `linkedin` MCP server (`uvx mcp-server-linkedin`), used for **reads only**. |
| `.claude/settings.json` | Permission deny-list. Blocks `connect_with_person` and every LinkedIn search tool at the permission layer, so the guardrails travel with the install. |
| `skills/linkedin-tracker/SKILL.md` | The command logic. |

## What each BDR still provides at runtime

These are per-person and can't ship inside a plugin:

- **Claude in Chrome**, logged into your own LinkedIn (used for the send path only).
- Your own **Notion** access (the tracking page lives under "BDR pages").
- Your own **Salesforce** access, plus your two downloaded report exports (leads report + owned-accounts report) when building a list.

## Using it

```
/linkedin-tracker
```

- **No prompt** → reconcile the inbox, report where any open lead list stands, and ask whether to continue sending, just reconcile, or build a new list.
- **With a prompt** (e.g. `add new people`, a campaign, a note) → treated as answers to the list-building interview; it asks for anything still missing.

First run for a BDR provisions their tracking page and databases and asks for both Salesforce report links.

## Safety

- Connection requests go **only** through Claude in Chrome, never the MCP.
- Hard cap of **40 connection requests per calendar day**, counted across all runs.
- On any CAPTCHA / checkpoint / auth hiccup: it stops and hands off. It never solves a CAPTCHA.
- Salesforce and the calendar are read-only.
