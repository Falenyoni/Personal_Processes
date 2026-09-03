# IDR System Rebuild — Architecture, KYC Scoping & Migration Strategy

Consolidated, diagram-first version of the IDR rebuild analysis. Everything here is grounded in direct inspection of the repos under `C:\Code` (IDR monolith, TemplateAPI, SonataOne.Core, SonataOneSecurity, Sync, and **ManagedServices**) — file paths and enum values are quoted from the actual source, not inferred from naming.

## How to read this

| Doc | Answers |
|---|---|
| [`01-KYC-Explained-And-Access-Control.md`](./01-KYC-Explained-And-Access-Control.md) | What are IKYC/MKYC/KYC? Who can be a KYC subject (not just investors)? Can analysts view any profile? Is "New KYC" only an Investor journey? |
| [`02-Journey-Scoping.md`](./02-Journey-Scoping.md) | How do Investor / Analyst / Fund Manager journeys map to modules, teams, and shared platform capabilities? |
| [`03-Greenfield-Architecture.md`](./03-Greenfield-Architecture.md) | If nothing existed yet, how would you design this? Bounded contexts, service split, database-per-context, event contracts, and a gap analysis against what's actually built. |
| [`04-Target-Solution-And-Database-Structure.md`](./04-Target-Solution-And-Database-Structure.md) | Recommended repo layout, new modules to add, IDR-schema-to-new-service database mapping, migration DB pattern. |
| [`05-Migration-Strategy.md`](./05-Migration-Strategy.md) | Strangler-fig phases, team ownership, risks, immediate next steps. |
| [`06-Glossary-BuyButton-And-ServiceLevels.md`](./06-Glossary-BuyButton-And-ServiceLevels.md) | What is "Buy Button"? What are the Standard/Silver/Gold service levels (and is there really a "Platinum" tier)? What's the difference between a `Connection` and a `Counterparty`? |
| [`07-Independent-Rebuild-Recommendation.md`](./07-Independent-Rebuild-Recommendation.md) | If I were rebuilding this with a blank slate — no reference to IDR, TemplateAPI, ManagedServices, or Subscription — what would I build? Full service list, a service-to-service link map (sync vs. async), two end-to-end sequence diagrams, and an explicit, separated comparison against what's actually being built. |
| [`08-Sync-Strategy-ETL-Vs-Unidirectional-Cutover.md`](./08-Sync-Strategy-ETL-Vs-Unidirectional-Cutover.md) | ⚠️ Would ETL have worked better than the IDR↔TemplateAPI bidirectional sync? Grounded in two live-traced bugs: the Date of Incorporation duplicate-answer defect, and a **confirmed sync echo loop** that resets Date of Birth edits minutes after they're made — full webhook map included. The real fix is unidirectional, per-domain write ownership, not a transport swap. |
| [`09-KYC-Profile-Caching-And-Cross-User-Exposure-Risk.md`](./09-KYC-Profile-Caching-And-Cross-User-Exposure-Risk.md) | ⚠️ Why does the KYC profile page sometimes show a profile the current user isn't authorized for? A likely cross-user data exposure in the frontend's Next.js fetch cache (`GET /kyc/profiles/me`), plus two related-but-distinct findings (stale Redis permission cache, non-permission-gated ownership tree). |
| [`10-Verified-Store-Proposal.md`](./10-Verified-Store-Proposal.md) | ⚠️ Should IDR and `S1.Module.Kyc` share one JSON snapshot per profile instead of today's event sync? Traces a Rebuild/Legacy-team-reported bug set (grouped-field duplication, no delete-on-removal, Rebuild→Legacy partial-payload data loss) to exact file/line evidence across three repos (`IDR`, `SyncExchange`, `S1.Module.Kyc`) — including the precise tax-residence GroupId bug (a stable ID exists and is discarded in favor of `uuid.uuid4()`). Worked through under a "don't change any IDR endpoint" constraint. |
| [`11-Questionnaire-Standard-And-Section-Counts.md`](./11-Questionnaire-Standard-And-Section-Counts.md) | How many questionnaires/sections exist, and how does `QuestionnaireStandard` link to `Questionnaire`/`Visa`/`Profile`? 27 questionnaires, 158 sections, 135 questions — plus live query results showing 3 of 6 standards are entirely unseeded, CBRE has 13 questionnaires but exactly 1 real-world visa, and merged (multi-visa) profiles genuinely exist in production. |
| [`12-Profile-Visa-And-Profile-Creation-Flow.md`](./12-Profile-Visa-And-Profile-Creation-Flow.md) | What is a Profile vs. a Visa, and exactly how does a Visa get determined when IDR creates a profile? Full traced flow from `saga.kyc.profile.synchronize` through `SyncCreateProfileHandler`'s single transaction, `VisaService`'s Global-standard-only matching rule, questionnaire assembly, and how an answer actually lands in `Kyc.Answer` — with mermaid flowchart and sequence diagrams throughout. |
| [`13-Questionnaire-Merging-Standard-Awareness-And-Terminology.md`](./13-Questionnaire-Merging-Standard-Awareness-And-Terminology.md) | Do `QuestionnaireMergingService`/`QuestionnaireMappingService`/`QuestionnaireService` account for `QuestionnaireStandard` or Fund when merging a profile's visas? No — traced directly, confirmed no such check exists anywhere (dormant, not currently exercised). Also: why "merge questions," not "merge questionnaires," is the accurate description — `Questionnaire` rows are never combined; the merge output is a transient DTO. |
| [`14-Sync-Pipeline-Trace-And-Standard-Change-Event.md`](./14-Sync-Pipeline-Trace-And-Standard-Change-Event.md) | How do entities actually get from IDR to Rebuild — class by class across all four repos (`IDR` → `Sync` → `SyncExchange` → `TemplateAPI`)? Extends `08`'s webhook map with the exact classes in between. Also corrects an earlier proposal: a due-diligence-standard-change event routed through `EventingManager` would never leave IDR — the real outbound mechanism is the separate `sync.MessageOutbox`, drained by `InvestorServices.Sync.Service`. |

