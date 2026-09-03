# Requirements Management — the Gap Between What's Built and What's Tracked

**Terms used throughout:** [`Glossary.md`](./Glossary.md).

**ADM position:** not a phase — per `../TOGAF/README.md`, this is the continuous activity sitting at the center of the cycle, referenced by every other phase, living in the Architecture Repository. This document is that repository's first real entry, because — as of this finding — it didn't have one. Everything produced by this initiative so far (Phases A/B, the Zachman matrix) was assembled by reading code and tracing bugs, not by consulting a standing set of tracked requirements. That absence is itself the finding.

---

## 1. The finding, stated plainly

There is a gap between what is actually being developed on the New KYC rebuild and what is communicated or tracked as a requirement for it. Specific symptoms:

- Requirements for the rebuild are not properly tracked at the project level.
- The overall system architecture is communicated vaguely, not as a defined, referenceable target.
- For a **rebuild** — the highest-stakes kind of initiative, where getting the target architecture wrong is expensive to unwind — there is no gap analysis and no roadmap describing the overall system's architecture.
- Requirements that should have been captured at the start of the rebuild were not.

## 2. This finding is already visible inside this initiative's own material — it didn't need new investigation to surface

The clearest evidence isn't hypothetical; it's the shape of the documents already produced:

- **`02-Phase-A-Architecture-Vision.md §1`** already recorded: *"No formal request exists... there is no signed Statement of Architecture Work."* That was noted as a scope caveat at the time — read against this finding, it's the same gap, just observed one level down.
- **Every architectural finding in [`01_Planninng/IDR-Rebuild/`](../../../01_Planninng/IDR-Rebuild/) was produced reactively** — triggered by a live bug (the Date of Birth echo loop, the tax-residence duplication, the duplicate-`Visa` race condition) or by a direct question asked mid-session, never by consulting a pre-existing requirement that said "sync must guarantee X" or "grouped answers must reconcile Y way." The fixes were correct, but nothing upstream of the bug predicted the need for them.
- **WI 24788 and WI 24853** (the two Azure DevOps tickets grounding the Phase 0 lookup-service work) are both scoped as narrow technical tasks ("separate the Kyc and RulesEngine modules," "re-architect the lookup service") with no visible trace back to an originating business requirement or architecture principle. They read as *"this got messy, fix it"* tickets, not *"requirement REQ-00X demands this"* tickets — because no such requirement existed to trace back to.
- **The Fund Manager attribution question** (`03-Phase-B-...md §4`) is a requirement that was never asked in the first place: nobody appears to have documented, at the point fund-manager-facing features were built, whether "Fund Manager" was meant to be a real external actor or an internal-staff capability. That's not a design decision that got made and later needs re-litigating — it's a requirement that was never captured as one.

## 3. What should have existed from the start: a Requirements Catalog

TOGAF's own artifact for this is the **Requirements Catalog** — a flat, traceable list: each requirement has an id, a statement, a source (which stakeholder/driver asked for it), a priority, a status, and a link forward to whatever architecture principle or work package addresses it. None of that existed for this rebuild. Reconstructing it retroactively, from what this session already confirmed, makes the gap concrete rather than abstract:

