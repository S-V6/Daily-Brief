# End-of-Day Brief — Runbook

This file is the **instruction set** for the automated end-of-day brief. A
scheduled cloud Routine fires each weekday evening, opens a fresh session in
this repo's environment, and follows these steps exactly. Everything the run
needs is here — do not assume prior conversation context.

---

## CONFIG (edit these values to change behavior)

- **Owner:** Sean Montplaisir — `sean@v6advisors.com`
- **Timezone:** US Eastern (America/New_York)
- **Draft recipients (`To:`):** `sean@v6advisors.com`
  > Safety default is the owner only. To send the brief to colleagues,
  > add their addresses here (comma-separated). The Routine can only
  > *create a Gmail draft* — it never sends — so the owner reviews and
  > sends manually.
- **Output file:** `briefs/YYYY-MM-DD.md` (today's date, Eastern)
- **Commit target branch:** `main`
- **Gmail scope:** primary inbox, `newer_than:1d`, plus the `To respond` label.
- **Airtable knowledge base:** the **"Meeting Intelligence"** base
  (`app4jGLgUGWH1IzDO`) in workspace "My First Workspace". Used as a
  **read-only** relational reference to enrich the brief (deals, customers,
  projects, tracked action items, decisions, org relationships).

> ⚠️ **AIRTABLE IS READ-ONLY. NEVER MODIFY IT.** Only ever call the Airtable
> *read* tools: `list_bases`, `search_bases`, `list_tables_for_base`,
> `get_table_schema`, `list_records_for_table`, `search_records`. You must
> **never** call any Airtable tool that creates, updates, deletes, publishes,
> comments, or reverts (`create_*`, `update_*`, `delete_*`,
> `create_record_comment`, `publish_interface`, `revert_action`, etc.). The
> base is a knowledge source only — its contents inform the writing; nothing
> is ever written back.

---

## STEPS

### 0. Establish "today"
Use the current date in **US Eastern** as `TODAY` (format `YYYY-MM-DD`).
Compute `TOMORROW` as the next calendar day. All windows below are relative
to `TODAY` in Eastern time.

### 1. Calendar — today's meetings + tomorrow's preview
Use `mcp__Google_Calendar__search_events` to gather:
- Events occurring **today** (title, start–end time ET, attendees, conference link).
- Events scheduled for **tomorrow** (title + time) as a look-ahead.

Search is semantic, so run a few broad queries (e.g. `"meeting"`,
`"call"`, `"sync"`, plus recurring series names) and de-duplicate by event
`id`. Keep only events whose `start.dateTime` falls on `TODAY`/`TOMORROW`
Eastern. Sort chronologically. If none, say "No calendar events."

### 2. Fireflies — today's meeting summaries + action items
- `mcp__Fireflies__fireflies_get_transcripts` to list recent transcripts;
  keep those recorded **today** (Eastern).
- For each, `mcp__Fireflies__fireflies_get_summary` (or
  `fireflies_get_transcript`) to pull the overview, key points, and
  **action items**. Attribute action items to owners where the summary does.
- If no meetings were recorded today, say "No recorded meetings today."

### 3. Gmail — notable threads / follow-ups
- `mcp__Gmail__search_threads` for `in:inbox newer_than:1d` — surface
  threads that look like they need a reply or a decision (skip newsletters,
  automated notifications, and the `DoD Newsletters` label).
- Also pull anything under the `To respond` label
  (`labelId: Label_1`) regardless of age.
- Summarize each as one line: sender — subject — why it matters. Cap at ~10.

### 3.5 Meeting Intelligence — Airtable relational context (READ-ONLY)
Pull structured context from the **"Meeting Intelligence"** base
(`app4jGLgUGWH1IzDO`) to enrich the brief. **Read only — never write.**

Tables (IDs are stable; use `get_table_schema` at runtime for singleSelect
choice IDs before filtering on them):

| Table | ID | Use |
|-------|----|-----|
| Meetings | `tblbDK2735FOHIkCM` | The intelligence hub — match today's meetings to catalogued records: `Meeting Type`, `Meeting Summary`, `Pursuits Discussed`, `Projects Discussed`, linked `Action Items`, `Decisions`. |
| Action Items | `tblL93JO61XkX1825` | `Name`, `Status`, `Owner`, `Due Date`, `Scope`, `Pursuit`, `Project`, `Source Meeting`. |
| Pursuits | `tblj7iQsbUNwSlUv7` | `Name`, `Stage`, `Owner`, `V6 Involvement`, `Customer`, `Sector`, `Notes on Pursuit`. |
| Projects | `tbloway9lsITnegTG` | `Name`, `Status`, `Project Lead`, `Client`, `End Date`. |
| Organizations | `tblV12iboJuVRc64Z` | `Name`, `Consulting Relationship`, `Investment Relationship` (labels a company as client / portfolio / partner / target). |
| Customers | `tblC9W8cyPqOhYB9L` | `Name`, `Type`, `Sector` (DoD program offices / agencies). |
| Decisions | `tblfv3wGj7B974OJl` | `Name`, `Type`, `Date Decided`, `Context`, `Pursuit`, `Project`, `Source Meeting`. |

Suggested reads (all read-only):
1. **Recent meetings** — `list_records_for_table` on Meetings filtered to
   `Date` within the past few days (`isWithin` / `pastNumberOfDays`, Eastern).
   Match them by title to today's Fireflies meetings so each meeting can be
   annotated with its **Meeting Type**, linked **Pursuit(s)** and
   **Project(s)**, and its catalogued **Action Items / Decisions** (often
   cleaner than the raw transcript).
2. **Open & due action items** — Action Items filtered to `Status` ≠ done
   (resolve choice IDs via `get_table_schema`), especially those overdue or
   due soon. These are tracked items beyond just today's meetings.
3. **Deal / relationship context** — for Pursuits, Projects, Organizations,
   and Customers linked from the above, resolve their names and key fields
   (stage, relationship type, customer/agency) so the brief can say *which*
   deal/portfolio company/agency a discussion concerns.

Use this purely to **inform composition** — do not dump raw records. If the
Airtable base is unreachable in a given run, note it briefly and continue
with the transcript-derived content.

### 4. Compose the brief
Write markdown in this structure:

```
# End-of-Day Brief — {Weekday}, {Month D, YYYY}

## 📅 Today's Meetings
- HH:MM–HH:MM  Title — attendees (n)  [Deal/Project · Stage — from Airtable, if matched]

## 📝 Meeting Notes & Action Items
### {Meeting title}  ·  {Meeting Type} — {Pursuit/Project · Customer, if linked in Airtable}
- Summary: ...
- Action items:
  - [ ] owner — task  {(due date / status, if tracked in Airtable)}

## 📌 Open & Overdue Action Items (tracked in Meeting Intelligence)
- [ ] owner — task — due {date} · {Pursuit/Project} {(OVERDUE) if past due}
  (Open items from the Airtable Action Items table — overdue first, then
  due soonest. Cap at ~12; note if more remain.)

## 🔗 Deal & Portfolio Context
- {Pursuit} — Stage {stage}, Customer {customer/agency} — one line on why it
  surfaced today. Label companies by relationship (client / portfolio /
  partner / target) using the Organizations table.
- Recent relevant Decisions (from the Decisions table), if any.

## ✉️ Follow-ups from the Inbox
- Sender — Subject — why it matters

## 🔭 Tomorrow at a Glance
- HH:MM  Title

## ✅ Suggested EOD Actions
- (synthesized: overdue/near-due tracked action items + today's new action
  items + unreplied threads worth closing today)
```

Keep it tight and skimmable. Prefer bullets over prose. Fold the Airtable
context into the meeting entries and the two relational sections — do not
paste raw records or IDs. If a section has no content, write a short "Nothing
today" line rather than omitting the header. The two Airtable-backed sections
(📌 and 🔗) should be omitted gracefully (with a one-line note) only if the
base was unreachable this run.

### 5. Save to the repo
- Write the brief to `briefs/{TODAY}.md`.
- Commit on `main`: `git add briefs/{TODAY}.md && git commit -m "EOD brief {TODAY}"`.
- Push with `git push -u origin main` (retry with backoff on network error).

### 6. Create the Gmail draft
- `mcp__Gmail__create_draft` with:
  - `to`: the CONFIG **Draft recipients**
  - `subject`: `EOD Brief — {Weekday}, {Month D, YYYY}`
  - `body`: the plain-text brief
  - `htmlBody`: an HTML rendering of the same brief (optional but preferred)
- Do **not** send. Report the draft `id` so the owner can review and send.

### 7. Report
End the run with a 2–3 line status: what was gathered (counts), the committed
file path, and the draft id. If any source was unavailable in the headless
run (e.g. Gmail/Calendar auth missing), state which one and continue with the
rest rather than failing the whole brief.

---

## NOTES

- **Headless auth caveat:** interactively-authenticated apps (Gmail, Calendar,
  Airtable) can occasionally be missing in a fully headless cron run. Always
  degrade gracefully — produce the brief from whatever sources *are* available
  and note the gap in the status.
- **Connector cold-start retry:** in a fresh headless session, Gmail/Calendar/
  Airtable MCP connectors can still be finishing authentication when Step 1-3.5
  first call them, which can surface as an immediate tool denial even though
  the tool is allow-listed. Do not treat a first failure as "source
  unavailable" — wait briefly (e.g. do other steps first, or pause a few
  seconds) and retry each failed source once before falling back to the
  graceful-degradation note. Only report a source as unavailable after that
  retry also fails.
- **Airtable is strictly read-only.** Never create, update, delete, comment on,
  or publish anything in Airtable. If a task ever seems to require writing to
  Airtable, stop and skip it — this runbook only *reads* the Meeting
  Intelligence base.
- **Idempotency:** if `briefs/{TODAY}.md` already exists, overwrite it and
  amend rather than creating a duplicate.
- **No secrets in this repo.** All access comes from the environment's
  connected integrations, not committed credentials.
