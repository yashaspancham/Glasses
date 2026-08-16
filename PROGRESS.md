# Glasses — Progress

Last updated: **16 Aug 2026**

## What this project is

A PC resource monitor: a background recorder samples system usage to local storage, and a CLI answers questions about the past — "what was my machine doing at 5:45pm last Tuesday?"

**The real purpose is process practice.** The build is deliberately small so the effort goes into doing it the way a good engineering team would: written docs, real review gates, tests, CI. Domain content (energy / stats / cloud) is deliberately minimal.

**Budget:** ~35 hours total, of which only ~10 is building.

---

## Where we are

| Stage | Artifact | Status |
|---|---|---|
| 1. Define the problem | `PRD.md` | ✅ **Approved** (Shreyas, 9 Aug 2026) |
| 2. Design the solution | `DESIGN.md` | 🔨 In progress — Technical goals & non-goals done (G1–G26, NG1–NG9), Overview drafted with diagram. Detailed design not started. |
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

**Detailed design, Alternatives considered, Risks, Testing strategy, Open issues — not started.**

---

## Design decisions still to make

| # | Decision | Status |
|---|---|---|
| D1 | Language and runtime | Python assumed throughout goals discussion; needs formal writeup with rejected alternatives |
| D2 | How the recorder starts and stays alive (Task Scheduler vs Service vs startup entry) | **Still fully open.** Foundational — G11, G20, and G21 all imply a system-level mechanism, but the actual choice hasn't been made |
| D3 | Storage engine | SQLite assumed throughout goals discussion; needs formal writeup with rejected alternatives |
| D4 | Schema shape | Most of the hard calls are made (see highlights above) — needs to be written up as an actual schema, including the variable-length shape needed for multi-disk and multi-network data |
| D5 | How gaps are represented | ✅ Resolved — see highlights above (G14, G15, NG6) |
| D6 | Concurrent access | ✅ Resolved — G7, G8 (read/write non-blocking, atomic reads), G20 (single-instance enforcement) |
| D7 | Pruning at the size cap | **Still fully open.** Original PRD open question, never revisited during the goals work |
| D8 | CPU % semantics — what does the first sample report | ✅ Resolved in discussion (priming call, see highlights above) — **not yet written into DESIGN.md**, needs to land as a goal or in Detailed Design |
| D9 | Install and uninstall | Partially resolved — G11/G13 (enable/disable behavior), G25 (uninstall removes all data) are decided; the underlying mechanism depends on D2 |

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

1. Decide D2 (recorder lifecycle mechanism) and D7 (pruning at size cap) — both still fully open
2. Write the CPU% priming decision (D8) into DESIGN.md — currently only decided in conversation, not on paper
3. Write Detailed Design: formalize D1/D3 (Python, SQLite) with rejected alternatives, the D4 schema, and key flows (enable/disable, sampling loop, query path)
4. Write Alternatives considered, Risks, Testing strategy, Open issues
5. Confirm Shrihari as senior engineer reviewer
6. Send DESIGN for review before writing any code
7. Create the repo and protect `main`
