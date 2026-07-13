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

### 4. Compose the brief
Write markdown in this structure:

```
# End-of-Day Brief — {Weekday}, {Month D, YYYY}

## 📅 Today's Meetings
- HH:MM–HH:MM  Title — attendees (n)

## 📝 Meeting Notes & Action Items
### {Meeting title}
- Summary: ...
- Action items:
  - [ ] owner — task

## ✉️ Follow-ups from the Inbox
- Sender — Subject — why it matters

## 🔭 Tomorrow at a Glance
- HH:MM  Title

## ✅ Suggested EOD Actions
- (synthesized: open action items + unreplied threads worth closing today)
```

Keep it tight and skimmable. Prefer bullets over prose. If a section has no
content, write a short "Nothing today" line rather than omitting the header.

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

- **Headless auth caveat:** interactively-authenticated apps (Gmail, Calendar)
  can occasionally be missing in a fully headless cron run. Always degrade
  gracefully — produce the brief from whatever sources *are* available and note
  the gap in the status.
- **Idempotency:** if `briefs/{TODAY}.md` already exists, overwrite it and
  amend rather than creating a duplicate.
- **No secrets in this repo.** All access comes from the environment's
  connected integrations, not committed credentials.
