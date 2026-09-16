---
name: linkedin-tracker
description: Standalone LinkedIn connection-request + cold-call lead tracker for a BDR. Reads the BDR's own LinkedIn inbox via the LinkedIn MCP (like /linkedin-sync) and reconciles who has accepted/replied against a per-BDR Notion page under "BDR pages", @-mentioning the BDR to send a same-day first message when someone connects. On a send run it interviews the BDR (note, campaign, scope, size), builds the candidate lead list from their downloaded Salesforce reports, then before sending asks ATL vs ATL+BTL and qualifies each person's persona live in Claude in Chrome, sends those connection requests via Chrome (never via the MCP), and logs everything so later runs resume where the last one stopped. Fully independent of the Conversation Hub / linkedin-crm-sync plugin. Use when the BDR runs "/linkedin-tracker".
---

# /linkedin-tracker — LinkedIn & Cold-Call Lead Tracker

A standalone, per-BDR tracker. It is **independent of the `linkedin-crm-sync` / Conversation Hub plugin**: its own Notion page and databases, its own state, its own bundled LinkedIn MCP and deny-list. It reads the LinkedIn inbox the same way `/linkedin-sync` does for **reads only**. **All list-building lives in this command** and is always driven by the BDR's answers to the questions below — the BDR never has to paste a list or a spec.

## Hard rules (never violate)

1. **`connect_with_person` is NEVER called through the LinkedIn MCP.** Connection requests are sent **only through Claude in Chrome** (`mcp__claude-in-chrome__*`) on the BDR's real logged-in LinkedIn. The MCP deny-list keeps `connect_with_person` blocked.
2. **Reads use the LinkedIn MCP** (`get_inbox`, `get_conversation`, `get_person_profile`, `get_my_profile`), never Chrome.
3. **Never touch** Conversation Hub, All Meetings, All Accounts, or Monthly Scoreboard. This command only ever writes to the BDR's own tracking page under **BDR pages**.
4. **Never guess** identity, company, persona, or campaign — ask or skip and flag, never invent.
5. **Salesforce is read-only.** Never write to Salesforce or the calendar.
6. **LinkedIn ToS risk is real.** Keep runs infrequent. On any auth hiccup, CAPTCHA, or checkpoint: **stop and hand off to the BDR** — never solve a CAPTCHA, never retry harder.
7. **Never send more than 40 connection requests per day. Hard cap, no exceptions.** Count sends across all runs on the same calendar day (lead-list docs + main-DB `Date of connection request sent`). If a run would push the day's total over 40, send only up to 40, stop, and report the remainder as carried over.
8. **`Name of campaign` is taken from the Salesforce leads report, never invented** (see the property note). The **lead-list name** Claude creates is a *separate* thing from the campaign name.
9. **Build the 200 list FIRST, then qualify persona at send time.** The list is the most-recent content-downloaders (with LinkedIn), built from the BDR's note/campaign/scope/size answers — **no persona/title filter at build.** The ATL-vs-BTL question is asked in the send loop, just before sending, and each person's persona is judged **live in Chrome**, never from the report title.
10. **First run = no tracking page yet for this BDR.** Before doing anything else, **ALWAYS ask the BDR for both Salesforce reports — the leads report and the owned-accounts report — as BOTH (a) the report link AND (b) the downloaded export file** (CSV/Excel). Store both links at the top of the new page; work from the downloaded files. Never create the databases, build a list, or send anything until both links are captured.
11. **Lists are built from the BDR's downloaded report exports, not from live Salesforce.** When building or refreshing a list, ask the BDR for the current downloaded leads report (and owned-accounts export) and read the rows from that file. The SF links at the top of the page are for reference/refresh; a live Salesforce MCP read is only an optional cross-check, never the primary source (avoids guessing a saved report's filter).
12. **Every send / add-people run ALWAYS asks for the connection-request note first** — before building a list or sending anything (Hard rule; never send without a BDR-given note). The note is **stored in the lead-list doc**, so on a later/continue run the command reads it from the doc and keeps using the same note.

## LinkedIn MCP tool scope (bundled with this plugin)

