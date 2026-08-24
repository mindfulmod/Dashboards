# AuditDesk v2 — Consolidated Build Spec (Team Mode + Resource Planning)

**Status:** THE authoritative implementation spec. v2.7, locked 2026-08-24. **Paper phase CLOSED with this revision.**
**Change policy (locked):** this spec changes only for (a) a failed executable acceptance test, (b) measured T0/T2 evidence, or (c) a reproducible data-loss, privacy, or divergence defect. General paper critiques are over; the next source of truth is executable evidence. Items listed in §20 are decided; items in §21 are implementation latitude needing no review.
**Revision note:** v2.7 incorporates the spec-15 closure memo in full: workspace-scoped storage + queue barrier, persisted write attempts, graph-wide conflict derivation, contaminated-event rule, member-lifecycle lineage, authored derived work, shared comments + local deliverable notes, the complete legacy-migration contract (revisedDue, approvals, reviews, legacy actor, ids), engagement-meta/milestone semantics, final actual-hours rule, custom-field/doc-type stable ids, dueTime/subtask dates, clock-skew editability, domain-separated hashing, restartable import, avatar bounds.
**Supersedes entirely:** specs 06–09 and 11–15 (`specs/history/`; never implement from them). **Self-contained: implement from this file alone, one phase at a time (§16).**

## 1. Hard constraints (unchanged from v1)

- One self-contained `.html` file: all CSS/JS inline, no CDN, no fetch, **zero network calls** (user-initiated navigation of a pasted link opens in the browser; the app itself never fetches). Edge from `file://`, locked-down machine, no installs.
- OneDrive/SharePoint sync is the only network transport; the app only reads/writes local files.
- Small internal team; **one work device per person, one active session per person**; no mobile/multi-device.
- Implementer: M365 Copilot, phase by phase; §17/§18 are the review checklist. All v1 behavior preserved per the migration contract (§6).

## 2. Product summary

Auditors collaborate through a shared OneDrive-synced folder: one replica file per member, deterministic field-level merge, advisory presence, engagement-scoped dashboards, local-only private tasks and personal notes. Planning: per-day hour allocations against per-person capacity, committed kickoff baselines, sub-minute daily check-ins, an at-risk engine that never fabricates signals, lead-facing estimate-calibration analytics.

## 3. Architecture: hybrid state/event model

- **State-based where state matters:** one full-replica snapshot per member; scalar fields carry version envelopes (§3.1), deterministic LWW.
- **Event-sourced where history matters:** append-only typed arrays with entry ids (`checkins`, `statusEvents`, `dueEvents`, `approvalEvents`, `consequentialFieldEvents`, `archiveEvents`, `memberLifecycleEvents`, `commentEvents`, `capacityEvents`, `reservationEvents`, `reviews`, `baselines`, `conflicts`, `conflictResolutions`, `acceptanceEvents`). Unions by id; entries immutable; corrections supersede (`supersedesId`).
- **Uniform event base:** every entry carries `{ id, type, ts, by }`; **consequential-field events additionally carry `seq`** so losing-branch ordering stays reconstructible after losing envelopes leave `fieldTs`. Merge ordering uses `(ts asc, by asc, id asc)` only; domain aliases are display.
- **Contaminated-event rule (order-independent):** the same event `id` with byte-different canonical bodies marks that id **contaminated**: **all** variants are excluded from canonical lineage, raw variants are preserved in a quarantine/data-repair view, the affected entity enters a visible repair state, and remaining data resolves via lineage-incomplete + deterministic LWW. File order, scan order, "local first," or arrival order never select a body.
- **Deterministic identifiers (everywhere):** never raw delimiter concatenation. All deterministic ids use domain-separated canonical tuples — `hash(canonicalize(["auditdesk.<domain>.v1", …inputs]))` — SHA-256 over UTF-8 bytes of the canonical serialization, output lowercase full hex. Domains: legacy genesis, derived-work seeds, engagement migration ids, field conflict ids, check-in conflict ids, review/checklist/custom-field/milestone migration ids, import fingerprints.

### 3.1 Version envelope (all scalar fields, uniform shape)

```
fieldTs[field] = { ts, by: personId, seq, opId, previousOpId }
```

- `seq`: persisted monotonic counter per member (§3.4). `previousOpId` = version visible when the edit began (null only for genesis).
- **Total order:** `(ts, by, seq, opId)` ascending; greatest wins.
- **Merge NEVER rewrites or clamps received timestamps.** Far-future `ts` (>10 min) → local-only clock diagnostic.
- **Clock-skew editability:** when writing a **local** envelope over a visible predecessor, the writer stamps `ts = max(wallClock, predecessorTs)` (and increments `seq`), so a deliberate local edit always lands strictly after the far-future predecessor in total order — received envelopes stay untouched, but no field can be frozen by remote skew. The logical advance persists in writer metadata; the clock diagnostic still shows. This is the complete HLC-lite rule; no other clock logic exists.
- **Explicit paths:** `fieldTs['plan.estimateH']`, `['plan.start']`, `['plan.alloc.<date>']`, `['custom.<definitionId>']` (§8 — definition ids, never labels); `engChecks` values use this envelope. Removal writes an explicit neutral value with a new envelope; **missing keys carry no removal meaning**.

### 3.2 Lineage classification (ancestry-based, guarded)

