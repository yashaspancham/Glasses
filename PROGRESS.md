# Glasses — Progress

Last updated: **5 Sep 2026**

## What this project is

A PC resource monitor: a background recorder samples system usage to local storage, and a CLI answers questions about the past — "what was my machine doing at 5:45pm last Tuesday?"

**The real purpose is process practice.** The build is deliberately small so the effort goes into doing it the way a good engineering team would: written docs, real review gates, tests, CI. Domain content (energy / stats / cloud) is deliberately minimal.

**Budget:** ~35 hours total, of which only ~10 is building.

---

## Where we are

| Stage | Artifact | Status |
|---|---|---|
| 1. Define the problem | `PRD.md` | ✅ **Approved** (Shreyas, 9 Aug 2026) |
| 2. Design the solution | `DESIGN.md` | 🔨 In progress — Technical goals & non-goals done (G1–G26, NG1–NG9), Overview drafted with diagram. Detailed Design: "The Recorder" subsection complete (D1–D15). "The Storage" subsection drafted (D16–D20 — SQLite chosen, Postgres/MySQL and flat files rejected). "The Database" section is a full `CREATE TABLE` schema (rewritten 30 Aug). **"The Display" subsection now drafted (D21–D28, 2 Sep)** — click, timestamp format, read-only SQLite access, NULL→"-", clock-jump marker, plain text, `schtasks` check on `glasses start`, and a vertical label:value layout (with worked example) for single-point commands (`now`/`at`). **"Alternatives considered" section added (2 Sep)** — Nuitka, Python, SQLite, Task Scheduler, SYSTEM account, click, each with a rejected alternative and reason; typos and vague reasoning fixed on review pass. Range-command (`last`/`top`) table format still undecided. **Risks section now written (5 Sep)** — `## RISKS`, R1–R7, covering crash-loop recovery, AV false-positive, SYSTEM-account access, WMI VPN misclassification, storage-budget/`auto_vacuum`, drive-letter identity, and `psutil` CPU-behavior uncertainty (ties to D7). Reviewer-not-confirmed deliberately left out of the tracked Risks per Yashas's call. **Testing strategy — in progress**, working through the framework conversationally (test units per component, what to fake/mock, ugly-case list, requirement traceability); nothing drafted into the doc yet. |
| 3. Break down the work | `PLAN.md` | ⬜ Not started |
| 4. Build | PRs | ⬜ Not started |
| 5. Test | test suite | ⬜ Not started |
| 6. Ship | installer | ⬜ Not started |
| 7. Learn | `README.md` | ⬜ Not started |

Reference: [`software-development-process.md`](software-development-process.md) — the general method, written up from doing stage 1.

---

## Decisions locked so far

From the PRD, approved:

| Decision | Value |
|---|---|
| Platform | Windows 10 / 11 only for v1 |
| Interface | CLI only — no GUI in v1 |
| Sample interval | Every 30 seconds |
| Retention | 30 days |
| Storage budget | Under 150MB after 30 days |
| Processes per sample | Top 10 by CPU, top 10 by memory |
| Time input | Fixed timestamp format — no natural language |
| Output | Plain text table, 80 columns |
| Boundary | Glasses observes; it does not fix, kill, or recommend |

22 numbered requirements (R1–R22), prioritised Must / Should / Could. 11 explicit non-goals.

---

## DESIGN.md progress

**Technical goals & non-goals — done.** 26 goals, 9 non-goals, each numbered (G1–G26, NG1–NG9) and traceable back to specific PRD requirements. Went through several rounds of catching contradictions and redundancies before landing clean. Highlights of what got decided along the way (still needs formal writeup with alternatives in Detailed Design):