- **Allowed (reads):** `get_inbox`, `get_conversation`, `search_conversations`, `get_person_profile`, `get_my_profile`, `close_session`.
- **Denied (must never be called via MCP):** `connect_with_person`, `search_people`, `get_sidebar_profiles`, `search_companies`, `get_company_profile`, `get_company_posts`, `get_company_employees`, `search_jobs`, `get_job_details`, `get_feed`.
- `get_person_profile` is only ever called on a `linkedin_username` pulled from `get_inbox`/`get_conversation` output — never on a name/company from any other source.

This plugin ships its own `linkedin` MCP server (`.mcp.json`) and its own permission deny-list (`.claude/settings.json`), so the guardrails travel with the install — no per-BDR edit of `~/.claude/settings.local.json` is required.

## Fixed location

- **Parent page** ("BDR pages", under GTM Home / Sales): `3dd50ebbbec0804589f2f06f45bd6353`
- Per-BDR page title: **`{First name} LinkedIn and Cold Call Lead Tracking`**

## Inputs / args

Parse `$ARGUMENTS`:

- **No prompt** → **status-and-choice run** (Step 4.5). Reconcile the inbox, report where any open lead list stands ("40 of 200 sent"), and ask the BDR whether to continue sending, just reconcile, or — if no list is open — **build a new list** (which kicks off the Step 5a interview).
- **A prompt** (e.g. "add new people", a campaign, targeting hints, a note) → treat it as *answers supplied up front* to the Step 5a interview. Use whatever it provides, and **ask for anything still missing** (especially the note — Hard rule). It is never required to be complete; the command fills the gaps by asking.

## Databases (created on first run)

### Main database — prospect tracking
| Property | Type | Notes |
|---|---|---|
| Name of prospect | Title | |
| Title | Text | the prospect's *actual* current LinkedIn title (written even if it differs from the report's title) |
| Account name | Text | the prospect's company |
| Name of campaign | Multi-select | **the campaign(s) as they appear on the Salesforce leads report** for that person — never invented. A person can carry several (e.g. multiple guide/report downloads); tag all that apply. Options are the real Salesforce campaign names (guide/report downloads etc.); `Cold` is also an option for an off-list inbox person with no SF campaign (ask the BDR before applying `Cold`). |
| Date of connection request sent | Date | the day the Chrome send happened |
| Note sent | Text | the connection note used |
| Connected? | Checkbox | set when acceptance is detected in the inbox |
| Replied? | Checkbox | set when the prospect replies |
| Date of next message | Date | = the day acceptance is detected (same-day cadence); blank until connected |
| Suggested next message | Text | **left blank in v1** |
| URL LinkedIn | URL | helper for reliable inbox matching |

### Run-tracking database — Claude's run log (below the main DB)
| Property | Type | Notes |
|---|---|---|
| Run | Title | e.g. `Run 2026-09-16 13:20` |
| Run date | Date | |
| Lead list | Text | the **name of the lead-list doc** Claude built and worked this run — the key that ties runs of the same list together and points the next run at the doc to continue from |
| List size | Number | total people in that lead list (e.g. 200) |
| New connections sent | Number | sent in this run |
| Cumulative sent | Number | running total sent for this lead list across all runs |
| Accepted since last run | Number | |
| Replied | Number | |

Progress for a list = the **most recent row with that `Lead list`** (`Cumulative sent` of `List size`). This is what Step 4.5 reads to know where to continue.

## Page layout (top → bottom)

```
{First} LinkedIn and Cold Call Lead Tracking
├─ Owned accounts (Salesforce report): <link>   ← company scope for reconcile + list-building
├─ Leads (Salesforce report): <link>            ← 7,000+ lead pool; carries LinkedIn profile URL, campaign, and Member First Associated Date (ranking field)
├─ Target personas: <reference block>           ← the persona guide below (written on first run)
├─ Lead lists: links to each lead-list doc (named candidate pools), newest first
├─ Main database (inline)
└─ Run-tracking database (inline)
```

Both Salesforce report links are **supplied by the BDR on the first run** and stored at the top, then reused every run.

