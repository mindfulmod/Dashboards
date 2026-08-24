# AuditDesk — Team Mode Spec (v2)

Decisions locked 2026-08-23 via design discussion. Extends `01-product-spec.md`; all v1 locked decisions stand unless explicitly amended here. Target file: `AuditDesk.html` (single file, all changes inline, no new files, no CDN, no network calls — unchanged hard constraints).

## One-liner

Multiple auditors collaborate through a shared OneDrive-synced folder: one JSON replica file per user, field-level merge in the app, advisory presence/soft-locks, engagement-scoped dashboards, and local-only private tasks. The app itself still makes **zero network calls** — OneDrive's sync client is the only thing that touches the network.

## Locked decisions

| Area | Decision (2026-08-23) | Declined |
|---|---|---|
| Sync transport | Shared OneDrive/SharePoint folder, synced locally on every machine; app reads/writes it via File System Access API directory handle | Any hosted backend, Microsoft Graph calls, SharePoint Lists (re-declined) |
| File layout | **One file per user**: `auditdesk-<userTag>.json`. Each file is that user's full replica snapshot (may contain records authored by anyone). Single writer per file — a user only ever writes their own file | Single shared file (collision risk); per-engagement files (no safety benefit, breaks free-text engagements; revisit only if folder-level access control is ever required) |
| Merge granularity | **Field-level last-writer-wins** via per-field timestamps; append-only arrays union by id; deletes via tombstones | Record-level LWW (silently loses concurrent edits to different fields of the same record) |
| Concurrency signal | Advisory **presence files** + **soft lock** (read-only with "Edit anyway" override) | Hard locks (OneDrive sync latency makes them unreliable; stale-lock deadlocks) |
| Save target | Directory handle chosen once per machine, **persisted in IndexedDB**; saves write by filename inside it, never via a save picker | Per-save `showSaveFilePicker` (handle lost on reload → picker drift → Downloads folder) |
| Fallback behavior | FSA failure shows a **persistent warning banner** naming the consequence; download fallback kept but never silent | Silent toast-and-download (current behavior — the cause of files landing in Downloads) |
| Dashboard scope | Default filter = my engagements + records assigned to me; "All engagements" toggle for everyone; lead role defaults to All | Hiding data from non-members at the file level (it's a relevance problem, not secrecy) |
| Private tasks | `private: true` tasks are **never written to the shared folder** — local partition only, optional personal-OneDrive backup file | App-level "hidden" flag on shared data (privacy theater — files are readable in Notepad) |
| Refs | New records get per-user namespaced refs: `F-<TAG>-001`, `T-<TAG>-001`, `D-<TAG>-001` | Global sequence (concurrent creation mints duplicate refs); renumber-on-merge (breaks refs quoted in emails) |
| Clock model | Field timestamps = ISO wall clock (corporate NTP assumed); ties broken by userTag ascending | Vector clocks / CRDT library (overkill for team size; no installs anyway) |

## Prerequisite — Phase 0 verification (do before building)

On an actual locked-down work machine, in Edge from `file://`, confirm:

1. `window.showDirectoryPicker` exists and successfully opens on the OneDrive team folder with `{mode:'readwrite'}`.
2. A `FileSystemDirectoryHandle` stored in IndexedDB survives a full browser restart and `requestPermission({mode:'readwrite'})` succeeds afterwards from a click/keydown gesture.
3. Files written into the local OneDrive folder sync up, and peers' file changes sync down (observe latency; expect seconds, occasionally minutes).

If (1) or (2) fails by policy: team mode falls back to manual multi-select load (`<input type=file multiple>` pointed at the folder) + download saves, and each teammate sets Edge's download location to the team folder. All merge logic below is identical; only the read/write plumbing differs. Build the FSA path first regardless; the manual path is the degraded mode.

## Identity

- Personal settings gain `userTag`: 2–4 lowercase letters (initials), set during team onboarding, **must be unique within the team** (the app warns if a peer file with the same tag exists that isn't ours).
- `createdBy: userTag` stamped on every new record.

## Storage partitions

| Partition | Where | Contents |
|---|---|---|
| Working copy | `localStorage['auditdesk.v1']` (unchanged key) | Full merged state incl. private records — same as today |
| Personal settings | `localStorage['auditdesk.local.v1']` (new) | `me`, `userTag`, `role` (`member`\|`lead`), theme, `topbarColor`, dashboard-scope toggle state. **Moved out of shared `settings`** — these must not merge across users |
| Shared replica | `<teamDir>/auditdesk-<userTag>.json` | Full state **minus** private records and personal settings |
| Private backup | Optional explicit save of `auditdesk-private-<userTag>.json` via save picker (user points it at **personal** OneDrive, never the team folder) | Private tasks only |
| Presence | `<teamDir>/presence-<userTag>.json` | See Presence section |
| Handles | IndexedDB db `auditdesk-fs`, object store `handles`, key `teamDir` | The persisted `FileSystemDirectoryHandle` |

Migration: on first run of the new version, if `settings.me` etc. exist in shared state, copy them to `auditdesk.local.v1` and delete from shared `settings`. Bump `meta.schemaVersion` to 4; `ensureDefaults` backfills all new fields (`fieldTs`, `createdBy`, `deletedAt`, `updatedAt`) as absent/epoch-zero so legacy data merges as "oldest".

## Merge model

### Record shape additions (findings, tasks, docs)

```
{ ...existing fields,
  createdBy: "ma",
  updatedAt: ISO,                       // max of fieldTs values; convenience for sorting
  fieldTs: { title: ISO, owner: ISO, ... },  // one entry per scalar field ever edited
  deletedAt?: ISO }                     // tombstone
```

Every editor save / inline mutation stamps `fieldTs[field] = now` for exactly the fields that changed (compare old vs new value; unchanged fields keep their timestamp). This is the single most important implementation detail — a save must NOT stamp every field, or it degenerates to record-level LWW.

### Merge algorithm (per sync)

Inputs: local state + every parsed peer file in the folder (including our own file — harmless). For each record id across `findings`/`tasks`/`docs` in any replica:

1. **Unknown id** → adopt the record wholesale.
2. **Known id, scalar fields** → for each field name present in either side's `fieldTs`, take the value from the side with the later timestamp; tie → lower `userTag` (of the replica file's owner) wins. Fields with no `fieldTs` entry anywhere (pure legacy) → keep local.
3. **Tombstones** → if any replica has `deletedAt`, the record is deleted, unconditionally and permanently (deletion always wins; state this in the delete-confirm dialog). Deleted records are kept in state as tombstones (id + deletedAt + ref only is enough) and written to the replica file so deletion propagates; they are excluded from every view, count, and export.
4. **`reviews` and `timeline`** → append-only: union by entry `id` (timeline entries need an `id` added — `uid()` at creation; legacy entries get a synthetic id of `ts+'|'+text` during migration). Sort reviews by `round`, timeline by `ts` desc. Never edit or delete entries.
5. **`subtasks`** → union by subtask `id`; for a subtask present on both sides, treat the whole subtask as one LWW unit using a per-subtask `updatedAt` (stamped on any subtask change). Removed subtasks get a `deletedAt` tombstone like records.
6. **`custom` values** → per-key LWW using `fieldTs['custom.'+label]`.

Other state:

- **`engagementMeta`** (gains `members: [names]`, `lead: name`) → per-engagement per-field LWW via `engagementMeta[eng].fieldTs`.
- **`engChecks`** → value shape changes from `bool` to `{v: bool, ts: ISO}`; per-item LWW. Migration wraps legacy booleans with `ts: epoch`.
- **Shared `settings`** (`people`, `checklists`, `customFields`, `docTypes`, `docSeed`) → per-key LWW via `settings.settingsTs = {key: ISO}`. `people` merges as a union by name, per-person LWW on color/avatar.

After merge: recompute `updatedAt`, run `syncDerivedWork`, `persistQuiet()`, re-render, and toast a summary when anything changed: `"Synced — 3 updates from sk, 1 new finding from jd"`. No toast when nothing changed.

### Sync triggers

- App load (after directory permission is available).
- Window `focus`.
- After every successful save (write own file, then read peers).
- Manual **Sync now** button in the topbar (shows last-sync relative time, e.g. "Synced 2m ago").
- Background interval every 60s while visible.

Peer file reads that fail to parse → skip that file, show a non-blocking warning naming the file; never let one corrupt file abort a sync.

## Save path (replaces current `saveToFile` behavior in team mode)

1. Team onboarding: "Choose your team's OneDrive folder" → `showDirectoryPicker({mode:'readwrite', id:'auditdesk-team'})` → store handle in IndexedDB → immediately write our replica file.
2. Ctrl+S / autosave-on-mutation: `dirHandle.getFileHandle('auditdesk-'+userTag+'.json', {create:true})` → write. **No picker is ever shown on save.** Topbar save indicator shows the target: `"→ <folderName>/auditdesk-ma.json"`.
3. On startup: load handle from IndexedDB; `queryPermission({mode:'readwrite'})`. If `'prompt'`, defer — the first user gesture (Ctrl+S, Sync now, or a "Reconnect" button on the banner) calls `requestPermission()`. Until granted, show a slim banner: `"Team folder needs one click to reconnect — press Ctrl+S."`
4. `NotFoundError` (folder moved/renamed) → clear stored handle, show the choose-folder step again with an explanatory message. Never guess.
5. Any other FSA failure → download fallback **plus a persistent red banner**: `"Save fell back to a download (check your Downloads folder). Team sync will NOT see this file until the team folder is reconnected."` The banner stays until a successful folder write.
6. Single-user mode (team not enabled) keeps the current single-file behavior, but gains the same persisted-handle treatment: persist the `showSaveFilePicker` file handle in IndexedDB (key `soloFile`), pass `id`/`startIn` on any picker, and use the same loud fallback banner. This part is worth shipping first — it fixes today's Downloads drift independently of everything else.

## Presence & soft locks

- File: `<teamDir>/presence-<userTag>.json` → `{ userTag, name, recordId, recordRef, recordType, ts }`.
- Opening any record editor writes the file; a 30s heartbeat refreshes `ts` while the editor stays open; closing the editor (or `beforeunload`) writes `{ userTag, name, recordId: null, ts }`.
- All presence files are re-read every 10s while any view is visible (cheap local reads).
- **Stale rule**: presence with `ts` older than 3 minutes is ignored.
- **Soft lock**: opening an editor for a record named in a fresh foreign presence file renders it **read-only** with a banner: `"Sara opened T-SK-014 about 1m ago."` and an **Edit anyway** button that unlocks the form. Register rows / board cards for such records show a small presence dot with the person's color and a title tooltip.
- Presence is advisory: it lags by OneDrive sync latency and never blocks a save. Field-level merge is the correctness backstop; presence exists to make same-field collisions rare.

## Refs

- `nextRef` becomes per-user: scan only refs matching `^[FTD]-<TAG>-(\d+)$` for our own tag; new refs are `F-MA-001` style.
- Legacy refs (`F-001`) remain valid and untouched; all ref-parsing regexes accept both forms: `/^([FTD])-(?:([A-Z]{2,4})-)?(\d+)$/`.
- Search, palette, and paste-parser matching updated accordingly.

## Visibility & scoping

Two mechanisms, deliberately different:

### 1. Engagement scoping (relevance — filtering only)

- `engagementMeta[eng].members` (names from `settings.people`) and `.lead`, edited on the engagement page.
- **My work scope** (default for `role: member`): records whose engagement lists me as member **plus** any record where I am owner/preparer/reviewer regardless of membership **plus** my private tasks **plus** records with no engagement.
- Applies to: Dashboard, Tasks (both views), Reviews hub, calendar. Registers and Engagements view get the same default with an inline **All engagements ⇄ My work** toggle (state remembered in personal settings). Lead role defaults the toggle to All.
- Explicit non-goal: this hides nothing from a determined reader — non-members can toggle to All or open the JSON. It is noise control, not access control. If genuine confidentiality per engagement is ever needed, that is a separate folder with its own SharePoint permissions containing its own per-user files (structure already compatible; out of scope for v2).

### 2. Private tasks (confidentiality — file-level exclusion)

- Tasks gain `private: true` (creation checkbox + editor toggle; lock icon in every list).
- Private tasks are **stripped from the shared replica file at write time** — they exist only in localStorage and the optional personal backup file. Structurally invisible to teammates, including the lead.
- Excluded by default from weekly status text and print report.
- Toggling **private → shared**: confirm dialog ("This task will become visible to the whole team.").
- Toggling **shared → private**: confirm dialog with honest copy: `"This removes the task from your shared file going forward, but teammates who already synced it keep their copy. To also delete their copies, delete the task instead. Truly private tasks should be created private."` On confirm, write a tombstone for the shared id and re-create the record privately under a new id.

## Exports & reports

- Weekly status text and print report respect the current scope toggle (lead on All ⇒ team-wide report; member default ⇒ their engagements).
- JSON backup (Ctrl+S) is now the replica write; a separate palette command **"Export full merged backup"** produces a classic single-file download of the entire merged state (minus others' private tasks, which we never had) for archival.

## Onboarding additions

Team setup flow (from Settings → "Team sync", or first-run prompt):

1. Enter `userTag` (validated: 2–4 letters, not used by an existing peer file).
2. Choose team folder (directory picker) — stored handle, permission model as above.
3. App writes replica + presence file, performs first sync, reports peers found: `"Connected — found 2 teammates: sk, jd."`

## Implementation phases (each independently shippable)

1. **Phase 1 — save-path fix (single-user value now):** persisted handles in IndexedDB, no-picker saves, `id`/`startIn`, loud fallback banner, target shown in topbar. *(Fixes the Downloads-folder drift on its own.)*
2. **Phase 2 — merge foundation:** schemaVersion 4 migration, `fieldTs`/tombstones/`createdBy`, timeline entry ids, `engChecks` value wrapping, settings split (personal vs shared), namespaced refs.
3. **Phase 3 — team sync:** userTag onboarding, per-user replica write, folder scan + merge algorithm, sync triggers, sync toasts, corrupt-file tolerance.
4. **Phase 4 — presence:** presence files, heartbeat, polling, soft-lock read-only editor with override, presence dots.
5. **Phase 5 — scoping & privacy:** engagement members/lead, My work ⇄ All toggle across views, role default, private tasks partition + confirm dialogs + export exclusions.

## Invariants (do not violate)

- The app makes **no network calls**; only the OneDrive client syncs.
- One writer per file in the team folder, ever.
- A save must stamp `fieldTs` only for fields whose values actually changed.
- Deletion (tombstone) always wins over concurrent edits.
- Private records never enter the team folder in any form.
- Merge failures on one peer file must never block sync of the others or corrupt local state.
- All v1 keyboard behavior, statuses, and lifecycle logic unchanged.