Consequential fields (owner/assignee, status, due, rating, approval/validation verdicts, `estimateH`, member lifecycle) have typed event histories sharing their envelopes' `opId`/`previousOpId`/`seq`. **Merge unions typed events first**, builds the ancestry index, then classifies: identical → ignore; current in incoming's ancestry → sequential, apply; incoming in current's ancestry → stale, ignore; divergent from a common ancestor → branch; missing/corrupt history → deterministic LWW + local lineage-incomplete diagnostic, never a fabricated conflict. **Guards:** visited set + max traversal bound; cycles, contaminated ids, excess depth, or missing predecessors end as lineage-incomplete; never hang the state queue. Non-consequential fields: direct-envelope LWW, no conflicts.

### 3.3 Collaborative collections & profile/lifecycle separation

List-like shared values merge **per item**; removal sets `active:false`, never deletes keys: engagement membership, person registry, checklist templates, custom-field definitions, document types, **milestones**. Within a registry item, **profile fields (`name`, `color`, `avatar`, `activeRefPrefix`) merge per-field with normal envelopes**; **member lifecycle is a separate consequential lineage** (`memberLifecycleEvents`: `member.deactivated`/`member.reactivated`, reactivation naming the active deactivation op, admin-gated) — `active`/`deactivatedAt`/`deactivatedBy` are **derived** from accepted lifecycle history, and ordinary profile saves never stamp or transmit lifecycle state (a stale profile edit can never resurrect a deactivated member). Checklist items get stable ids; `engChecks` rekeys to item ids.

### 3.4 Persisted writer state

`metadata[workspaceId, writerPersonId]`: `{ lastWriterTs, nextSeq }`. Next `seq` reserved in the same transaction as the mutation; `lastWriterTs` (including logical advances from §3.1) persists before success; restart continues; failed transactions advance nothing; gaps acceptable, reuse never.

## 4. Team folder, workspace & bootstrap

```
AuditDesk Team/
├── workspace.json
├── replicas/auditdesk-<personId>.json     (single writer each)
└── presence/presence-<personId>.json      (single writer each)
```

```js
const replicasDir = await teamDir.getDirectoryHandle('replicas', {create:true});
const presenceDir = await teamDir.getDirectoryHandle('presence', {create:true});
```

`workspace.json` (written once at creation; `schemaVersion` changes only via explicit single-machine migration):

```json
{ "workspaceId": "uuid", "name": "Internal Audit", "schemaVersion": 5,
  "minimumAppVersion": "2.7", "timezone": "America/Toronto",
  "workspaceAdminIds": ["<personId>"], "createdAt": "ISO", "createdBy": "<personId>" }
```

- **Join opens `workspace.json` `{create:false}`**; Create is distinct. Read-only (banner) on: foreign `workspaceId`, newer schema, or `APP_VERSION` < minimum. `timezone` governs dates/end-of-day once a workspace exists; **solo mode uses device timezone**. Stored timestamps UTC. Copy: **"Saved to team folder"**, never "Synced".

### Bootstrap (contamination-safe; Create ≠ Join)