- **Lead-list doc** = one Notion child page **per lead list** (persists and is updated across runs; never one-per-run). Title = the **lead-list name** Claude assigns when the list is built (distinct from the campaign — e.g. `Guides & Reports Downloads — most recent — 2026-09-16 (200)`). This name goes into the run log's `Lead list`. It contains:
  - a **progress summary** line kept current: `Sent X / {size} · Accepted Y · Issues Z · last worked {date}`;
  - the **full candidate pool** as an ordered checklist (`☐ {Name} — {LinkedIn URL} — {company} — {title}`), **checked** as each request is sent, each checked item tagged with **date sent** and the **note** used;
  - an **Issues** section: anyone skipped/failed (left the company, wrong persona, ambiguous, CAPTCHA) with the reason and the **date of the attempt** (so re-tries are visible);
  - the campaign name and the note text.
  Later runs **update this same doc**. It's linked under **Lead lists** at the top of the page.

## Target personas (marketing & creative) — the qualification guide

Written to the top of the page on first run, and used to qualify every profile in Step 5. **A strong guide, not exhaustive string-matching** — accept clear equivalents, obvious seniority variants, and regional/global spellings. The BDR's Step 5a answers decide *which tiers* are in scope for a given list.

**Senior / ATL tier**
- Marketing: CMO / Chief Marketing Officer, VP / Head / Director / Senior Manager of Marketing, Global Head of Marketing.
- Brand: Chief Brand Officer, VP / Head / Director of Brand, Brand (Marketing) Director, Head of Brand Marketing, Global Brand Director.
- Creative & Design: CCO / Chief Creative Officer, VP / Head / Director of Creative, (Executive / Global) Creative Director, Head / Director / VP of Design, Head of Studio, Studio Director, Head of In-House Agency.
- Content: Head / Director / VP of Content, Content Marketing Director.
- Communications: Chief Communications Officer, VP / Head / Director of Communications, Communications Director, Head / VP of Corporate Comms.
- Growth / Demand / Performance / Digital: Head / Director of Performance Marketing, Head of Growth Marketing, VP of Growth, Head / Director of Demand Generation, Head / Director / VP of Digital Marketing.
- Product Marketing: Head / Director / VP of Product Marketing.
- Marketing / Creative Operations & Production: Head / Director of Marketing Operations, Head / Director of Creative Operations, Head of Creative Production.

**BTL tier — managers & creative/design leads** (same functions: marketing, brand, creative, design, content, comms, growth)
- Managers: Marketing Manager, Senior Marketing Manager, Brand Manager, Senior Brand Manager, Content (Marketing) Manager, Communications Manager, Social Media Manager, Product Marketing Manager, Growth / Performance / Demand Gen / Digital Marketing Manager, Campaign Manager, Marketing Ops Manager, Creative Ops Manager, Studio Manager, Creative Production Manager, Design Manager.
- Leads & senior creatives: Creative Lead, Design Lead, Lead Designer, Senior Designer, Art Director, Senior Art Director, Content Lead, Brand Lead, Senior Copywriter.

**AI variants — qualify in either tier at the matching level**, only when the role clearly sits within marketing/brand/creative/content/comms: Head / VP / Director of AI, Head of Generative AI, AI Creative Director, Head / Director of AI Marketing, Director of Marketing AI / Marketing Automation & AI, Head of Content & AI, Head of AI Studio, Creative AI / Creative Technology Lead, AI Marketing Manager.

**Never qualify:** Coordinators, Specialists, Associates, Assistants, interns; and anyone outside those functions (Sales, Engineering, Finance, HR, general Operations, product-AI/data/eng roles). When genuinely unsure, flag in chat rather than guessing.

## Steps

### Step 0 — Resolve the BDR
- Notion identity: `notion-fetch self` → BDR name + Notion **user id** (for the @mention in Step 4). Derive the first name.
- LinkedIn identity: `get_my_profile()` — confirm which LinkedIn account is logged in. Never assume a fixed person.

### Step 1 — Locate or create the tracking page
- Search under BDR pages (`3dd50ebbbec0804589f2f06f45bd6353`) for `{First} LinkedIn and Cold Call Lead Tracking`.
- **First run (page not found) — Hard rule 10:** the very first thing to do is **ask the BDR for both Salesforce reports — (1) leads report and (2) owned-accounts report — each as a link AND a downloaded export file (CSV/Excel).** Do not create anything until at least both **links** are provided (the downloads can come when the first list is built). Then create the page (`notion-create-pages`, parent = BDR pages), write **both links** + the **Target personas** block at the top, and create both databases (`notion-create-database`). Confirm both links are stored at the top before moving on. Report it was provisioned.
- **Otherwise:** reuse it.

