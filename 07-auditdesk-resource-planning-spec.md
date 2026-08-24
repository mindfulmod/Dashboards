# AuditDesk — Resource Planning & Delivery Analytics Spec (v2.1)

All decisions locked 2026-08-23 via discovery interview (12 questions, 3 rounds). Extends `01-product-spec.md` and `06-auditdesk-team-mode-spec.md`; builds on team mode's merge model (field-level LWW, append-only unions, per-user replica files). Target: `AuditDesk.html`, same hard constraints (single file, Edge, `file://`, zero network calls).

## One-liner

Plan an audit by loading tasks with hour estimates spread across days per person, commit the plan as a baseline, check capacity fit at kickoff, have each assignee close their day with a sub-minute check-in, surface at-risk work while there's still time to act, and — after enough history — answer "is this person consistently missing, or are we planning too aggressively?" with estimate-calibration math instead of vibes.

## Design principles (from research + interview)

1. **Ceremony budget is the binding constraint.** The daily check-in must complete in under 60 seconds with everything prefilled, or the habit dies and every downstream analytic starves. (Industry consensus from standup/timesheet tooling.)
2. **Measure to calibrate planning, never to scoreboard people.** Individual velocity metrics get gamed (estimate padding, softened check-ins — Goodhart's law). The workable framing: each person's actual-vs-estimate ratio *compared to the team's ratio*. Lead-facing, plus each person sees their own history. Shared files mean this is a norm, not a secret — the spec's copy and placement enforce the norm.
3. **The plan is a commitment, drift is a signal.** No silent auto-reflow; the grid shows where reality diverges from plan, and rebalancing is an explicit, logged act.

## Locked decisions

| Area | Decision (2026-08-23) | Declined |
|---|---|---|
| Planning grain | Task = total estimated hours + start/due range; app auto-spreads evenly across workdays; any day cell editable (Float model) | Manual per-day grid only (ceremony); estimate+range with no per-day override (can't express uneven weeks) |
| Assignment | **Exactly one assignee per task** (existing `owner`); multi-person work is split into tasks/subtasks | Multi-assignee bookings (merge + attribution complexity); owner+helpers (helper hours invisible to capacity) |
| Capacity | Per-person default chargeable hours/day (suggest 6) + per-date exceptions (PTO, holiday, training) + shared holiday list; workweek Mon–Fri | Flat team number (vacation weeks lie); no capacity ceiling (fit check stays manual) |
| Daily check-in | Status (On track / At risk / Blocked) + hours spent today + **hours left** + optional comment | Status+comment only (weak analytics); full timesheet (second timesheet system, dies) |
| Daily ritual | **End-of-day "Wrap up my day"** batch dialog over all of today's allocated tasks, plus inline quick check-in on any task anytime | Per-task inline only (no ritual → patchy data); morning-standup style (day-late signal) |
| At-risk signals | (a) Projected overrun: hoursLeft > remaining planned hours before due; (b) self-flagged At risk; (c) Blocked > N working days (default 2, configurable) | Check-in silence (deferred to v2.2 — noisy until the habit is established) |
| Replanning | Plan static; drift displayed; **explicit Rebalance** action re-spreads remaining hours forward | Nightly auto-reflow (hides plan failure); manual-only (tedious) |
| Baseline | **Explicit "Commit plan"** per engagement snapshots estimates + allocations + due dates + owners; append-only | Continuous compare-to-current (rebalancing erases evidence); auto-baseline on first assignment (baselines rough drafts) |
| Plan scope | **Tasks only** carry hours. Deliverable prep = a task (optionally linked to the doc) | Planning hours on docs (double-counting); planning validation reviews (machinery ≫ signal) |
| On-time metric | Measured against **baseline due**; due-date revisions never erase a miss — revision count + days slipped are themselves drift metrics. Live views keep using effective due (revised ?? due) | On-time vs revised due (plan rescued on paper); reporting both rates (invites misreading) |
| Plan control | Post-commit: lead edits estimates/assignments/due dates; **assignee may re-spread their own remaining hours across their own future days** (totals and dates unchanged). All plan edits → timeline | Lead-only (interruption factory); anyone-edits-anything (baseline loses meaning) |
| Analytics reach | Person-vs-team comparison lives in a **lead-facing Insights screen**; every user sees **their own** accuracy history. No cross-person view outside Insights | Fully transparent scoreboard (gaming incentive); engagement-level only (abandons the core question) |

## Data model (schemaVersion 5)

### Task additions

```
task.plan = {
  estimateH: number,            // total estimated hours, 0.5 granularity
  start: "YYYY-MM-DD",          // planning range start (due/revisedDue = range end)
  alloc: { "YYYY-MM-DD": h }    // hours per workday; auto-spread on create, hand-editable
}
task.checkins = [               // append-only; merge = union by id (like reviews)
  { id, date: "YYYY-MM-DD", userTag,
    status: "on-track" | "at-risk" | "blocked",
    hoursToday: number, hoursLeft: number,
    comment?: string, ts: ISO }
]
```

- Merge (per spec 06): `estimateH`, `start` are scalars under `fieldTs`; `alloc` merges per-key like `custom` (`fieldTs['alloc.<date>']`); `checkins` union by `id`, sorted by `ts`.
- Derived (never stored): `actualH` = Σ `hoursToday`; `currentHoursLeft` = latest check-in's `hoursLeft`, else `estimateH − actualH`, else `estimateH`; `remainingPlannedH` = Σ `alloc[d]` for d ≥ today.
- Editing `estimateH` or dates does NOT touch `checkins`. Auto-spread runs on create and on explicit Rebalance only — never implicitly on edit.

### Capacity (shared settings, per-key LWW per spec 06)

```
settings.capacity = {
  defaults: { "<person name>": hoursPerDay },   // fallback 6 if unset
  exceptions: { "<person name>": { "YYYY-MM-DD": hours } },  // 0 = PTO/off
  holidays: ["YYYY-MM-DD", ...],                // 0 capacity for everyone
  blockedRiskDays: 2, riskTolerancePct: 0       // at-risk tuning
}
```

Workdays = Mon–Fri minus holidays. `capacityOn(person, date)` = exception ?? (holiday/weekend → 0) ?? default.

### Baseline (per engagement, append-only; merge = union by committedAt)

```
engagementMeta[eng].baselines = [
  { id, committedAt: ISO, committedBy: userTag,
    tasks: { "<taskId>": { estimateH, start, due, alloc: {...}, owner } } }
]
```

- **Commit plan** button on the engagement page snapshots every task of that engagement that has a `plan`. Confirm dialog: "This locks the kickoff plan as the measuring stick for this audit's analytics."
- Baselines are immutable once written. Re-commit is allowed (confirm: "The original kickoff baseline remains the reference; this re-commit is recorded as a replan event.") and appends.
- **Analytics always measure against `baselines[0]`** (kickoff). Later entries are counted as replan events, not new measuring sticks.

Migration: `ensureDefaults` backfills `plan: null`, `checkins: []`, capacity defaults; bump `meta.schemaVersion` to 5. Tasks without `plan` behave exactly as today (planning is opt-in per task).

## At-risk engine (computed, never stored)

A task with a `plan`, active status, is **At risk** when any of:

1. **Projected overrun** — `currentHoursLeft > remainingPlannedH × (1 + riskTolerancePct/100)`. Also fires when remainingPlannedH is 0 and hoursLeft > 0 (plan exhausted, work isn't).
2. **Self-flagged** — latest check-in status is `at-risk`.
3. **Blocked too long** — task status Blocked and the timeline shows the Blocked transition ≥ `blockedRiskDays` working days ago.

Severity order for badges: Blocked > At risk > On track. Overdue (existing rule) remains its own flag and stacks. Risk reasons are always shown with the badge ("needs 12h, 6h planned before due"), never a bare icon — the lead should see *why* without clicking.

## Views & flows

### Plan grid (extends the v1 workload calendar; nav key unchanged)

- **Person rows × workday columns**, filterable to one engagement or all; horizon = this week + N weeks (scrollable).
- Cell = Σ allocated hours vs capacity: neutral under 80%, amber 80–100%, **red > 100%** with the overage shown ("7.5 / 6"). Zero-capacity days (PTO/holiday) render dimmed; any allocation on them is automatically red.
- Cell expands to task chips (`ref · title · h`); chips open the task editor. Lead can drag hours between a task's own days (respecting plan-control rules).
- **Fit check panel** (shown prominently pre-commit): list of overallocated person-days · tasks with estimates but no allocation · tasks allocated past their due date · people with zero load during the engagement window. "Commit plan" lives here and warns if fit problems remain (commit is allowed anyway — the lead decides).

### My day (dashboard panel, first position for anyone with allocations today)

- Today's allocated tasks: ref, title, planned hours today, risk badge, quick check-in button.
- **Wrap up my day** button opens one dialog listing all of today's allocated tasks as rows: status segmented control (defaults to previous status), hoursToday (prefilled with today's planned hours), hoursLeft (prefilled with `currentHoursLeft − hoursToday`, floor 0), optional comment (required only when status = Blocked: "what's blocking, who can unblock"). One save writes one check-in per touched row. Untouched rows write nothing. Target: under 60 seconds.
- Inline quick check-in (same fields, single task) available from any task card/editor at any time — flipping to Blocked mid-morning shouldn't wait for evening.
- Check-ins write to the assignee's own tasks only. Each check-in also appends a timeline entry.

### Drift & rebalance

- On any planned task where past-day allocations ≠ reported hours or `currentHoursLeft ≠ remainingPlannedH`, the editor and grid show a drift line: "Plan: 6h remaining · Latest estimate: 10h remaining (+4h)".
- **Rebalance** button: re-spreads `currentHoursLeft` evenly across the task's remaining workdays through its effective due date, previewing the before/after row before applying. Assignee: own tasks, own future days, cannot change `estimateH`, dates, or owner. Lead: may also extend dates / change estimate in the same dialog (fields visible but marked as plan changes). Every rebalance → timeline entry.
- Dashboard (lead scope): "At risk" panel listing risk-flagged tasks with reasons, days blocked, and assignee — replaces guessing with a queue.

### Insights (lead-facing screen, opened from Engagements or Settings-adjacent nav; every user additionally gets a "My accuracy" panel showing only their own numbers)

Per engagement (vs `baselines[0]`):
- **Planned vs actual hours** (total and per person), on-time rate (complete by baseline due), due-date revision count and total days slipped, replan events (extra baseline commits), blocked-day totals.
- **Aggressiveness read**: team overrun ratio = Σ actual ÷ Σ baseline estimate. Guidance copy, not verdict: ratio ≤ 1.1 "plan held"; 1.1–1.3 "plan ran hot"; > 1.3 "plan was aggressive — estimates or capacity assumptions need recalibrating."

Per person (Insights only, min sample gate):
- Overrun ratio (their actual ÷ their baseline estimates) side-by-side with the team ratio, on-time rate vs team on-time rate, blocked frequency.
- **Sample gate: no per-person ratio renders with fewer than 5 completed baselined tasks** — shows "not enough history (n=3)" instead. Small-n numbers are noise and read as verdicts.
- Interpretation copy baked into the screen: "If the team ratio is high, recalibrate planning before reading individual numbers — a person can only be evaluated against the plan quality they were given."

Exports: engagement analytics summary block is appended to the print report when opened from Insights; weekly status text gains an optional "At risk" section (on by default for lead scope).

## Interaction with team mode (spec 06)

- `checkins` and `baselines` are append-only → conflict-free unions. `plan.alloc` per-key LWW. Capacity settings shared per-key LWW.
- Wrap-up writes only to the current user's own tasks → in practice single-writer per check-in stream.
- Presence/soft-locks apply to the task editor as normal; the wrap-up dialog bypasses soft-lock (it writes append-only check-ins, not fields).
- Scoping (spec 06 §Visibility): Plan grid respects My work ⇄ All toggle; Insights per-person view is lead-role gated in UI (norm, not secret — same honesty rule as spec 06). Private tasks: may carry a `plan` for the owner's own capacity math, appear in their own grid row only, are excluded from team grid totals shown to others (they're not in the shared file anyway), and never enter analytics.

## Implementation phases (each independently shippable; feed Copilot one at a time)

1. **Phase A — planning core:** schemaVersion 5 migration, `task.plan`, auto-spread + per-day editor in the task editor, capacity settings UI (defaults, exceptions, holidays).
2. **Phase B — plan grid + fit + baseline:** person × day grid with capacity coloring, fit check panel, Commit plan snapshot.
3. **Phase C — check-ins:** wrap-up dialog, inline quick check-in, My day panel, timeline integration.
4. **Phase D — risk & drift:** at-risk engine, badges with reasons, drift display, Rebalance action with role rules, lead At-risk dashboard panel.
5. **Phase E — Insights:** engagement analytics, per-person view with sample gate, My accuracy panel, export additions.

## Invariants (do not violate)

- Check-ins and baselines are append-only; nothing edits or deletes an entry.
- `baselines[0]` is the sole reference for on-time and overrun analytics; re-commits never replace it.
- No per-person analytic renders below n = 5 completed baselined tasks.
- At-risk is derived at render time, never persisted.
- Auto-spread never runs implicitly — only on task-plan creation and explicit Rebalance.
- Assignee rebalance cannot change estimate, dates, or owner.
- Hours exist on tasks only; one assignee per task.
- The wrap-up dialog must be completable with keyboard only, prefilled, in under a minute; a Blocked status is the only thing that makes any field mandatory.
- Zero network calls; all v1/spec-06 invariants stand.

## Open on purpose (decisions, not oversights)

- Check-in-silence risk signal — revisit after the wrap-up habit is established (v2.2).
- Default chargeable hours (6 suggested) — confirm against the team's real meeting load before rollout.
- Whether Insights' engagement summary should feed the report gate — parked until Insights exists.
