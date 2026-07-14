# Daily-Brief

Automated **end-of-day brief** for Sean Montplaisir / V6 Advisors. Each
weekday evening a scheduled cloud Routine gathers the day from connected
apps, compiles a brief, saves it to this repo, and drafts it in Gmail for
review.

## How it works

1. A cloud **Routine** (scheduled trigger) fires **weekdays at 6:00 PM ET**.
2. It opens a fresh session in this repo's environment and follows
   [`brief-runbook.md`](./brief-runbook.md).
3. The run pulls from:
   - **Google Calendar** — today's meetings + tomorrow's preview
   - **Fireflies** — summaries & action items from today's recorded meetings
   - **Gmail** — follow-up threads and `To respond` items
   - **Airtable — "Meeting Intelligence" base (read-only)** — relational
     context: which deal/pursuit, customer/agency, project, and portfolio
     relationship a discussion concerns, plus tracked open/overdue action
     items and prior decisions. The routine only *reads* this base and never
     modifies it (enforced by a `deny` list in `.claude/settings.json`).
4. It writes the brief to [`briefs/`](./briefs/) (one file per day) and
   creates a **Gmail draft** addressed per the runbook's CONFIG. It never
   auto-sends — Sean reviews and sends.

## Configuring

All behavior is controlled by the **CONFIG** block at the top of
[`brief-runbook.md`](./brief-runbook.md) — recipients, timezone, Gmail
scope, output path. Edit that file (no code changes needed) and the next run
picks it up.

## Layout

```
brief-runbook.md   # instructions the Routine executes each run
briefs/            # dated output, e.g. briefs/2026-07-13.md
```
