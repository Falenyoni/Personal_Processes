# Phase G - Implementation Governance

**Terms used throughout:** [`Glossary.md`](./Glossary.md).

**ADM focus:** oversee implementation projects to ensure what gets built stays conformant with the architecture. Outputs: Architecture Contracts, compliance assessments against them.

**Status, stated plainly per `01-Preliminary-Governance-And-Principles.md §2`:** aspirational, not active. There is no chartered implementation program for this initiative to govern - no PR gate, no architecture review board, no enforcement mechanism. What follows is the compliance checklist that *would* apply if the work packages in `06-Phase-E-F-...md` were being executed under real governance, built now so it exists before it's needed rather than invented retroactively per work package.

---

## 1. The compliance checklist

One row per Architecture Principle (`01-Preliminary-...md §3`), phrased as a yes/no question a reviewer would ask against any change touching that area.

| Principle | Compliance question |
|---|---|
| 1 - Single Writer Per Field | Does this change introduce a field written by more than one system? If yes, is the conflict-resolution rule explicit, not just "we'll sync"? |
| 2 - No Direct Cross-Module Database Coupling | Does this change add an EF/DB reference from one module to another module's concrete types? If yes, reject - route through a contract-mediated interface instead. |
| 3 - Reconciliation Over Blind Append | Does this change accept a full-state resend of a repeatable/grouped structure? If yes, does it reconcile (add + remove) rather than append-only? |
| 4 - Database Constraints Back Application-Level Invariants | Does this change introduce a uniqueness/ownership invariant enforced only in application code? If yes, is there a matching database constraint, or a documented reason there can't be one? |

## 2. Worked example - retroactively applying this checklist to work already done this session

Since no real governance process was active, this is the closest available substitute: checking whether already-completed work *would* have passed, after the fact.

**The `AnswerReconciliationService` consolidation** (three duplicated handlers merged into one shared service, `SubmitAnswersHandler`'s missing delete-on-removal step fixed):

- Principle 2: **Pass.** The consolidation moved logic *into* a shared service within the same module (`S1.Module.Kyc.Services`), not across a module boundary - no new cross-module DB coupling introduced.
- Principle 3: **Pass, and directly on-point.** This work *is* the reconciliation-over-blind-append fix - `GetGroupedAnswersToDelete` reconciles rather than appends.
- Principle 1: **Not applicable** - this fix operates within one system's own write path, not across a sync boundary.
- Principle 4: **Fail, not yet addressed.** The fix corrects the *application-level* matching logic but does not add a database constraint backing `(ProfileIdRef, QuestionIdRef, FundIdRef, GroupId)` uniqueness on `Kyc.Answer`. This is exactly REQ-006 / WP4 in `06-Phase-E-F-...md` - the checklist correctly flags it as still open, which is the checklist doing its job, not a criticism of the fix itself (WP4 was always scoped as separate, later work).

This is the value a real Phase G would add: not blocking the fix, but making the Principle 4 gap visible *at the time of the change* instead of only surfacing it later in a separate gap-analysis pass, as actually happened here.

## 3. What would need to exist for this to become real governance

Named explicitly so the gap between "aspirational" and "active" stays concrete, not vague:

- A place these checks actually run - a PR template checklist, a CI lint rule, or a human review step - none of which currently exists for this codebase under this initiative's authority.
- An Architecture Contract per work package in `06-Phase-E-F-...md` - a short, signed-off statement that WP1-WP8's implementation will comply with the four principles above, checked at completion.
- Someone with actual authority to block a merge on a Principle failure - which requires the charter this initiative has been explicit about not having (`01-Preliminary-...md §1`).

None of the above exists today. This document is the checklist waiting for that authority to exist, not a claim that it does.
