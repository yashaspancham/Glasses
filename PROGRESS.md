# Glasses — Progress

Last updated: **9 Aug 2026**

## What this project is

A PC resource monitor: a background recorder samples system usage to local storage, and a CLI answers questions about the past — "what was my machine doing at 5:45pm last Tuesday?"

**The real purpose is process practice.** The build is deliberately small so the effort goes into doing it the way a good engineering team would: written docs, real review gates, tests, CI. Domain content (energy / stats / cloud) is deliberately minimal.

**Budget:** ~35 hours total, of which only ~10 is building.

---

## Where we are

| Stage | Artifact | Status |
|---|---|---|
| 1. Define the problem | `PRD.md` | ✅ **Approved** (Shreyas, 9 Aug 2026) |
| 2. Design the solution | `DESIGN.md` | 🔨 In progress — outline agreed, D1 next |
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

## Design decisions still to make

To be worked through in `DESIGN.md`, each with rejected alternatives and reasoning:

- **D1 — Language and runtime.** Start here; D2, D3 and D9 all hang off it.
- **D2 — How the recorder starts and stays alive.** Task Scheduler vs Windows Service vs startup entry.
- **D3 — Storage engine.** Embedded DB vs flat files.
- **D4 — Schema shape.** Drives the 150MB budget directly.
- **D5 — How gaps are represented.** R7: "no data" must not look like a real zero.
- **D6 — Concurrent access.** Recorder writes while the CLI reads.
- **D7 — Pruning at the size cap.** Carried over from the PRD as an open question.
- **D8 — CPU % semantics.** Measured between two points; what does the first sample report?
- **D9 — Install and uninstall.** R15 and R18 want single commands.

---

## Open questions carried from the PRD

- What happens if stored data hits the size budget before 30 days — prune early, or stop recording? *(→ D7)*
- Should the recorder reduce its sampling rate on battery power?

---

## Reviewers

| Role | Person | Status |
|---|---|---|
| Product | Shreyas | ✅ Reviewed and approved the PRD |
| Senior engineer | TBD | ⚠️ **Not yet confirmed** — needed for DESIGN *before* coding starts |
| Tester | — | Owned in-house (Yashas is an automation test engineer) |
| Pre-review | Claude | Ongoing, before anything goes to a human |

---

## Blockers and lead times

- **Senior engineer not confirmed.** This is the critical path — the design review gate comes before any code, so their availability sets the schedule. Confirm early.
- **New Windows laptop.** Development starts there. Docs need no hardware, so writing continues meanwhile.

---

## Next actions

1. Confirm the senior engineer reviewer
2. Write `DESIGN.md`, starting with D1
3. Send DESIGN for review before writing any code
4. Create the repo and protect `main`
