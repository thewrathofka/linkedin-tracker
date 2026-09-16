---
description: Standalone per-BDR LinkedIn connection-request + cold-call lead tracker. Reconciles your LinkedIn inbox against your own Notion tracking page, @-mentions you on new connections, and on send runs builds a lead list from your Salesforce reports and sends connection requests via Claude in Chrome (never via the MCP).
---

Run the workflow in `${CLAUDE_PLUGIN_ROOT}/skills/linkedin-tracker/SKILL.md`.

Read the full SKILL.md file before doing anything else in this run — it contains the hard rules, fixed Notion IDs, database schema, target-persona guide, and the step-by-step flow (Steps 0-7). Do not skip reading it even if you recall it from a prior run; schema options and property names can change.

Pass whatever the rep typed after `/linkedin-tracker` as `$ARGUMENTS` per the "Inputs / args" section: no prompt = a status-and-choice reconcile run; a prompt = answers supplied up front to the list-building interview.

Step 0 resolves which BDR is running this from their own Notion identity and LinkedIn login — this plugin is shared across the team, so never assume it's any specific person.

Hard rule: connection requests are sent ONLY through Claude in Chrome on the rep's real LinkedIn, never through the LinkedIn MCP, and never more than 40 per calendar day. On any CAPTCHA / checkpoint / auth hiccup, stop and hand off.