### Step 2 — Load scope
- Read the top-of-page owned-accounts company list, the Target personas block, and the linked lead-list docs (open lists, their progress, campaigns, send dates).

### Step 3 — Load the scope from the BDR's downloaded reports
- Work from the BDR's **downloaded report exports** (Hard rule 12), not live Salesforce. Ask for the current **leads report** export and **owned-accounts** export if not already provided this session. Owned-accounts export → in-scope company list (reconcile filter + a scope option for list-building). Leads report export → the pool lists are built from, and the source of each person's **campaign** label, LinkedIn URL, title, and **Member First Associated Date**.
- The two SF links at the top of the page are for reference/refresh. A live Salesforce MCP read (`getUserInfo` / `soqlQuery`) is only an optional cross-check, read-only; never the primary source, never write.

### Step 4 — Reconcile (THE GATE — always runs first, before any sends)
1. `get_inbox(limit=…)` via MCP. Parse each conversation → participant name + last-message date/snippet. Note truncation if any.
2. For each participant, `get_person_profile(linkedin_username)` (inbox usernames only) → company/title. **Fuzzy-match** company to owned accounts. Skip out-of-scope companies **unless** the person is on a lead-list doc.
3. **Connected detection** (Kali's rule): the connection-request note now appears as a **sent message in the thread with no reply yet** → accepted. For each pending lead-list person now connected: check their item on the doc; set main-DB `Connected?` = true and `Date of next message` = **today**; **post a Notion `@mention` comment** on their row (`notion-create-comment`, mentioning the BDR's Notion user id): "Connected — send your first message today."
4. **Replied detection:** prospect replied → set `Replied?` = true (next message is the BDR's to write).
5. **Campaign attribution (multi-select):** tag `Name of campaign` with **all** the Salesforce campaign(s) that apply to the person (from the leads report / lead-list doc) — never invented. Off-list person with no SF campaign → **ask the BDR** which campaign to apply (existing options **plus `Cold`**) before writing the row.
6. Update lead-list docs' progress summaries; tally `Accepted since last run` and `Replied`.

### Step 4.5 — Status & choice (NO-PROMPT runs)
- Read the run log. **Open list** (most recent row has `Cumulative sent` < `List size`): state it plainly — **"Lead list '<name>': 40 of 200 sent. Continue today, just check the inbox and correct the database, or something else?"** Wait for the choice.
  - *Continue* → Step 5 resuming that list (reuse its doc, note, campaign; no interview needed).
  - *Just reconcile* → skip to Step 6 with 0 sends.
- **No open list** (all complete, or none yet): say so and **ask if the BDR wants to build a new list.** If yes → go to Step 5a (the interview). If no → finish after reconcile.

### Step 5 — Send path
Runs when: the BDR gave a prompt to add people, chose *continue* in 4.5, or chose to *build a new list* in 4.5.

**5a — Establish the lead list (interview-driven; skip the interview if continuing an existing list — but still read that list's stored note from its doc and keep using it).**
Ask the BDR (use anything the prompt already answered; only ask what's missing). **Do NOT ask about personas/ATL-vs-BTL here — that is asked in the send loop, just before sending, and checked live in Chrome (see below).**
1. **Connection-request note — ASK FIRST, before anything else (Hard rule 11).** Never build a list or send without a BDR-given note. This note is stored in the lead-list doc so later/continue runs reuse it automatically.
2. **Campaign (multi-select)** — `Name of campaign` is filled from each lead's real Salesforce campaign(s), never invented. When building a guide/report-download list, add **all** the guide/report-download campaign names found as multi-select options, and tag each person with the specific campaign(s) they belong to (a person may carry several).
3. **Scope** — all leads in the leads report, or only leads at owned accounts (cross-reference the owned-accounts list)? (Note: the leads report may already be owner-scoped — check its filters.)
4. **Ranking & size** — default: **most recent guides/reports downloads**, ranked by the **`Member First Associated Date`** column in the leads report, **most recent first**; take the top **200**. Confirm size. If `Member First Associated Date` isn't present, say so and ask which date to rank by — never rank on an unknown field.

Then build the list **first, WITHOUT any persona/title filter**: read the **downloaded leads report export** the BDR provides (ask for it if not yet given), keep only rows in the chosen guide/report-download campaigns that have a LinkedIn URL, apply the scope, rank by **`Member First Associated Date`** (most recent first), dedupe per person (collect all their campaigns for the multi-select), take the top N. The 200 is simply the most-recent content-downloaders — **persona is NOT judged from the report title** (it's stale); it's judged live on LinkedIn at send time (step 5, per the ATL/BTL choice). Assign a **lead-list name** (distinct from the campaign, e.g. `Guides & Reports Downloads — most recent — {date} ({N})`), create the lead-list doc with the full ordered pool + the note + the campaign, and record `List size`.

Then the send loop:
1. **Ask the persona tier NOW, before sending anyone: "ATL only, or ATL + BTL?"** (ATL = senior tier; BTL = also managers + creative/design leads — see Target personas.) This choice governs the live persona check below. Ask it here every send run, not at list-build.
2. **Daily budget** = min(requested/remaining-in-list, 40 − already-sent-today). If 0, stop and say the cap is reached.
3. **Echo the plan** (list name, campaign, tier choice, note, how many will send given the cap) and **get an explicit go-ahead.**
4. Work the list in order up to the budget. For each person, **qualify then send via Claude in Chrome** (navigate to their LinkedIn URL). Never call `connect_with_person` via MCP. On any CAPTCHA/checkpoint, stop and hand off.
   - **Still at the company?** Check the company by the profile heading; if unclear, scroll to Experience. If they've left → don't send, flag, log as an issue with today's date.
   - **Right persona? (checked LIVE in Chrome, not from the report.)** Read the title under their name (scroll if ambiguous) and judge it against the Target personas at the chosen tier (ATL only, or ATL + BTL). Not a target → don't send, flag, log as issue. Always write the **actual** current LinkedIn title into the main-DB `Title`.
   - A person skipped for persona/left-company does **not** count against the daily budget; move to the next on the list.
4. As each send succeeds: check it off the doc (with date + note); create the main-DB row (`Name of prospect`, `Title`, `Account name`, `Name of campaign`, `Date of connection request sent` = today, `Note sent`, `URL LinkedIn`, checkboxes unchecked, dates blank). Stop when the budget is exhausted; report the remainder as carried over.
5. Update the lead-list doc (progress summary, checkoffs, issues) and ensure it's linked under **Lead lists**.

### Step 6 — Write the run-tracking row
- `Run date` = now, `Lead list` = the list worked (blank for a pure reconcile), `List size`, `New connections sent`, `Cumulative sent` = prior cumulative + this run, `Accepted since last run`, `Replied`. This is what lets the next no-prompt run resume the right list.

### Step 7 — Report in chat
- Inbox pulled/skipped counts (already-processed vs out-of-scope, separately); newly-connected people (with @mentions posted) and new replies; new sends and carryover; and anything flagged (unclear persona/company/campaign, people who left, auth hiccups). Never repeat full original thread text — names, companies, gist only.

## Scheduling
Sends need Chrome and a human go-ahead, so any scheduled/automated run is **reconcile-only** (Steps 0–4, 6–7 — skips the 4.5 questions, never sends). On-demand runs do the sends. Wire via `/schedule` or launchd later if wanted.

## Notes
- Team-ready & standalone: this is a self-contained plugin. It bundles its own `linkedin` MCP server (`.mcp.json`) and permission deny-list (`.claude/settings.json`), and does not depend on `linkedin-crm-sync`, `convohub-sync`, or any other plugin. Identity is resolved per run (Notion + LinkedIn `get_my_profile`), so once a BDR installs the plugin it works for them against their own tracking page and login — no editing, no shared state.
- Per-BDR runtime prerequisites (not shippable in the plugin): Claude in Chrome logged into the BDR's own LinkedIn (send path only), and the BDR's own Notion + Salesforce MCP access. The read-only LinkedIn MCP and its deny-list come with the install.