| ID | Requirement | Source | Traces to | Status |
|---|---|---|---|---|
| REQ-001 | `RulesEngine` must not depend on `Kyc`'s concrete `LookupService` | WI 24788 | Principle 2 (`01-Preliminary-...md`) | Closed — Phase 0, this session |
| REQ-002 | Lookup resolution must handle unmapped types gracefully, not throw | WI 24853 acceptance criteria | — | Closed — this session |
| REQ-003 | Inbound DD Questionnaire sync must reconcile grouped answers (add + remove), not blind-append | Rebuild/Legacy team bug report | Principle 3 (`01-...md`) | Partially closed — `SubmitAnswersHandler` fixed this session; `SyncExchange`'s GroupId-minting bug and the Rebuild→Legacy partial-payload bug (`10-Verified-Store-Proposal.md §6`) remain open |
| REQ-004 | IKYC and MKYC must share one confirmed KYC-subject identity record | Inferred from `ManagedServices.LinkedProfile` being a stub — no originating ticket found | — | Not started. **No owner identified.** |
| REQ-005 | Fund Manager's access model (self-service vs. staff-performed) must be confirmed before further Fund/TransferAgency feature work is attributed to that journey | Inferred from `02-Journey-Scoping.md §6` | — | Blocked — needs live permission data, not further code investigation |
| REQ-006 | Answer/Visa write invariants must be backed by database constraints, not application logic alone | Duplicate-answer defect (doc 08) + duplicate-Visa race (doc 12 §3a) | Principle 4 (`01-...md`) | Not started |
| REQ-007 | `TemplateAPI` must have an Onboarding module covering staff-driven bulk import + invitation, matching IDR's `OnboardingSessionsController`/108 `OnboardingImportCdd*Service` coverage | Phase C gap analysis (`04-Phase-C-...md §5`) — no originating ticket found | — | Not started. **No owner identified.** |
| REQ-008 | `TemplateAPI` must have a Compliance module covering FATCA/CRS classification, screening, DD scheduling, W-Forms | Phase C gap analysis (`04-Phase-C-...md §5`) — no originating ticket found | — | Not started. **No owner identified.** |
| REQ-009 | `TemplateAPI` must have a Reporting module covering SSRS replacement, exports, FATCA/CRS filing output | Phase C gap analysis (`04-Phase-C-...md §5`) — no originating ticket found | — | Not started. **No owner identified.** |
| REQ-010 | Search technology for the target platform must be decided (SQL full-text vs. Elastic) — currently "under evaluation" with no decision date | Phase D gap analysis (`05-Phase-D-...md §4`) | — | **Open decision, no owner or deadline identified** |
| REQ-011 | Reporting/BI technology must be decided (SSRS vs. Power BI Embedded / FastReport) before `S1.Module.Reporting` (REQ-009) can be scoped | Phase D gap analysis (`05-Phase-D-...md §4`) | Blocks REQ-009 | **Open decision, no owner or deadline identified** |

**Every row above except REQ-001/002 was reconstructed after the fact, this session, from bug traces and code reads — not from a document that stated the requirement before the work started.** That's the finding, made concrete: this table should have existed *before* WI 24788 was ever written, not been assembled retroactively by an EA initiative months later.

## 4. Why this sharpens Phase E/F specifically

Phase E (Opportunities & Solutions) exists precisely to consolidate gap analyses into a real, prioritized set of work packages, and Phase F to turn that into a costed, sequenced roadmap. That is *exactly* the artifact this finding says is missing for the New KYC rebuild overall. Two consequences for how this initiative proceeds:

1. **Phase E/F's deliverable is no longer a formality to complete once B/C/D are done — it's the direct answer to this finding.** When it's written, its first job is producing the thing that doesn't currently exist anywhere: one document that states the New KYC rebuild's target architecture, the gaps against it, and a real roadmap — not scattered across bug-fix tickets and reactive investigation docs.
2. **Requirements Management can't be a one-time catch-up exercise.** The table in §3 is a snapshot, not a solution — if this initiative (or the real rebuild effort) stops maintaining it the moment this document is finished, the exact gap being described here reappears within weeks. The minimal viable fix, starting now: every new architectural finding from this point forward — in this initiative or in the real codebase work it's grounded in — gets a row in this table with a source and a status, before it becomes a fix, not after.

## 5. Immediate, concrete recommendation

Don't wait for Phase C/D to finish before acting on this. Two things worth doing in parallel with the rest of the ADM sequence:

- **Keep §3's table alive as this initiative's actual Requirements Repository** — add a row every time a new gap surfaces (the way REQ-004/005/006 were just added), rather than letting findings live only in prose inside whichever phase document happened to surface them.
- **When Phase E/F is written, open it by explicitly answering this finding** — state the New KYC rebuild's target architecture in one place, the gaps against it (pulling from Phase B §7 and whatever Phase C/D produce), and a real roadmap — and cross-reference back to this document as the reason that phase was scoped that way, not just "next in the ADM sequence."

---

Cross-references: `02-Phase-A-Architecture-Vision.md §5` ("what this phase does not yet answer") should be read alongside this document — Phase A already flagged the absence of a formal charter; this document generalizes that into the full requirements-tracking gap across the rebuild.
