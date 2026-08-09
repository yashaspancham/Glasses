# How Software Gets Built Properly

A general reference for running a project the way a good engineering team runs one.
Written after doing it once end-to-end, not before.

---

## The core idea

> Every stage produces a written artifact, and somebody other than the author reads it before the next stage starts.

Everything below is detail. If you keep only that sentence, you have most of the value.

Two reasons it works:

1. **Arguing in a document costs hours. Arguing in code costs weeks.** Contradictions, impossible constraints and missing requirements are cheap to find on paper.
2. **You cannot see your own assumptions.** Not through carelessness — they're invisible by definition. Only another reader finds them.

---

## The pipeline

| # | Stage | Artifact | Gate before moving on |
|---|---|---|---|
| 1 | Define the problem | `PRD.md` | Product owner approves |
| 2 | Design the solution | `DESIGN.md` | Senior engineer approves — **before any code** |
| 3 | Break down the work | `PLAN.md` + issues | — |
| 4 | Build | Pull requests | Code review + CI green |
| 5 | Test | Test suite | Runs on every change |
| 6 | Ship | Release / installer | — |
| 7 | Learn | `README.md`, retro | — |

---

## The three documents

### PRD — Product Requirements Document

**Answers:** what are we building, for whom, and why.
**Written by:** product owner (or you wearing that hat).
**Read by:** everyone.

| Section | Contents |
|---|---|
| Header | Author, date, **status** (Draft / In Review / Approved), reviewer |
| Summary | 2–3 sentences. Someone reads only this and understands. |
| Problem | Who hurts and how. Must be true even if your product never existed. |
| User | One concrete person, and what they do today instead. |
| Goals | 3–5 user outcomes. No numbers. |
| Success metrics | Numbers you can actually check. |
| Requirements | Numbered, prioritised Must / Should / Could. |
| In-scope | Short. |
| Non-goals | Usually longer than in-scope, with a reason each. |
| Open questions | Genuine unknowns. |
| Future work | One line per future version. |

**The rule that governs everything:** *if you can't write a test for it, it's not a requirement — it's a wish.*

> ❌ "The tool should be fast and lightweight."
> ✅ "The background process uses under 1% CPU averaged over a minute."

**Never put solution words in a PRD.** No library names, no schemas, no architecture. Naming the solution here quietly steals the decision from the design doc. If a sentence names a technology, it belongs in DESIGN.

**Two pages.** Longer means you started designing.

### DESIGN — Technical Design Document

**Answers:** how will we build it, and what did we reject.
**Written by:** the engineer.
**Read by:** a senior engineer, before implementation starts.

```
1. Context                    — link the PRD, don't repeat it
2. Technical goals/non-goals  — about the system, not the user
3. Overview                   — a few sentences plus a diagram
4. Detailed design            — components, data model, key flows
5. Alternatives considered    — the heart of the document
6. Risks                      — what could make this wrong, and how you'd find out early
7. Testing strategy
8. Open issues
```

**A design doc is a record of decisions.** For each significant choice: what you picked, what you rejected, why. *A decision with no rejected alternative usually means you didn't consider any* — and that is the first thing a good reviewer looks for.

**Design for the change you're confident is coming, not every change you can imagine.** Building for an imagined future is how side projects die. A future plan belongs in this doc only if it changes what you build now.

### PLAN — Implementation Plan

**Answers:** in what order, and how do I know when I'm done.

```markdown
## v1 — <name>          ← ONLY THIS GETS DETAIL
Goal: ...
Done when: ...

- [ ] #1  Repo skeleton, CI, lint
- [ ] #2  ...

## v2 — <name>          One line.
## v3 — <name>          One line.
## Later / unsorted
```

**Only the current version gets detail.** A task list for v3 is fiction — you'll know ten things by then that make it wrong. Plan the near thing precisely, the far thing vaguely.

Mirror the versions as milestones in the issue tracker. That's what stops scope creep: not discipline, but having somewhere else to put the idea.

---

## Reviews

The reviewers matter more than the documents. A document nobody read is a diary.

| Artifact | Reviewer | Where |
|---|---|---|
| PRD | Product owner | Google Docs (comments, no git needed) |
| DESIGN | Senior engineer | Pull request |
| Code | Senior engineer | Pull request |
| Tests | Same standard as code | Pull request |

