# Glasses — Progress

Last updated: **23 Aug 2026**

## What this project is

A PC resource monitor: a background recorder samples system usage to local storage, and a CLI answers questions about the past — "what was my machine doing at 5:45pm last Tuesday?"

**The real purpose is process practice.** The build is deliberately small so the effort goes into doing it the way a good engineering team would: written docs, real review gates, tests, CI. Domain content (energy / stats / cloud) is deliberately minimal.

**Budget:** ~35 hours total, of which only ~10 is building.

---

## Where we are

| Stage | Artifact | Status |
|---|---|---|
| 1. Define the problem | `PRD.md` | ✅ **Approved** (Shreyas, 9 Aug 2026) |
| 2. Design the solution | `DESIGN.md` | 🔨 In progress — Technical goals & non-goals done (G1–G26, NG1–NG9), Overview drafted with diagram. Detailed Design: "The Recorder" subsection now written as a local decision table (D1–D12), covering boot mechanism, language/runtime, metrics/modules, logging, clock-change detection, single-instance mutex, CPU% priming, sampling-loop timing, per-value and per-write failure handling, and battery absence-vs-zero. 3 gaps remain in Recorder (see below). Storage and Display subsections not started. |
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

**Detailed design ("The Recorder") — largely done, see below. Storage, Display, Alternatives considered, Risks, Testing strategy, Open issues — not started.**

### The Recorder — now a local decision table (D1–D12, numbered within that section only, distinct from this file's D1–D9 tracking list below)

- **D1** Boot mechanism: Windows Scheduled Task `GlassesRecorder`, registered once at install (`glasses start`) via `schtasks /Create`, trigger `ONSTART`, account `SYSTEM`
- **D2** Language/runtime: Python 3, compiled to a standalone `.exe` via Nuitka (rejected-alternatives writeup for Alternatives Considered still owed)
- **D3** Metrics & modules: CPU/memory/disk/network/battery/top-10 processes via `psutil` and `wmi`; also `logging`, `sqlite3`, `datetime`
- **D4** Logging: `C:\ProgramData\Glasses\logs\glasses_recorder.log`, 7-day retention (the sentence on whether logs count against the 150MB storage budget is still grammatically ambiguous — needs a clean rewrite)
- **D5** Clock-change detection: compare UTC vs. monotonic elapsed time between samples
- **D6** Single-instance enforcement: resolved this session — named mutex `Global\GlassesRecorderMutex`; second instance sees `ERROR_ALREADY_EXISTS`, logs, exits. (The user-facing "you're already running" message is Display's job, not Recorder's — deferred to when Display gets written.)
- **D7** CPU% priming: resolved this session — one throwaway `cpu_percent(interval=None)` call at recorder startup (system-wide); any PID seen for the first time is individually primed and shown as `-` for that one sample, not a misleading `0%`
- **D8** Sampling loop: `sleep_time + record_time = 30s` (sleep is computed as the remainder after the work, not a hardcoded number) — resolves the drift a naive `sleep(30)` would have
- **D9–D11** Failure handling: a failed per-value read logs and shows `-` for that field; a failed write logs and the recorder continues; both keep one bad reading from taking down the whole sample or the process
- **D12** Battery: `psutil.sensors_battery()` — `None` → empty cell, present-but-drained → literal `0%`

**Gaps still open in Recorder, unaddressed as of this session:**
1. **Crash/restart resume (G5, G6).** ✅ **Decided (2026-08-23): accepted as a known v1 risk, not engineered around.** If the recorder crash-loops past Task Scheduler's retry limit, it stays down until next reboot — considered acceptable given how often Windows machines reboot anyway (update cycles, sleep issues) and that this is a monitoring tool, not critical infrastructure. Options considered and rejected for v1: a second time-based Task Scheduler trigger as a self-healing backstop (cheap, builds on the D6 mutex, but adds a moving part); switching from Scheduled Task to a Windows Service with SCM recovery policy (more robust, no hard retry ceiling, but reopens the D1 boot-mechanism decision). **Still needs to be written into DESIGN.md's Risks section** (what the risk is, why accepted, what would trigger revisiting it) — not yet drafted.
2. **Disk enumeration logic (G22, NG8).** ✅ **Decided (2026-08-23): use `psutil.disk_partitions(all=False)` for v1**, which on Windows is backed by `GetDriveType()`'s `DRIVE_FIXED`/`DRIVE_REMOVABLE` split. This also solves the physical-disk-to-drive-letter mapping for free (psutil returns drive letters directly). Known limitation, accepted rather than engineered around: an internal SD-card-reader slot can report as removable, and some external docked/eSATA drives can report as fixed — occasional misclassification is an accepted v1 tradeoff. Rejected alternative: `MSFT_PhysicalDisk.BusType` (a different WMI namespace, `root\Microsoft\Windows\Storage`) — more accurate bus-type classification, but doesn't by itself map physical disks to drive letters (would need `Win32_DiskDrive`'s associator classes in `root\cimv2` to do that cleanly) — more correct but more implementation work than v1 warrants. **Still needs to be written into DESIGN.md's Recorder decision table** — not yet drafted.
3. **Network interface enumeration logic (G26, NG9).** ✅ **Decided (2026-08-23): enumerate `Win32_NetworkAdapter` once at recorder startup (cached, not re-queried each sample), filter to `PhysicalAdapter == True`, and build a set of `NetConnectionID` values from that filtered list.** Each sample, match that set against `psutil.net_io_counters(pernic=True)`'s per-NIC keys (join on `NetConnectionID`, not WMI's own `Name`, which is a verbose driver string that won't match psutil's interface names). Known limitation, accepted rather than engineered around: `PhysicalAdapter` occasionally misclassifies adapters created by VPN clients, Hyper-V virtual switches, or some vendor NIC-teaming drivers — same class of tradeoff as the disk `GetDriveType` approximation. No simpler psutil-only shortcut exists (unlike disks) — psutil has no built-in physical/virtual distinction for NICs, so WMI is required here. **Still needs to be written into DESIGN.md's Recorder decision table** — not yet drafted.

---

## Design decisions still to make

| # | Decision | Status |
|---|---|---|
| D1 | Language and runtime | ✅ Resolved — Python 3 + Nuitka (see above). Rejected-alternatives writeup for Alternatives Considered still owed |
| D2 | How the recorder starts and stays alive (Task Scheduler vs Service vs startup entry) | ✅ Resolved — Scheduled Task, `ONSTART`, `SYSTEM` account (see above), drafted into DESIGN.md |
| D3 | Storage engine | SQLite assumed throughout goals discussion; needs formal writeup with rejected alternatives |
| D4 | Schema shape | Most of the hard calls are made (see highlights above) — needs to be written up as an actual schema, including the variable-length shape needed for multi-disk and multi-network data |
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

1. Finish Recorder's 3 remaining gaps: crash/restart resume (G5/G6), disk enumeration logic (G22/NG8), network enumeration logic (G26/NG9) — directions discussed this session, not yet written into DESIGN.md (slots D13+ reserved in the table)
2. Fix the ambiguous "logs will the 150MB budget" sentence in Recorder's D4
3. Decide D7 (pruning at size cap) — still fully open
4. Write Storage and Display subsections of Detailed Design, including D3/D4 (SQLite writeup with rejected alternatives, and the actual schema)
5. Write rejected-alternatives writeup for D1/D2 (Python+Nuitka, Task Scheduler) into Alternatives Considered
6. Write Risks, Testing strategy, Open issues
7. Confirm Shrihari as senior engineer reviewer
8. Send DESIGN for review before writing any code
9. Create the repo and protect `main`