Working state keys by `workspaceId` (`"solo"` otherwise); in-memory datasets are never reused across switches; a foreign-workspace check precedes every write/projection. **Create:** two explicit options with a counts preview (never inferred): *Use my current AuditDesk data* (creator's machine performs the one canonical migration per §6) or *Create empty*; solo state retained as recovery snapshot either way. **Join:** team data authoritative — read workspace + replicas **before** profile creation; snapshot solo state; load merged team projection; keep only local preferences + private tasks; **never auto-publish prior solo records** (first replica = merged team state + new profile); later solo import only via the explicit **Import** command (§6). **Team-workspace guards:** "Load sample data" and "Clear all" unavailable in team workspaces.

### Storage scoping & workspace switching (queue barrier)

All durable writer data is compound-scoped: `state[workspaceId]`, `outbox[workspaceId, writerPersonId, batchSequence]`, `metadata[workspaceId, writerPersonId, name]`, `handles[workspaceId, handleType]`; `local` splits into global (theme/visual) vs workspace-scoped (personId, scope, presentation role, UI state). Every outbox entry and file attempt carries `workspaceId` + `writerPersonId`; **the file worker rejects any batch whose workspace, writer, or handle mismatches its active context.** Switching workspaces is a queue barrier: stop scheduling old-context work → let an in-flight attempt finish only against its captured old handle → discard old in-memory state → load new state + writer metadata → only then enable queue and UI.

### Replica metadata & revisions

Replicas carry `{ replicaRevision, savedAt, writerPersonId, schemaVersion }`. **A revision is reserved once per write attempt** (persisted before file I/O); failed/abandoned attempts leave permanent gaps, never reused; only read-back verification advances `lastVerifiedRevision`. Peer scan skips a file only when both file metadata and parsed revision are unchanged since the last merged revision (mtime = optimization only). A lower-than-observed peer revision → stale-replica warning; state never rolls backward.

**Degraded mode** (FSA blocked): **Manual exchange mode** — persistent banner, explicit export/import, no presence, no background merge; merge logic identical.

## 5. Identity: one person registry, one ID domain

```
state.people = { "<personId>": { id, kind: "member"|"external",
    name, color, avatar,                       // profile: per-field envelopes (§3.3)
    userTag, activeRefPrefix, refPrefixHistory,  // members only
    /* lifecycle derived from memberLifecycleEvents */ } }
```

- **Member `person.id` IS the memberId** (names replica/presence files). External people (auditees) never receive capacity, check-ins, workload rows, presence, or permissions. One ID domain: `task.ownerId`, `finding.ownerId`, `doc.preparerId`/`reviewerId`, `engagement.membership`/`leadId`, `baseline.tasks[].ownerId`, `checkin.by`, capacity/reservation `personId`. Task owner, doc preparer/reviewer, engagement members/lead must be members; finding owner may be either. Renames change display only.
- **Avatars are bounded:** either a built-in token (`builtin:fox` — assets embedded once in the HTML, replicas store only the token) or a normalized thumbnail with an explicit MIME allowlist, fixed max dimensions, an explicit encoded-size limit, and stripped metadata; oversized/invalid avatars are rejected before the profile commits; avatar bytes count in replica-size diagnostics.
- **Ref prefixes:** refs `F-MA-001` from `activeRefPrefix` (2–4 uppercase); a prefix is immutable history once used (`refPrefixHistory`); **every historical prefix — including deactivated members' — is permanently reserved** (validation scans all active + historical prefixes, all refs, active and inactive profiles). Onboarding reads team data first, blocks known collisions, then mints identity. **Honest guarantee:** known collisions blocked before record creation; a **delayed** collision (eventual consistency) may create ambiguous refs but is detected and contained — affected members cannot create records until one changes prefix; UUIDs authoritative; ambiguous refs require explicit selection. Legacy refs valid: `/^([FTD])-(?:([A-Z]{2,4})-)?(\d+)$/`.
- **Lifecycle:** deactivation/reactivation via `memberLifecycleEvents` (§3.3), admin-gated. Deactivated: history/attribution kept, no new assignments/capacity, presence ignored, replica readable, tasks → **reassignment queue**. **Startup identity confirmation:** team mutations, presence, and replica writes stay disabled until the initial team-folder read confirms the local personId active (never from cached state; unreadable folder → explicit offline/read-only). Deactivated local identity: read-only; private tasks readable/exportable; banner. Reactivation restores the same personId.
- **Legacy person migration:** normalize (trim/collapse/NFKC/case-fold) for matching only; one match → link; none → unresolved entry (original text retained); multiple → user chooses. Unknown historical actors (legacy timelines/reviews with no trustworthy author) attribute to the reserved **`system:legacy-import`** actor — "Actor unavailable in legacy data"; not assignable, no capacity/presence, excluded from person analytics; **never falsely attributed to the workspace creator.** Unresolved ownership fails Fit Check, never silently assigned.
- **Engagements** (entities; reverses v1 free-text lock, signed off 2026-08-24): `{ id, name, membership, leadId, archivedAt?, fieldTs }`; records store `engagementId`. Migration ids: domain tuple `auditdesk.engmigration.v1` over the normalized name (full SHA-256); normalized key stored; key mismatch → stop, manual resolution. Creation: UUID ids; normalized-name duplicate warns and offers the existing entity. UX unchanged.

## 6. Storage, migration & import

```
auditdesk-db v1 — stores (compound-scoped per §4):
  state | local (global + workspace-scoped) | handles | outbox | metadata
localStorage: only { lastWorkspaceId, themeBootstrap }.
```

**localStorage→IndexedDB migration:** restartable, idempotent; validate → one transaction → read back + verify → old value kept as recovery until one verified save → marker.

### Legacy migration contract (complete; creator-only for shared data)

Only the workspace creator (Create → *Use my current data*) migrates shared data; joiners never mint genesis for team records. Every mapping is **persisted before publication; crash + rerun = byte-equivalent results.**

- **Genesis:** each consequential field with a value but no lineage gets one deterministic genesis event + envelope: id = domain tuple (`auditdesk.genesis.v1`, workspaceId, entityId, field); `previousOpId: null`, `seq: 0`, `ts` = workspace `createdAt`, `by` = creator. One genesis for the effective value beats invented transitions. Migration markers record completion per workspace and schema step. First real edit names the genesis op.
- **Finding due dates:** canonical `due` = `revisedDue || due`; original due preserved; deterministic legacy due history marked **`legacy:true`** (not pretended to occur at workspace creation); pre-baseline legacy revisions never inflate post-baseline analytics.
- **Approvals & reviews:** `approvedBy`/`approvedAt` → typed approval history; finding validation rounds, task review rounds — dates, verdicts, dispositions, levels, outcomes, notes — become uniform-base events with deterministic ids; day-only legacy dates convert by one specified workspace-time rule; person names via legacy-person resolution (unknown → `system:legacy-import`).
- **Identity & structure:** deterministic/persisted migration for `createdAt`/`createdBy`, generic timeline entries, missing/duplicate record ids, review/event ids, custom-field definition + value ids, milestone ids, checklist ids + duplicate labels (explicit mapping, never silent collapse).

### Explicit Import (post-join; restartable, collision-safe)

Per imported source record: **Skip** / **Link to existing** (confirmed) / **Import as copy** (new workspace UUID). The source-fingerprint → workspace-id mapping persists **before** genesis or publication; repeat imports are idempotent via the mapping; changed source content = a new update/import decision, never a second genesis body under the same id. A source id colliding with a different existing entity never enters normal merge until rekeyed or explicitly linked. Import genesis is authored by the importing member (same rules).

IndexedDB failure → blocking persistence banner. Diagnostics panel: state size, last verified team write, pending outbox count, replica-size warning (incl. avatars).

## 7. State transaction pipeline, shared projection & outbox

One serialized `stateQueue` for every state-changing operation (mutation, composer commit, peer merge, migration, archive/restore, conflict resolution, derived-work reconciliation, outbox reconciliation): load latest state → apply → produce envelopes **and typed events atomically** → commit IndexedDB → update in-memory view → compute projection; enqueue batch only if its hash changed → render.

### Shared projection (privacy boundary)

`completeLocalState` (IndexedDB) vs `sharedProjection` (the only thing files ever see), **built positively from allowlisted data**: shared findings/tasks/docs + typed histories (incl. shared `commentEvents`), archive events, people/engagements, shared settings, capacity/reservations (hours only), baselines, conflicts/resolutions/acceptances. **Excluded:** private tasks + all child data; **deliverable local notes (§8)**; local-only links; personal settings; local diagnostics; local UI state; private-only op ids. **Private-only mutations create no team write and no outbox entry.** Shared→private = one atomic transaction (archive shared + create private; batch carries only the archive op). **Fail-closed check** before every write: `private:true` records, private id namespaces, local-note fields, local settings/diagnostics → block, privacy banner, never silent retry.

### Canonical serialization (one canonicalizer everywhere)

Recursively sorted keys; schema-normalized omitted-optional vs `null`; stable ordering for non-semantic arrays; append-only arrays by `(ts, by, id)`; order preserved only where position is data; consistent number/date forms; reject non-finite numbers and invalid timestamps before hashing. **Content-hash boundary excludes volatile transport metadata** (`replicaRevision`, `savedAt`, diagnostics, scan metadata) — identical domain content always hashes identically; no revision ping-pong.

### Outbox & write attempts (state-snapshot semantics)

Batches: `{ writeBatchId, workspaceId, writerPersonId, batchSequence, targetContentHash, createdAt, attempts }`. **Before file I/O, a write-attempt record persists** — `{ writeAttemptId, workspaceId, writerPersonId, targetBatchSequence, targetContentHash, reservedReplicaRevision, status, createdAt, attempts }` — with the committed projection and max included batch sequence captured atomically through the state queue. File queue: canonicalize → hash → write one complete replica at the reserved revision → read back, parse, validate, recompute hash, confirm hash **and** revision → record `lastVerifiedRevision` → **clear every batch with `batchSequence` ≤ the captured target** (a verified snapshot causally subsumes earlier pending snapshots of the same single writer; a batch enqueued during file I/O stays pending unless captured). Restart reconciliation: matching workspace/writer/revision/hash → verify + clear through target; mismatch/absent → keep batches, reserve newer revision; foreign workspace/writer → never write through the current handle. Mismatched hash at expected revision → pending → retry once → degraded banner.

## 8. Data model (schemaVersion 5)

### All records (findings, tasks, docs)

`createdAt`/`createdBy`, derived `updatedAt`, `fieldTs`, typed arrays. Envelopes stamped **only for changed fields**.

- **Canonical event lineage:** consequential mutations write typed events (same `opId`/`previousOpId`/`seq`, plus `field`, `from`, `to`) atomically with envelopes. Analytics walk only winning ancestry. `consequentialFieldEvents` covers rating, `plan.estimateH`, owner transfers. `completedAt` = latest canonical transition into Complete after the latest out, existing only while effective status is Complete.
- **Archive dominance:** an unmatched active archive keeps the record archived regardless of ordinary field timestamps; only an explicit restore naming the `archiveOpId` reactivates. Archive-vs-edit is **informational** ("An edit arrived while this record was archived"), never a formal conflict; the edit survives in history and shows on restore. Archive view for search; analytics keep archived baselined records. **No physical purge in v2.**
- **Comments vs notes vs description:** shared collaboration uses append-only **`commentEvents`** (`{id, type:'record.comment', entityId, body, ts, by, supersedesId?}`): additions union; corrections supersede; deletion (if offered) is a typed redaction event; private-task comments never enter the projection; only `http:`/`https:` URLs render as links, escaped, `target="_blank" rel="noopener noreferrer"`; navigation is user-initiated. Long-form `description` stays scalar LWW, visually distinct from comments. **Deliverable "My notes" (`doc.notes` — UI-promised "never leaves the app") moves to a local sidecar keyed `[workspaceId, recordId]`:** never in the projection, a replica, presence, comments/activity, or ordinary team export; included only in an explicitly requested personal backup with a privacy warning; peer merges cannot overwrite it.
- **Custom fields & doc types by stable id:** definitions are per-item collections with stable ids; values live at `record.custom[definitionId]` with `fieldTs['custom.<definitionId>']` — labels are display, renames never orphan values, labels never appear in paths. Docs store `documentTypeId`; removed/renamed types become inactive, historical records stay readable. Migration assigns ids and rekeys; duplicate labels require explicit mapping.
- **Reviews:** legacy rounds gain deterministic ids + uniform base (round = display); scalar review fields in LWW gain envelopes where applicable. Sign-off via `approvalEvents`; `approvedBy/At` derived.

### Task planning & scheduling

```
task.plan = { estimateH, start, alloc: {date: hours} }
task.due = "YYYY-MM-DD"|"";  task.dueTime = "HH:mm"|""      // dueTime: non-consequential scalar envelope
task.checkins = [{ id, type:'checkin', workDate, by, deliverySignal, hoursTodayTotal, hoursLeft, supersedesId?, comment?, ts }]
```

- **dueTime:** interpreted in workspace timezone (device tz solo); empty `due` forces empty `dueTime`; due+dueTime edits in one UI action = one atomic transaction; sorting uses workspace date/time, never per-device timezones.
- **Subtasks:** per-item id, LWW unit, `deletedAt` removal; blank subtask date inherits the task due dynamically; a local save rejects an explicit subtask date after the task due; **a merge producing an invalid combination preserves both received values and flags the task for correction — never clamps or deletes**; due-date edits preview newly invalid subtask dates.
- Planning opt-in; 0.5h granularity; one member assignee; all tasks plannable (incl. derived work — reviewer load is real capacity). `hoursTodayTotal` = cumulative daily total; later same-day check-ins supersede; effective = deterministic latest **after full union** (never scan order); noon 2h + evening 6h ⇒ 6h; branched corrections → deterministic latest + check-in conflict. `deliverySignal` independent of lifecycle; `blocked` offers status change, comment mandatory only for it. No `revisedDue` — `dueEvents`. Derived: `actualH`, `currentHoursLeft`, `futurePlannedH` (strictly after latest check-in `workDate` + flagged unconsumed same-day). **Complete actuals:** latest effective check-in `hoursLeft === 0` at completion, else Incomplete unless typed lead acceptance.

### Engagement metadata & milestones (team semantics)

`engagementMeta[engId]`: progress, attention, health trend, `budgetHours` use normal scalar envelopes. **Milestones are a per-item collection** with stable ids: fields merge per-field; deletion = `active:false`; `task.milestoneId` references the stable id (inactive milestones remain displayable historically); concurrent additions survive; local validation never deletes remote values; composer milestone operations (add/complete/reschedule) run through the same state queue and envelope rules as the editor.

### Actual hours — one system only

Actuals come **exclusively** from effective check-ins. `task.spentHours`, `engagementMeta.spentHours`, milestone `spentHours` become **legacy read-only** (clearly labeled; hidden where they'd mislead); manual spent-hours editing is removed when P1 ships; milestone actuals may derive from check-ins of tasks linked via `milestoneId`. Manual percent (`pct`) remains visual progress only — excluded from capacity, risk, actuals, calibration, and completion analytics. Engagement/milestone planned-vs-actual uses baseline/plan/check-in data only.

### Capacity & reservations

`settings.capacity.defaults{<personId>}` (fallback 6), `blockedRiskDays` (2), `riskTolerancePct` (0) at `settingsTs['capacity.…']`; `capacityEvents`/`reservationEvents` with explicit `clear`. Workdays Mon–Fri minus holidays (workspace tz; device solo).

### Baselines

`engagementMeta[engId].baselines` (`{id, type:'baseline', kind:"kickoff"|"replan", parentBaselineId?, ts, by, tasks:{taskId:{estimateH, start, due, alloc, ownerId}}, fitExceptions}`) + `kickoffBaselineId`. Immutable; competing kickoffs = the one blocking conflict (analytics only, lead resolves).

### Derived work (authored, atomic — not independently constructed)

**The machine authoring the source workflow transition creates or reopens the derived task in the same state transaction**: mints the ref in the author's own prefix namespace, writes normal envelopes/events, and publishes source change + derived task in one snapshot. **Peers adopt the authored task; they never construct it from source state.** Workflow-cycle keys are explicit (reopen deliverable-review vs new validation round is stated in the triggering event, never inferred from array lengths per replica). `syncDerivedWork` is demoted to an **idempotent reconciliation tool** for legacy/missing states: automatic repair requires a complete immutable `derivedWorkSeed` (stable derived id + authored ref + initial values) carried by the triggering event; without a seed it surfaces a repair action instead of guessing. Concurrent source branches must not leave two active tasks for one accepted cycle (reconciliation keys on the accepted canonical source lineage).

## 9. Merge rules

Triggers: load, focus, after save, **Sync now**, 60s interval while visible. Peer reads per the revision gate; local mutations debounce into one verified write.

1. Verify workspace compatibility → else read-only. Foreign-workspace check precedes any write.
2. Parse replicas; corrupt files quarantined, named, non-blocking.
3. **Union all typed events first** (`(ts, by, id)`; contaminated-id rule §3; incoming array order never decides anything).
4. Records: unknown id → adopt; known → per-field ancestry classification + total order. Received timestamps never rewritten.
5. Archive dominance; maps/collections per-key/per-item; explicit zeros/clears are values.
6. **Conflict derivation runs on the complete post-union event graph** per `entityId + field` (§9.1) — never on scan order.
7. Post-merge (in the state queue): recompute derived state, derived-work reconciliation, commit, render; change toast only on change.

### 9.1 Conflicts: graph-wide pairwise derivation + resolutions

After union, per `entityId + field`: validate ids/predecessor links → identify **maximal branch heads** → evaluate **every unordered pair of heads** → skip pairs where one head is the other's ancestor → `baseOpId` = nearest common ancestor → pair winner by the same total order as LWW → conflict id = domain-tuple hash of the canonical pair. Byte-equivalent on every machine in every arrival order; repeated merges add nothing; conflicts merge by id; LWW keeps work unblocked.

```
{ id, type:'conflict', ts: max(branch ts), by: winning author,
  entityId, recordRef, field, baseOpId, branchOpIds:[two, sorted], winnerOpId, alternateOpId }
```

**Check-in subtype:** `kind:'checkin-correction'`, id from (taskId, workDate, personId, sorted branch ids), derived post-union. **Grouping & resolution:** UI groups pairs by `entityId + field + baseOpId`; one group-resolution action appends resolution events for every currently unresolved pair; the group is resolved when every known pair has an effective resolution; a later new branch creates a new unresolved pair without touching history. Resolutions: `{id, type:'conflict.resolution', conflictId, resolutionOpId, ts, by}`; restoring an alternate = normal new field operation (typed event, lifecycle validation) based on the current winner. No `detectedAt`, no mutation, history never deleted; diagnostics local-only.

## 10. Save path & write safety

Saves write inside `replicas/` via the persisted, workspace-scoped handle — **no picker on save, ever**; topbar shows the real target. Startup: reload handle; `queryPermission`; `'prompt'` → slim banner + first-gesture `requestPermission()`; `NotFoundError` → clear handle, explicit re-pick. Team writes wait for identity confirmation (§5). State queue + projection upstream; file queue + attempt records + read-back verify downstream. **Single-session guard:** `navigator.locks.request('auditdesk-<workspaceId>', {ifAvailable})` → read-only "one tab at a time" (fallback BroadcastChannel). **Loud fallback:** forced downloads → persistent red banner. **Solo mode ships first (T1):** persisted `soloFile` handle, `id`/`startIn`, same banners — fixes the Downloads drift immediately.

## 11. Presence & soft locks (advisory only)

`presence-<personId>.json` `{personId, name, recordId, recordRef, recordType, ts}`; write on **shared-record** editor open, 30s heartbeat, clear on close/`beforeunload`; read every 10s. **Never written for private records.** Stale >3 min, deactivated members, and unconfirmed identities ignored/none. Fresh foreign presence → read-only editor + "Sara opened T-SK-014 about 1m ago" + **Edit anyway**; presence dots. Never blocks saves; wrap-up bypasses soft locks.

## 12. Visibility, privacy & authorization honesty

**Scoping (filtering only):** My work = my engagements + records where I'm ownerId/preparerId/reviewerId + my private tasks + engagement-less records; **My work ⇄ All** toggle per user. Hides nothing from a determined reader. **Authorization (cooperative UI, not security — say so):** lead actions → `engagement.leadId`; admin actions (schema migration, workspace settings, deactivation) → `workspaceAdminIds` (fallback `createdBy`); personal `role` = presentation default only; SharePoint permissions are the real boundary. **Private tasks:** structurally excluded (projection + fail-closed + presence ban); optional personal backup on personal OneDrive; excluded from weekly/print by default; shared→private honest copy + atomic archive/recreate; **reservations** hours-only opt-in ("excludes private commitments" shown when declined). **Deliverable local notes** per §8: local sidecar, never shared anywhere.

## 13. Planning workflow

**Plan:** `estimateH` + start/due; auto-spread across workdays; cells editable; auto-spread only on creation and explicit Rebalance. **Grid:** member × workday; engagement filter; Σ allocated (incl. reservations) vs capacity — neutral <80%, amber 80–100%, red >100% with overage; zero-capacity days dimmed; chips → editors; lead drags within a task's days. **Fit Check** (hosts Commit): overallocated member-days · estimates without allocation · allocation before start/after due/on zero-capacity days · allocated ≠ scheduled · no valid workdays · unresolved/non-member assignee · idle members; commit-with-flags = explicit override recorded in the baseline. **Check-ins:** My day panel; **Wrap up my day** (<60s, keyboard-only, ghost suggestions never prefills, rows write only when confirmed/edited, unplanned touched tasks listed, only `blocked` mandatory); inline anytime with **Day not finished** (`unconsumedToday = max(0, plannedToday − hoursTodayTotal)` — approximation; `hoursLeft` authoritative). **Data quality:** Known / Stale (effective `workDate` earlier than the most recent **elapsed** allocated workday ≤ today; future allocation never makes today stale) / Unknown; overrun math on Known only, else neutral "Needs update". **At-risk (derived):** Known overrun vs `futurePlannedH×(1+tol)` (or plan exhausted with Known remaining) · self-flag · Blocked ≥ `blockedRiskDays` working days (canonical events); Blocked > At risk > On track; Overdue stacks; badges show reasons; lead At-risk panel. **Drift & rebalance:** drift lines; Rebalance re-spreads Known remaining with preview; removed days = explicit zeros; lead edits estimates/assignees/due; assignee re-spreads own future days only; all edits → typed activity.

## 14. Insights & analytics

Lead **Insights** (`leadId`) + personal **My accuracy**; measured against `kickoffBaselineId` and canonical ancestry only. **Per engagement:** planned vs actual; on-time = canonical `completedAt` ≤ baseline due; post-baseline revisions + slip (legacy-marked events excluded); replans; blocked totals; aggressiveness read (≤1.1 held / 1.1–1.3 hot / >1.3 aggressive); calibration before person views. **Per member (gated n ≥ 5 completed baselined tasks AND ≥ 40 baseline hours across ≥ 2 engagements):** overrun vs team, on-time vs team, blocked frequency; completeness shown; Incomplete actuals excluded unless typed acceptance; attribution baseline→owner / actuals→check-in author; transfers split; blocked/transferred broken out first; archived included with reason; `system:legacy-import` excluded. **Never** ranked lists, red/green scores, league tables. Labels: "Planning evidence, not performance assessment." **Exports:** scope-respecting weekly/print; optional At-risk section; Insights print block; full-backup palette command (own private tasks and local notes only via explicit choice + warning).

## 15. Existing-code reconciliation summary (D1/P1)

Complete contract in §6/§8: findings `revisedDue`; approvals/reviews; `dueTime`; subtask dates; milestones + `milestoneId`; `budgetHours`/`spentHours` (all levels → legacy read-only); `pct` visual-only; custom fields/doc types → stable ids; avatars → bounded; deliverable notes → local sidecar; unknown actors → `system:legacy-import`. **No two parallel hour systems; no unreconciled persisted field.**

## 16. Implementation phases (in order; each independently shippable)

- **T0 — environment only** (two real machines): directory picker/read/write from `file://` · handle persistence + re-grant · IndexedDB availability/quota · `navigator.locks` · `crypto.subtle` · OneDrive latency (diagnostic files) · offline write → delivery. Gate for T2/T3.
- **T1 — safe persistence** (solo value now): IndexedDB stores (workspace-scoped) + migration, state queue + projection scaffolding + **canonical serializer + golden fixtures**, persisted writer state, handles, file queue + attempt records + read-back verify, single-session guard, banners, real save target.
- **D1 — data foundation:** `APP_VERSION`; schema-5; person registry + lifecycle events + prefix reservation + avatar bounds; engagement entities + migration; ownerId migrations; envelopes + HLC-lite; lineage events incl. `consequentialFieldEvents` (+`seq`); genesis + full legacy contract (§6); per-item collections (checklists, custom fields by id, doc types, milestones) + `engChecks` rekey; comment events + local-notes sidecar; authored derived work + seeds.
- **P1 — planning core** (needs D1): plans + versioned allocations, capacity/reservations, daily-total check-ins + deliverySignal, data-quality states, risk + drift, rebalance, wrap-up/inline, dueTime/subtask rules, legacy-hours lockdown.
- **P2 — grid, Fit Check, baselines** (needs D1).
- **T2 — team sync** (needs T0 + D1): Create/Join bootstrap, replica publish + revision gating, event-first merge + graph-wide conflicts, contaminated-id handling, schema gate, quarantine, identity confirmation, queue barrier; **gate = two-machine correctness pilot + scale test; budgets recorded here before rollout.**
- **T3 — presence, scoping, privacy:** presence (shared only) + soft locks, scoping, private tasks + projection verification, reservations, deactivation + reassignment queue.
- **P3 — Insights.**

## 17. Acceptance criteria before team rollout

All criteria are executable fixtures built alongside implementation (per phase), not prose checks.

**Bootstrap & workspace:** fresh browser with sample data joins → publishes none of it · solo engagements never published by Join; only by explicit Import · private tasks/local notes create no outbox entry during Join · Create→Use-current imports previewed counts exactly once across restart · Create→Empty publishes nothing · workspace-A batches/attempts can never write through workspace B's handle; switching mid-write verifies/clears only the captured A sequence · sample-load/clear-all unavailable in team workspaces.

**Genesis, migration & import:** migration ×2 → byte-identical ids, no duplicates; interrupted = uninterrupted · two machines reading the creator-migrated workspace derive zero conflicts · Join never creates a second genesis lineage · first real edit names genesis · a live v1 fixture (revised dues, approvals, validation + review rounds, timelines, custom fields, milestones, uploaded avatars, manual hours, dueTime, subtask dates) migrates with no missing values · unknown actors show as `system:legacy-import`, never the creator · re-import via the persisted mapping is idempotent and can never mint a duplicate event id with different content.

**Lineage & conflicts:** A→B→C before a peer sync = sequential, no conflict · two chains from one ancestor → exactly one deterministic pairwise conflict · **three concurrent heads merged in all six arrival orders → identical pairwise conflict-id sets, identical `baseOpId` (nearest common ancestor), identical winner, identical grouped UI** · one group resolution resolves all known pairs; a later branch alone becomes unresolved · ancestry reconstructible after convergence; branch values retained (events carry from/to + seq) · restore = valid canonical operation · branched check-in corrections → byte-equivalent conflict derived post-union · same-id/different-body events → identical contaminated/quarantine state in every scan order · cycles/corruption → lineage-incomplete + LWW, never a hang · diagnostics never enter replicas.

**Merge & versions:** clock-skewed replicas → identical state; received timestamps never rewritten · **a field stamped a year ahead shows the diagnostic, remains deliberately editable, and the descendant edit wins identically in both merge orders and after restart** · same replicas any order → identical state · writer seq/ts survive restart · corrupt replica never blocks others · version gates hold · revisions reserved once per attempt, gaps never reused, never backward; restart finds a written matching attempt and completes verification without duplicate content change; unchanged peers skipped without missing a higher revision.

**Outbox:** rapid A→B→C debounces to one verified write, no stuck batch · a multi-field action is wholly in the verified snapshot or wholly pending · a batch enqueued during file I/O stays pending unless captured · identical projection → no write, no revision · metadata-only changes never alter the domain hash · no revision ping-pong between idle peers · expected revision + wrong hash stays pending → degraded banner after retry.

**Privacy & collaboration:** private-only edits → no replica write, no outbox entry · serialization fails closed on private content/ids/local-note fields · shared→private writes only the archive op · reservations expose hours only · no presence for private records · **deliverable local notes never appear in a replica, outbox, presence, export, or peer state** · concurrent comments both survive in canonical order · private-task comments create no shared event · only http(s) URLs render as links; navigation user-initiated.

**Identity:** one profile storage location; one ID domain; renames display-only · known prefix collisions blocked; delayed collisions detected and contained (two-machine blind-join test: both records survive, ambiguity surfaced, creation blocked until repair) · historical prefixes of inactive members never reused · **concurrent profile edit + deactivation → member inactive everywhere; stale `active:true` can never reactivate; admin reactivation keeps the same id** · deactivated local identity opens read-only, writes nothing · team writes wait for identity confirmation.

**Lifecycle & records:** losing Complete never mints `completedAt` · losing due branches never inflate counts · approval analytics follow accepted lineage · ordinary edits never reactivate an archive; archive-period edits show the informational note and survive restore · offline replicas cannot resurrect archives · no purge · archived baselined tasks stay in analytics · every event orders through `(ts, by, id)` · custom-field/doc-type renames preserve linkage and values · concurrent milestone additions survive; deletion not undone by unrelated edits · merged task/subtask date violations preserved and flagged, never clamped · dueTime renders identically in workspace time on both machines.

**Derived work:** source transition + derived task publish atomically with one authored ref · peers never mint different refs/values · missing derived work without a seed → repair action, never guessed state · concurrent source branches never leave two active tasks for one accepted cycle.

**Planning & analytics:** check-ins never double-count · `blocked` reaches the risk queue directly · missing check-ins → Needs update, never fabricated overrun · future allocation never stales today · removed hours/reservations cannot resurrect · check-in actuals are the only actual-hour source; `pct` affects no analytics · competing kickoffs block identically; acceptance converges · concurrent collection additions survive · role cannot grant lead behavior · UI never claims cloud sync.

**Canonicalization & size:** golden fixtures (Unicode, missing optionals, explicit null, reordered keys, fractional hours, `-0`) → identical bytes/hashes · every deterministic id uses its named domain tuple · oversized avatars rejected; built-ins contribute only tokens.

**Scale test** (real machine; budgets recorded here before rollout): 6 members · 2 years of check-ins · ~1,500 work records · ~400 findings+deliverables · ~30,000 events · conflicts + archives. Measure: cold start, IndexedDB load, changed-peer merge, unchanged-folder scan, save-to-verified-write, dashboard + grid render.

## 18. Invariants (do not violate)

- Zero network calls. One writer per file. One serialized state pipeline; IndexedDB commit before UI success; file writes downstream, only from the allowlisted projection (fail-closed), only after identity confirmation, only through workspace/writer-matched handles.
- Envelopes only for changed fields; received timestamps never rewritten; local envelopes stamp `ts = max(wallClock, predecessorTs)`; total order `(ts, by, seq, opId)`; ancestry-walked classification with guards; conflicts derived graph-wide post-union, pairwise, fully derived, immutable; resolutions separate; contaminated ids exclude all variants; diagnostics local-only.
- Consequential mutations write typed events (field/from/to/seq) atomically with envelopes; analytics consume canonical ancestry only; genesis and derived-work creation have one deterministic author; deterministic ids use domain-separated canonical tuples.
- Append-only entries: `{id, type, ts, by}` base, immutable, `(ts, by, id)` order; duplicate id + different body = contamination, never array-order selection.
- **Join never publishes prior solo state.** State keys by workspaceId; workspace switch is a queue barrier; foreign-workspace check precedes every write. Sample-load/clear-all unavailable in team workspaces.
- Archive is the only removal; dominance unconditional until explicit restore; archive-vs-edit informational only.
- Collections per-item; profile fields per-field; **member lifecycle only via lifecycle events — profile saves never carry it**; removal = `active:false`/explicit zero/clear, never a missing key.
- Writer state persists transactionally; seq never reused; revisions reserved per attempt, gaps permanent; a verified snapshot subsumes earlier batches of the same writer; state never rolls backward; one canonical serializer; domain hash excludes volatile metadata.
- Actual hours from check-ins only; `pct` visual only; overrun math on Known only; at-risk derived; auto-spread never implicit; baselines immutable; `kickoffBaselineId` sole reference; assignee rebalance limited to own future days; hours on tasks; one member assignee.
- One person registry, one ID domain; prefixes permanently reserved (incl. deactivated members'); renames never rewrite refs/attribution; avatars bounded, built-ins by token.
- Private content — records, child data, comments, op ids, presence, **deliverable local notes** — never enters the team folder; private-only mutations produce no team writes.
- Wrap-up: keyboard-only, ghosts not prefills, <60s, only `blocked` mandatory. Lead/admin gating by shared facts; personal `role` presentation-only. Merge failures never corrupt or block. Statuses/ratings/lifecycles not user-editable.

## 19. Open items (decisions, not oversights)

1. Workspace timezone at creation (proposal: America/Toronto).
2. Default chargeable hours/day (6) — confirm against real meeting load.
3. Check-in-silence risk signal — after wrap-up habit established.
4. Insights → report gate — parked until Insights exists.
5. Physical purge/compaction — deferred (tombstone+ack design).
6. Scale-test budgets — measured in T2, recorded here before rollout.
7. Exact avatar byte limit — explicit and tested (implementation latitude within §21).

## 20. Closed decisions (do not reopen without a failed test, measured evidence, or a concrete defect)

Full replica per member, single writer per file · OneDrive local-folder transport · zero app network calls · IndexedDB working store · Create/Join isolation · allowlisted projection + fail-closed privacy check · canonical content-hash snapshot writes · single-author deterministic genesis · event-first ancestry merge · graph-wide pairwise conflicts · archive dominance, no purge · private-task and local-note exclusion, no private presence · honest prefix containment · startup identity confirmation · check-in/planning/capacity/baseline/risk/insight concepts · T0–T3 phasing with two-machine gates.

## 21. Implementation latitude (no review needed)

Exact UI copy/placement for banners and repair panels · animation, spacing, responsive polish, icons · internal helper/module names · retry timing and debounce durations within documented behavior · the avatar byte limit (explicit + tested) · performance budgets (measured in T0/T2) · the §19 product defaults.
