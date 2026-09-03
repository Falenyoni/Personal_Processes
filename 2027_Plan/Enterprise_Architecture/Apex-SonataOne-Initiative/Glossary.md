# Glossary

Single source for every term used across this initiative's documents — linked from each, not repeated in each.

## TOGAF terms

| Term | Meaning |
|---|---|
| **ADM** | Architecture Development Method — TOGAF's core cyclical process, phases Preliminary through H. |
| **Architecture Vision** | Phase A's deliverable — high-level scope, stakeholders, and target state. |
| **Business/Data/Application/Technology Architecture** | The four domains the ADM works through in Phases B, C (split into two), and D. |
| **ABB / SBB** | Architecture Building Block (abstract/logical, e.g. "a patient record capability") / Solution Building Block (its concrete implementation, e.g. a specific module). |
| **Enterprise Continuum** | TOGAF's classification of artifacts from generic/foundational to organization-specific, split into the Architecture Continuum and Solutions Continuum. |
| **Architecture Repository** | Where everything the ADM produces is stored — conceptually a wiki/tool/folder structure; this initiative's repository is this folder. |
| **Architecture Principle** | A named statement + rationale + implications, constraining future architecture decisions. See `01-Preliminary-Governance-And-Principles.md`. |
| **Statement of Architecture Work** | Phase A's formal, sponsor-approved scoping document. Not chartered for this initiative — see `01-...md §1`. |
| **Transition Architecture** | An intermediate state between baseline and target, used when the move happens in stages. |
| **Architecture Contract** | Phase G's formal agreement that a delivery project will comply with the architecture. |
| **Requirements Management** | The continuous, center-of-the-cycle activity — requirements are gathered/validated across every phase, not once at the start. |
| **Requirements Catalog** | The flat, traceable list of requirements: id, statement, source, priority, status, and a link to the principle/work package addressing it. See `00a-Requirements-Management.md §3`. |
| **Requirements Repository** | Where the Requirements Catalog (and everything else the ADM produces) is meant to live on an ongoing basis — part of the Architecture Repository. Confirmed absent for the New KYC rebuild prior to `00a-Requirements-Management.md`. |

## Zachman terms

| Term | Meaning |
|---|---|
| **The 6 columns (interrogatives)** | What (data), How (function), Where (network), Who (people), When (time), Why (motivation). |
| **The 6 rows (perspectives)** | Scope (Planner) → Business Model (Owner) → System Model (Designer) → Technology Model (Builder) → Detailed Representations (Subcontractor) → Functioning Enterprise (the real running thing). |
| **Primitive** | A single, undecomposed cell of the matrix — one perspective's answer to one question. |
| **Composite** | A model that blends multiple primitives — most real-world diagrams are composites, not pure primitives. |

## Apex/SonataOne domain terms

| Term | Meaning |
|---|---|
| **IKYC** | Investor-driven KYC — the self-service journey, built in `S1.Module.Kyc` (TemplateAPI). |
| **MKYC** | Managed KYC — the analyst-driven journey, built in the standalone `ManagedServices` repo. |
| **Profile** | The KYC subject record (`S1.Module.Kyc.Domain.Profile`) — who is being KYC'd. |
| **Visa** | The join between a Profile and a Questionnaire — confusingly named, nothing to do with immigration. See `12-Profile-Visa-...md`. |
| **Questionnaire** | Structure (`JsonConfig`) + text (`Question` rows) defining what must be answered; seeded as SQL, not editable via any UI. |
| **Answer** | A profile's response to one question, optionally scoped to a Fund and/or a `GroupId`. |
| **GroupId** | Correlates multiple answers into one instance of a repeated block (e.g. one of several Tax Residence entries). Root cause of the duplication bug traced in `10-Verified-Store-Proposal.md`. |
| **QuestionnaireStandard** | Which variant of a questionnaire applies (Global, US, Investment KYC, Hellman and Friedman, CBRE) — only Global is used at real scale, per `11-Questionnaire-Standard-And-Section-Counts.md`. |
| **Connection** | An ownership/control relationship between two Profiles (Owners and Controllers). |
| **LinkedProfile** | The (currently stub, unresolved) field meant to connect an MKYC `CounterpartyProfile` back to an IKYC `Profile`. |
| **Strangler-fig** | The migration pattern in use — `IDR` (legacy) being replaced piece by piece by `TemplateAPI`, rather than a big-bang rewrite. |
| **Sync / SyncExchange** | The two services carrying data between IDR and TemplateAPI — `Sync` is a generic .NET proxy, `SyncExchange` (`-tl`/`-tr`) is the Python transform layer. |