**How to ask so you actually get a review.** Vague asks get "looks good", which is worthless. Send a deadline, a time estimate, and 2–3 specific questions:

> "PRD, 2 pages, 20 minutes, by Friday. Specifically: (1) is the problem believable to a real user? (2) are the metrics measurable or hand-wavy? (3) what's in scope that should be a non-goal?"

**Batch your asks.** You get two or three rounds of goodwill from any volunteer. Don't spend one on a variable name.

**Send at ~80%.** Rough edges give the reviewer something to do. A document that arrives perfect got no value from review and usually means days lost polishing alone.

**Close the loop in writing.** A comment gets a written response and a visible change to the document — a new requirement, a new non-goal, a line in Future work. Deciding in your head doesn't count; the reviewer must see their comment land.

---

## The build loop

```
issue → branch → small commits → self-review your own diff →
open PR → CI runs → reviewer comments → revise → merge → close issue
```

- **No commits to `main`.** Ever. Use branch protection so it isn't a matter of willpower.
- **Small PRs** — under ~300 lines. If it's bigger, slice the work.
- **Every PR closes a numbered issue.** `Closes #12`.
- **Self-review before assigning.** Read your own diff in the PR view; it catches about half your mistakes and costs five minutes.
- **CI green before merge**, enforced by the platform, not by you remembering.
- **Commit messages a stranger can follow.**

Your git history is a portfolio artifact in its own right. Someone scrolling it should see a person who works the way their team works.

---

## Testing

- Tests ship **with** the change, not in a cleanup phase afterwards.
- Test code is reviewed to the same standard as source code.
- Fake the slow and the external — hardware, clocks, network — so the suite is deterministic and finishes in seconds.
- Test the ugly cases, not the happy path: empty input, one item, a gap in the middle, a request for data that doesn't exist, permission denied.
- A "definition of done" written in PLAN.md and applied honestly.

---

## Durable principles

**Separate what from how.** The PRD is the problem space, the design doc is the solution space. Mixing them means design decisions get made by accident.

**Non-goals are load-bearing.** What you deliberately won't do is as much a decision as what you will.

**Do the long-lead item first.** Find the thing bounded by wall-clock rather than effort — data collection, a reviewer's availability, hardware arriving — and start it while you write the docs.

**Constraints multiply; check the arithmetic.** Three plausible requirements can be jointly impossible. Multiply them out on paper before you build.

**Prefer a boring build done properly** over an ambitious one done loosely. If you run out of time, cut features — never the tests or the process.

**Decisions leave a trace.** Six months later nobody asks what the code does; they ask why it's like that. The design doc and PR history are the answer.

---

## Working solo

You can't have a standing team, so substitute:

| Role | Substitute |
|---|---|
| Product owner | One friend, one round on the PRD |
| Senior engineer | One experienced dev, on DESIGN *and* code |
| QA | You |
| Cheap unlimited iteration | An LLM, before any human sees it |

Use the LLM as the **pre-review** — it raises the floor so your scarce human reviewers spend their attention on the ceiling. But know its limits: it doesn't know whether the problem is real, it's agreeable unless told to attack, and it has no taste earned from maintaining things. Take a human's "this is wrong" more seriously than a model's "looks good".

**Don't let it write the artifacts.** Reviewed work has a texture that generated work doesn't, and the project stops being evidence of anything.

---

## Starting checklist

- [ ] Confirm reviewers **first** — their availability sets your timeline
- [ ] Create the repo, protect `main`
- [ ] Start any long-lead item now
- [ ] Write the PRD → review → approve
- [ ] Write DESIGN → review → approve
- [ ] Write PLAN, create issues and milestones
- [ ] Set up CI before the first feature
- [ ] Build, one small PR at a time
- [ ] README with a screenshot and three-command setup
- [ ] Write up what you learned

---

## Common failure modes

| Symptom | What's actually happening |
|---|---|
| Beautiful docs, half-finished code | Overscoped. Cut features, keep the process. |
| "Looks good 👍" from every reviewer | Asks were vague. Send specific questions. |
| Scope keeps growing | No milestone to park ideas in. |
| Can't explain why the code is like that | Design decisions were never written down. |
| Rebuilt the same thing three times | Started coding before the design was settled. |
| Endless polishing before showing anyone | Perfection used to avoid review. Send at 80%. |
| Weighing a fifth alternative project | The process only starts once you commit. |