Related, deeper-dive documents already in the parent folder (not duplicated here):
- `..\IDR-Tax-Compliance-Domain.md` — full FATCA/CRS/W-Forms/withholding domain extract
- `..\KYC-Notes.md` — sync mechanism internals, reliability findings, "Profile vs Passport" history
- `..\ADR-001-Contract-Mediated-Module-Communication.md` — the module-to-module contract pattern referenced throughout

---

## ⚠️ Open question — not yet resolved

**Is "Fund Manager" a real self-service journey, or is that work actually performed by Apex-internal staff on the fund manager's behalf?** `SonataOneSecurity`'s complete role list has 23 internal Apex operations roles (`TransferAgencyAnalyst`, `TaxAnalyst`, `ComplianceUser`, etc.) and one generic external-user role — no "Fund Manager" role. Every IDR controller behind the "Fund Manager journey" (fund closing, transfers, tax reporting, SARs) lives under IDR's staff-only `Admin` namespace, the same signal that correctly flagged Onboarding as staff-driven elsewhere in this review. This is flagged, not fixed — full evidence in `02-Journey-Scoping.md` §6, cross-referenced from `05-Migration-Strategy.md` and `07-Independent-Rebuild-Recommendation.md`. Everything describing a "Fund Manager journey" in this folder should be read with that caveat until someone who knows the real access model confirms one way or the other.

---

## The single most important correction in this pass

Earlier analysis assumed MKYC (managed KYC — the analyst-driven workflow where an ops team fills in KYC on behalf of a counterparty who never logs in) hadn't been started in the new platform. **That's wrong.** `C:\Code\ManagedServices` (repo `SonataOne.ManagedServices`) is a live, actively-developed standalone microservice that *is* the MKYC engine:

- `ProjectType` enum: `Mkyc = 1`, `Ikyc = 2`, `MkycAndIkyc = 3`
- `DealStatus` enum: `AwaitingClientUpdate → WithCounterparty → QuestionPendingWithS1 → Complete` — mirrors IDR's `ManagedKycDashboard` state machine
- Domain: `Project` (analyst-run engagement) → `CounterpartyRequest` → `CounterpartyProfile`/`CounterpartyUser`
- Internal-staff-only auth (`BasicIdentityAuthorizationPolicy`), own database (`SonataOne.ManagedServices.Database`), calls IDR and the Fund module via `IInternalServiceClient`
- Git history: continuous PR merges through 2026-08-11

So the real picture is: **IKYC lives in `TemplateAPI` → `S1.Module.Kyc`; MKYC lives in its own repo, `ManagedServices`.** Both are live. The one concrete gap is that they don't yet share one underlying "who is this KYC subject" record — `ManagedServices.LinkedProfile` is a stub with no typed link back to `S1.Module.Kyc.Profile`. That's the priority integration item, not "build MKYC from scratch."

## Quick answers

**Q: Can internal analysts view any profile in KYC?**
No. Access is per-entity, explicitly granted (Full / Read-only KYC / Managed KYC), enforced by `AuthorizeOrReadOnlyKYCOrManagedKyc` in IDR and by service-specific policies in the new platform. No blanket "view all" role exists anywhere in the system. Full detail: `01-KYC-Explained-And-Access-Control.md` §0, §5.

**Q: Is "New KYC" only building an Investor journey?**
No — it's two services covering both actor types: `S1.Module.Kyc` (IKYC, investor-driven) and `ManagedServices` (MKYC, analyst-driven). Full detail: `01-KYC-Explained-And-Access-Control.md` §0, §6.

**Q: KYC isn't only for investors — how is that modeled?**
IDR's `EntityType` enum defines 13 possible KYC subjects (Individual, Trust, LLC, Limited Partnership, Sovereign Wealth entity, etc.) — "investor" is one of thirteen. Full detail: `01-KYC-Explained-And-Access-Control.md` §2.
