# Glasses — Progress

Last updated: **24 Aug 2026**

## What this project is

A PC resource monitor: a background recorder samples system usage to local storage, and a CLI answers questions about the past — "what was my machine doing at 5:45pm last Tuesday?"

**The real purpose is process practice.** The build is deliberately small so the effort goes into doing it the way a good engineering team would: written docs, real review gates, tests, CI. Domain content (energy / stats / cloud) is deliberately minimal.

**Budget:** ~35 hours total, of which only ~10 is building.

---

## Where we are

| Stage | Artifact | Status |
|---|---|---|
| 1. Define the problem | `PRD.md` | ✅ **Approved** (Shreyas, 9 Aug 2026) |
| 2. Design the solution | `DESIGN.md` | 🔨 In progress — Technical goals & non-goals done (G1–G26, NG1–NG9), Overview drafted with diagram. Detailed Design: "The Recorder" subsection now complete (D1–D15) — boot mechanism, language/runtime, metrics/modules, logging, timestamp storage, single-instance mutex, CPU% priming, sampling-loop timing, per-value and per-write failure handling, battery absence-vs-zero, disk enumeration, network adapter enumeration, first-sample timing. "The Storage" subsection drafted (D16–D20 — SQLite chosen, Postgres/MySQL and flat files rejected). "The Database" section drafted with the actual table list (sample, memory, disk, network_interface, network_io, timestamps, process). 1 gap remains in Recorder (see below); a few schema details and the Display subsection remain open. |
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

### The Storage (D16–D20) and The Database — drafted this session

- **Storage engine (D16–D20):** SQLite. Rejected Postgres/MySQL (needs a running server process — conflicts with G1's CPU/memory budget and the local-only non-goal) and flat files/CSV (no indexed queries, no write atomicity for G8, poor fit for variable-length multi-disk/NIC data).
- **Table list (The Database section):** `sample`, `memory`, `disk`, `network_interface`, `network_io`, `timestamps`, `process` — see DESIGN.md for exact columns.
- **Network per-channel tracking resolved in discussion:** rather than one aggregate `NetworkIO` row per sample, split into `network_interface(id, connection_id)` (a lookup table populated once from D14's WMI enumeration) and `network_io(sample_id, interface_id, bytes_sent, bytes_recv)` (one row per physical interface per sample, values from psutil). The aggregate total is computed at read time (`SUM(...) WHERE sample_id = ?`) rather than stored, so it can't drift out of sync with the per-channel rows. This is written into DESIGN.md's Database section.
- **Not yet written into DESIGN.md** (discussed in conversation, not committed to the doc):
  - Explicit primary keys for `disk` (`sample_id, partition`) and `process` (`sample_id, resource_top_type, rank`) — both are one-to-many per sample and currently have no stated PK.
  - Gap representation: leaning towards no marker row at all — a gap is just the absence of `sample` rows for that period (satisfies R7 without a schema change). Open question: what delta between consecutive `utc_timestamp`s Display should treat as "system was off" vs. normal scheduling jitter from D8 (a candidate number floated: 1.5× the interval, 45s) — not decided, not written anywhere.
  - Clock-change detection (G19): direction floated is a Display-time computation over the three stored timestamps (D5) rather than a flag stored by the Recorder — not decided, not written anywhere.

---

## Design decisions still to make

| # | Decision | Status |
|---|---|---|
| D1 | Language and runtime | ✅ Resolved — Python 3 + Nuitka (see above). Rejected-alternatives writeup for Alternatives Considered still owed |
| D2 | How the recorder starts and stays alive (Task Scheduler vs Service vs startup entry) | ✅ Resolved — Scheduled Task, `ONSTART`, `SYSTEM` account (see above), drafted into DESIGN.md |
| D3 | Storage engine | ✅ Resolved — SQLite, with Postgres/MySQL and flat files rejected (DESIGN.md D16–D20) |
| D4 | Schema shape | Mostly resolved — table list drafted in DESIGN.md's Database section, including the network per-channel split (`network_interface`/`network_io`). Still open: explicit PKs for `disk`/`process`, and how gaps + clock-change flag get represented (see Storage/Database notes above) |
| D5 | How gaps are represented | ✅ Resolved — see highlights above (G14, G15, NG6) |
| D6 | Concurrent access | Partially resolved — G8 (atomic reads) decided. **G20 (single-instance enforcement) ✅ resolved this session** — named mutex, written into DESIGN.md's Recorder table (its D6). **G7 (reads must not block writes) still open** — explicitly scoped to Storage during this session's discussion, not Recorder; needs writing up once Storage's subsection starts |
| D7 | Pruning at the size cap | **Still fully open.** Original PRD open question, never revisited |
| D8 | CPU % semantics — what does the first sample report | ✅ Resolved and written into DESIGN.md this session (Recorder's D7) — covers both the recorder-startup priming call and the mid-run new-process case, both shown as `-` for their first sample |
| D9 | Install and uninstall | Partially resolved — G11/G13 (enable/disable behavior), G25 (uninstall removes all data) are decided; mechanism now depends on D2 (resolved) but pruning ownership (R5, D7) and which component owns lifecycle commands (start/stop/status/uninstall) is still unassigned across Recorder/Storage/Display |

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

1. Write crash/restart resume (G5/G6) into DESIGN.md's Risks section — last undrafted Recorder gap
2. Add explicit primary keys to `disk` and `process` in the Database schema
3. Decide and write down the gap-marker threshold (candidate: 45s) and clock-change detection ownership (leaning Display-side) — currently only discussed, not in DESIGN.md
4. Decide D7 (pruning at size cap) — still fully open
5. Write the Display subsection of Detailed Design
6. Write rejected-alternatives writeup for D1/D2 (Python+Nuitka, Task Scheduler) into Alternatives Considered
7. Write Risks, Testing strategy, Open issues
8. Confirm Shrihari as senior engineer reviewer
9. Send DESIGN for review before writing any code
10. Create the repo and protect `main`
