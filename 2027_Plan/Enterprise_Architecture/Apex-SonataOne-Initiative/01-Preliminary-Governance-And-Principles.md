# Preliminary Phase — Governance and Architecture Principles

**Terms used throughout:** [`Glossary.md`](./Glossary.md).

**ADM focus:** establish the EA capability itself, before touching any specific architecture. Per `../TOGAF/README.md`, this phase's outputs are: a tailored ADM, Architecture Principles, a governance framework, and the scope of what's affected.

---

## 1. Scope of this initiative

This is a **personal EA initiative**, not (yet) a chartered organizational one — there is no formal sponsor, no signed Statement of Architecture Work. That's an honest constraint worth stating up front rather than pretending otherwise: the ADM is being walked as a learning-and-practice exercise, using real data, at real depth, but without organizational authority behind it yet.

**In scope:** the Investor Services / KYC platform and its directly connected systems — `IDR`, `TemplateAPI` (all modules), `ManagedServices`, `Sync`, `SyncExchange`, `SonataOneSecurity`, `Identity`. This is the area with by far the deepest existing investigation (the whole `01_Planninng\IDR-Rebuild` body of work).

**Out of scope, for now:** `ApiGateway`, `Notification`, `PublicAPI`, `Subscription`, `TransferAgency`, `Models`, `Roboframework`, `configuration` — these exist in the landscape (confirmed via directory listing) but haven't been investigated at the level of depth needed to say anything grounded about them. Listed in the Application Portfolio Catalog (`00-Diagram-And-Artifact-Guide.md`) as known-but-unexamined, not silently omitted.

## 2. Tailoring the ADM

Two adjustments to the standard cycle, both because this is a solo, non-chartered initiative rather than a resourced program:

- **Phases run sequentially over an extended, unscheduled timeframe** rather than each phase being a scheduled workshop with stakeholders. Each phase document is dated and versioned informally instead.
- **Governance (Phase G) is aspirational, not active** — there's no implementation happening under this initiative's authority to govern yet. Phase G's document, when it's written, will describe what governance *would* look like if this were chartered, not enforce anything today.

Everything else — the phase sequence, the artifact types, Requirements Management running continuously — follows the standard ADM as documented in `../TOGAF/README.md`.

## 3. Draft Architecture Principles

TOGAF principles have a standard shape: **Name, Statement, Rationale, Implications**. Drafted here from patterns *already observed* in this session's investigation — not aspirational best practices pasted in from a textbook, but principles that would have prevented specific, real, already-confirmed problems.

### Principle 1 — Single Writer Per Field

**Statement:** For any given data field shared between two or more systems, exactly one system owns the write at any point in time. All other systems hold it read-only, refreshed by one-directional projection.

**Rationale:** The confirmed Date of Birth echo-loop bug and the tax-residence duplication bug both trace to the same root cause — bidirectional writes with no ownership rule (`08-Sync-Strategy-ETL-Vs-Unidirectional-Cutover.md §1-2`). This isn't a hypothetical risk; it's the diagnosed cause of two independent, already-shipped defects.

**Implications:** Every new cross-system data flow proposal must state, explicitly, which system owns the write. "Both can write, we'll sync" is not an acceptable answer without also specifying the conflict-resolution rule (and per the evidence gathered, the honest answer is usually that no such rule can be made reliable without ownership).

### Principle 2 — No Direct Cross-Module Database Coupling

**Statement:** Modules within the modular monolith communicate via contract-mediated interfaces (shared DTOs + internal service adapters), never via direct EF/DB references to another module's concrete types.

**Rationale:** `ADR-001-Contract-Mediated-Module-Communication.md` documents this as the intended design, and this session's Phase 0 work (the lookup-service decoupling) both confirmed real violations existed (RulesEngine depending on Kyc's concrete `LookupService`) and fixed one of them.

**Implications:** New module boundaries get reviewed against this before merge, not after. WI 24788's remaining "deepest coupling" item (`GetQuestionFieldsHandler` reading Kyc's `Question` entity directly) is a known, tracked violation of this principle, not yet resolved.

### Principle 3 — Reconciliation Over Blind Append

**Statement:** Any handler accepting a full-state resend of a repeatable/grouped data structure must reconcile against current state (insert new, remove absent) rather than blindly appending.

**Rationale:** Directly derived from the `GroupedAnswersToDelete` gap found and fixed in `SubmitAnswersHandler` this session, and the still-open `SyncExchange` GroupId-minting bug (`10-Verified-Store-Proposal.md §6`).

**Implications:** Any new sync/import endpoint accepting a collection gets a reconciliation review as a matter of course, not only after a duplication bug is reported.

### Principle 4 — Database Constraints Back Application-Level Invariants

**Statement:** Where application code enforces a uniqueness or ownership invariant (e.g. "one answer per question+fund+group"), a matching database constraint should exist wherever practical, not just application logic.

**Rationale:** `08-Sync-Strategy...md §3` traced a duplicate-answer defect directly to `Kyc.Answer` having no unique constraint on `(ProfileIdRef, QuestionIdRef, FundIdRef, GroupId)` — application-level matching failed silently, and the database had nothing to reject the bad write with. The duplicate-`Visa` race condition found in `12-Profile-Visa-...md §3a` is the same pattern again, in a different table.

**Implications:** Schema review for new tables explicitly asks "what invariant is this supposed to hold, and is it backed by a constraint" — not left as an implicit assumption in a service class.

**Status:** draft, unreviewed by anyone but the author. The value of writing these down now, even informally, is that Phase B onward should actively test whether they hold up against a *different* part of the platform — if Business/Data Architecture work later contradicts one of these, that's a finding worth having, not an inconsistency to quietly paper over.