- **Timestamps**: three fields per sample — UTC (source of truth for ordering), a monotonic/relative timestamp (immune to manual clock changes), and local machine-clock time (preserves historical display fidelity if the timezone setting changes later)
- **Clock changes**: detected by comparing monotonic vs. wall-clock elapsed time between samples; flagged as an explicit marker (no retroactive correction of past timestamps — NG6)
- **Gaps**: "system was off" gets a marker line with no per-timestamp rows for that period; a corrupt/missing value keeps its timestamp but shows `-`; both marker types show the linked before/after timestamps so the display "states plainly" per R12, not relying on color (color can't be load-bearing — it's stripped by piping, `--json`, and non-ANSI terminals)
- **Disks**: all internal disks tracked (via WMI bus-type, since `GetDriveType`'s removable/fixed split doesn't map to internal/external); external/USB disks explicitly excluded (NG8)
- **Network**: physical interfaces only (via WMI's `PhysicalAdapter` flag — excludes Bluetooth PAN, VPN, virtual switches); both an aggregate total and a per-channel breakdown
- **Battery**: no battery hardware → empty cell; battery present but drained → literal 0% (same absence-vs-zero pattern as gaps)
- **CPU%**: `psutil.cpu_percent(interval=None)` — first call after every recorder start (including after every reboot, since psutil's internal state doesn't survive a process restart) is a throwaway priming call, discarded; real sampling starts from the next call
- **Single instance**: a second recorder instance is actively blocked from starting, not just assumed not to happen — this is what makes the concurrency goals (G7/G8) actually hold
- **System-level, not per-user**: recording continues regardless of which Windows user is logged in
- **Uninstall**: removes all recorded data, not just the binary

**Overview — drafted.** Three components (Recorder, Storage, Display) inside an OS boundary, plus a User/CLI actor. Diagram at `Glasses-overview.drawio.png`, reviewed and embedded.

**Detailed design ("The Recorder") — done, see below (1 gap remains — crash/restart resume still needs writing into Risks). "The Storage" and "The Database" — drafted (SQLite choice with rejected alternatives, and a first-pass table list). Display, Alternatives considered, Risks, Testing strategy, Open issues — not started.**

### The Recorder — now a local decision table (D1–D15, numbered within that section only, distinct from this file's D1–D9 tracking list below)

- **D1** Boot mechanism: Windows Scheduled Task `GlassesRecorder`, registered once at install (`glasses start`) via `schtasks /Create`, trigger `ONSTART`, account `SYSTEM`
- **D2** Language/runtime: Python 3, compiled to a standalone `.exe` via Nuitka (rejected-alternatives writeup for Alternatives Considered still owed)
- **D3** Metrics & modules: CPU/memory/disk/network/battery/top-10 processes via `psutil` and `wmi`; also `logging`, `sqlite3`, `datetime`
- **D4** Logging: `C:\ProgramData\Glasses\logs\glasses_recorder.log`, 7-day retention, counts against the 150MB storage budget (wording fixed this session — was grammatically ambiguous)
- **D5** Timestamp storage: UTC, local machine time, and monotonic timestamps all stored per sample (wording tightened this session; note — this sentence no longer states the clock-change *detection* logic itself, see open item below)
- **D6** Single-instance enforcement: named mutex `Global\GlassesRecorderMutex`; second instance sees `ERROR_ALREADY_EXISTS`, logs, exits. (The user-facing "you're already running" message is Display's job, not Recorder's — deferred to when Display gets written.)
- **D7** CPU% priming: one throwaway `cpu_percent(interval=None)` call at recorder startup (system-wide); any PID seen for the first time is individually primed and shown as `-` for that one sample, not a misleading `0%`
- **D8** Sampling loop: `sleep_time + record_time = 30s` (sleep is computed as the remainder after the work, not a hardcoded number) — resolves the drift a naive `sleep(30)` would have
- **D9–D11** Failure handling: a failed per-value read logs and shows `-` for that field; a failed write logs and the recorder continues; both keep one bad reading from taking down the whole sample or the process
- **D12** Battery: `psutil.sensors_battery()` — `None` → empty cell, present-but-drained → literal `0%`
- **D13** Disk enumeration: `psutil.disk_partitions(all=False)` (Windows `GetDriveType()`'s fixed/removable split; also gives drive letters for free) — written into DESIGN.md this session
- **D14** Network adapter enumeration: WMI `Win32_NetworkAdapter.PhysicalAdapter` filter, enumerated once at startup, joined to `psutil.net_io_counters(pernic=True)`'s per-NIC keys via `NetConnectionID` — written into DESIGN.md this session
- **D15** First sample: recorded 30s after the recorder is enabled — written into DESIGN.md this session

**Gaps still open in Recorder:**
1. **Crash/restart resume (G5, G6).** Decided (2026-08-23): accepted as a known v1 risk, not engineered around — if the recorder crash-loops past Task Scheduler's retry limit, it stays down until next reboot. Options considered and rejected for v1: a second time-based Task Scheduler trigger as a self-healing backstop; switching to a Windows Service with SCM recovery policy (reopens the D1 boot-mechanism decision). **Still needs to be written into DESIGN.md's Risks section** — not yet drafted. This is now the only undrafted Recorder gap; disk and network enumeration (below) were written in this session.

### The Storage (D16–D20) and The Database — schema rewritten 30 Aug 2026

- **Storage engine (D16–D20):** SQLite. Rejected Postgres/MySQL (needs a running server process — conflicts with G1's CPU/memory budget and the local-only non-goal) and flat files/CSV (no indexed queries, no write atomicity for G8, poor fit for variable-length multi-disk/NIC data).
- **The Database section is now a real `CREATE TABLE` schema in DESIGN.md**, replacing the earlier one-line table list. Tables: `sample` (id, utc/machine/monotonic timestamps, cpu_percent, memory_free/total_bytes, battery_percent — all singleton per-tick values live directly on `sample`, no more separate `memory`/`timestamps` 1:1 tables), `disk` (static per-disk identity + total_bytes, enumerated once — mirrors `network_interface`), `disk_usage` (sample_id, disk_id, used_bytes — PK `(sample_id, disk_id)`), `network_interface`, `network_io` (unchanged), `process` (PK `(sample_id, resource_top_type, rank)`).
- **Previously-open schema gaps resolved this session:**
  - Explicit PKs added for every one-to-many table (`disk_usage`, `process`) and every table generally — was flagged as missing, now fixed.
  - `memory` and `timestamps` folded into `sample` directly (they were 1:1, splitting them only added join overhead with no normalization benefit).
  - Corrupt/missing values are stored as SQL `NULL`, not the literal string `"-"` — `"-"` (G9) is now explicitly a Display-layer rendering choice, so `SUM`/`AVG` aggregates (R9) work without special-casing text in a numeric column.
  - Retention/pruning (R5) now has a concrete mechanism: `ON DELETE CASCADE` on every child table's FK to `sample`, `PRAGMA foreign_keys=ON`, so deleting old `sample` rows cleans up children automatically. (This resolves *how* to delete by age — it does **not** resolve D7, pruning-at-the-size-cap, which is still open — see below.)
  - G7 (reads must not block writes) and G8 (read only when complete) now have concrete mechanisms: `PRAGMA journal_mode=WAL` for G7, and one transaction per sample (`sample` + all its child rows in a single `BEGIN...COMMIT`) for G8 — both written into DESIGN.md's Database section as prose, but **not yet promoted into the D16–D20 decision table** where a reviewer would expect to find them (see gaps below).
  - Gap representation (unchanged from before): no marker row — a gap is the absence of `sample` rows for that period. Threshold for "system was off" vs. scheduling jitter (candidate: 45s = 1.5× interval) still undecided, still unwritten.
  - Clock-change detection (G19): still leaning Display-side computation over the three stored timestamps, not a Recorder-stored flag — still undecided, still unwritten.

**New Storage-level gaps surfaced this session (none of these existed as open items before today):**

1. **D7 — pruning at the size cap is still fully open** (original PRD open question, never resolved). Cascade-delete only handles age-based pruning (R5); it says nothing about what happens if 150MB is hit before 30 days pass.
2. **Pruning ownership unassigned** — nothing says whether the Recorder runs the age-based `DELETE` inline (and how often) or a separate task owns it.
3. **File size after delete — no `auto_vacuum`/`VACUUM` strategy.** SQLite doesn't shrink the `.db` file when rows are deleted; without `PRAGMA auto_vacuum` (must be set before any tables are created) plus periodic `PRAGMA incremental_vacuum`, the file stays at its historical high-water mark regardless of pruning — this can silently violate G2 (stay under 150MB) even with correct age-based deletion. Flagged as the most urgent of the new gaps since it threatens a Must-have success metric.
4. **WAL checkpointing not decided.** The `-wal` file from item above also counts toward the 150MB budget; default auto-checkpoint is probably fine given ~one write/30s, but it's undecided, not stated.
5. **DB file path not stated.** D4 gives the log file path (`C:\ProgramData\Glasses\logs\...`); no equivalent path was ever given for the `.db` file itself.
6. **G7/G8 mechanisms (WAL, per-sample transaction) live only as Database-section prose** — should be promoted into the Storage decision table (as new D21/D22) so they're traceable alongside D16–D20.

### The Display (D21–D28) and Alternatives considered — added 2 Sep 2026

- **CLI framework:** `click`, chosen over `argparse` (too weak for a multi-command CLI) and `fire` (too much magic, less control over help/validation).
- **Timestamp input:** fixed format `YYYY-MM-DD HH:MM:SS` (D22) — resolves the PRD's "fixed format, no natural language" requirement concretely.
- **DB access:** Display reads via `sqlite3`, separate from the Recorder's write path (D23).
- **Rendering:** SQL `NULL` → `-` (D24); a jump in monotonic time not mirrored in machine time gets a marker (D25 — this is the clock-change case, distinct from a "system was off" gap; the gap case itself isn't separately called out in D21–D27, flagged and consciously left as-is for now).
- **Output:** plain text (D26); `glasses start` checks live `schtasks` state each time it runs, rather than trusting a cached assumption (D27).
- **Single-point layout (D28):** `now`/`at` render as a vertical label:value block (Time/CPU/Memory/Disk/Battery/Network), with Top CPU and Top Memory as separate ranked lists underneath — not a row/column table, since a single row with 8+ fields can't fit 80 columns. Worked example written into DESIGN.md.
- **Range-command layout (`last`/`top`) — still undecided.** Two options were discussed: a summary block (peak + average, matching R9's wording) vs. a row-per-sample table with a gap marker row. Not chosen yet.
- **Known gaps raised and explicitly not pursued this session** (per Yashas, "not needed due to various reasons"): battery NULL ambiguity (no-battery vs. corrupt both currently NULL in schema), `--json` output, 80-column table width for row-style output, the "recorder already running" user-facing message, and lifecycle commands beyond `start` (`stop`/`status`/`uninstall`). These are not being tracked as open blockers going forward — treat as deliberately deferred, not forgotten-but-pending.
- **Alternatives considered table** now covers Nuitka, Python, SQLite, Task Scheduler, SYSTEM account, and click, each stating what was rejected and why (e.g. SYSTEM account is required by G21 — recording must survive user logon/logoff/switching, which a per-user account can't do).

### Risks — written into DESIGN.md (5 Sep)

DESIGN.md now has a `## RISKS` section, R1–R7:

1. **R1** — Recorder crash-loops past Task Scheduler's retry limit (G5/G6); how it re-enables without user action.
2. **R2** — Antivirus/Defender could block the Nuitka-compiled `.exe`.
3. **R3** — The system-level (SYSTEM account) recorder could get blocked at the user level.
4. **R4** — WMI `PhysicalAdapter` could misclassify virtual/VPN adapters.
5. **R5** — Storage budget might not be enough (the `auto_vacuum`/file-shrink-after-pruning gap).
6. **R6** — Drive letter reassignment could become a problem for the `disk` table's identity (keyed on `partition`).
7. **R7** — Developer isn't yet definite about `psutil`'s CPU% behavior (ties to D7) — flagged to verify empirically rather than assume.

Typos fixed on review pass (buget→budget, "name come"→"become", defiante→definite).

**Deliberately not tracked as a Risk, per Yashas's call (5 Sep):** Shrihari-not-yet-confirmed-as-reviewer. Still true and unresolved — just not carried in the Risks list for now.

---

## Design decisions still to make

| # | Decision | Status |
|---|---|---|
| D1 | Language and runtime | ✅ Resolved — Python 3 + Nuitka (see above). Rejected-alternatives writeup for Alternatives Considered still owed |
| D2 | How the recorder starts and stays alive (Task Scheduler vs Service vs startup entry) | ✅ Resolved — Scheduled Task, `ONSTART`, `SYSTEM` account (see above), drafted into DESIGN.md |
| D3 | Storage engine | ✅ Resolved — SQLite, with Postgres/MySQL and flat files rejected (DESIGN.md D16–D20) |
| D4 | Schema shape | ✅ Resolved 30 Aug — full `CREATE TABLE` schema written into DESIGN.md, explicit PKs/FKs on every table, `memory`/`timestamps` folded into `sample`. Still open: gap-marker threshold and clock-change-flag representation (unchanged, see notes above) |
| D5 | How gaps are represented | ✅ Resolved — see highlights above (G14, G15, NG6) |
| D6 | Concurrent access | ✅ Resolved 30 Aug — **G7** via `PRAGMA journal_mode=WAL`, **G8** via one transaction per sample. **G20** (single-instance) resolved earlier via named mutex (Recorder's D6). Mechanisms are written into DESIGN.md's Database section as prose; still need promoting into the Storage D16–D20 table as D21/D22 |
| D7 | Pruning at the size cap | **Ownership decided (5 Sep): the Recorder does the pruning**, not a separate task — not yet written into DESIGN.md. The policy itself (prune early vs. stop recording at 150MB) is still open — Yashas has deliberately deferred it for now |
| D8 | CPU % semantics — what does the first sample report | ✅ Resolved and written into DESIGN.md this session (Recorder's D7) — covers both the recorder-startup priming call and the mid-run new-process case, both shown as `-` for their first sample |
| D9 | Install and uninstall | Partially resolved — G11/G13 (enable/disable behavior), G25 (uninstall removes all data) are decided; mechanism now depends on D2 (resolved). Pruning ownership now assigned to the Recorder (see D7 row). Which component owns the other lifecycle commands (start/stop/status/uninstall) is still unassigned across Recorder/Storage/Display |

---

## Open questions carried from the PRD

- What happens if stored data hits the size budget before 30 days — prune early, or stop recording? *(→ D7, still open)*
- Should the recorder reduce its sampling rate on battery power? *(still open, not discussed yet)*

---

## Reviewers

| Role | Person | Status |
|---|---|---|
| Product | Shreyas | ✅ Reviewed and approved the PRD |
| Senior engineer | Shrihari (per DESIGN.md header) | ⚠️ **Not yet confirmed as reviewing** — needed for DESIGN *before* coding starts |
| Tester | — | Owned in-house (Yashas is an automation test engineer) |
| Pre-review | Claude | Ongoing, before anything goes to a human |

---

## Blockers and lead times

- **Senior engineer not confirmed.** This is the critical path — the design review gate comes before any code, so their availability sets the schedule. Confirm early.
- **New Windows laptop.** Development starts there. Docs need no hardware, so writing continues meanwhile.

---

## Next actions

Picking back up next session (5 Sep progress: Risks section written into DESIGN.md as R1–R7, typos fixed, pruning ownership decided — Recorder does it — but not yet written in, Testing strategy underway conversationally). Priority order:

1. **Write the Testing strategy section into DESIGN.md** — in progress conversationally (test units per component, what to fake/mock, ugly-case list, requirement traceability). Not yet drafted into the doc.
2. **Decide D7 — pruning policy at the size cap.** Ownership is settled (Recorder), but the policy itself — prune early (oldest-first) vs. stop recording if 150MB is hit before 30 days — is still open. Deliberately deferred by Yashas for now; downstream items (3, 4) depend on it.
3. **Decide the `auto_vacuum`/`VACUUM` strategy** — without it, deleting old rows doesn't shrink the `.db` file, which can silently blow the G2 150MB budget even with correct pruning.
4. Write the pruning-ownership decision (Recorder) and D7's eventual policy into DESIGN.md's Storage/Recorder sections once D7 is settled.
5. **Decide the range-command (`last`/`top`) table format** — summary block (peak/average, matching R9) vs. row-per-sample table with a gap marker. Only the single-point (`now`/`at`) layout is settled so far (D28).
6. Decide WAL checkpoint strategy and state the DB file path (parallel to D4's log path).
7. Promote the WAL/per-sample-transaction mechanisms out of Database-section prose into the Storage decision table as D29/D30 (D21–D28 are now taken by Display).
8. Decide and write down the gap-marker threshold (candidate: 45s) and clock-change detection ownership (leaning Display-side) — separate from D25's clock-jump marker, which covers the divergence case, not the "system was off" gap case.
9. Write Open issues section.
10. Send DESIGN for review before writing any code. (Confirming Shrihari as reviewer is deprioritized for now, per Yashas's call — not being tracked as an active blocker.)
11. Create the repo and protect `main`.
