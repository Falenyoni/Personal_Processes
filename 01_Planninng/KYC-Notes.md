# KYC Handover Notes

Living notes for onboarding onto the KYC area before moving teams. Built up incrementally as questions come in — newest sections go at the bottom of each part unless noted.

---

## 1. The big picture

This is a **strangler-fig migration**. The legacy, monolithic KYC system lives in **IDR** (`InvestorServices.*`). A new, standalone KYC module is being built inside **TemplateAPI** (`src/S1.Module.Kyc`) to eventually replace it. Both systems run simultaneously and stay in sync via events while the new module gradually takes over.

```
IDR (legacy "core KYC")              TemplateAPI (new, standalone KYC)
InvestorServices.Api/DD/Models  <--> S1.Module.Kyc
      ^                                     ^
      |  InvestorServices.Sync.Service      |  KycSyncOutboxWorker
      |  (Saga: LegacyToRebuild /           |  -> SyncDataOutboxMessage
      |   RebuildToLegacy)                  |  -> Azure Service Bus
      +----------- Azure Service Bus -------+     (topic: kyc-global-events,
                                                    subject: SonataOne.Models.Events.EntityChangedEvent)
```

## 2. `S1.Module.Kyc` in TemplateAPI

**Architecture**: vertical-slice / feature-folder style. Each use case under `Features/` gets its own folder with an `Endpoint` (thin ASP.NET controller), a MediatR `Request`/`Handler`, and a `Response`.

**Core domain model** (`Domain/`):
- **`Profile`** — the KYC subject (Individual, Joint Account, Partnership, LLC, Trust, Pension, Regulated Entity, etc. — see `ProfileType`). Rich entity with factory methods (`CreateNewProfile`, `UpdateProfile`, `CreateFromLegacySync`) enforcing validation (individuals need name/surname, non-individuals need a legal name, valid country codes).
- **`Connection`** — links two profiles (e.g. owner/controller relationship) with an ownership percentage.
- **`Questionnaire` / `Question` / `Answer`** — the KYC due-diligence questionnaire and submitted answers (polymorphic answer types: bool/int/currency/string/evidence).
- **`Evidence` / `EvidenceDocument` / `EvidenceCertifier` / `EvidenceCertification`** — supporting documents and the certifier workflow (someone certifying a document is a true copy; includes DocuSign integration for e-signature status).
- **`RiskAssessment`**, **`Visa` / `VisaDeclaration`** (declarations of interest), **`EidVDetails`** (electronic ID verification via provider **IdPal** — see `Features/IdPal/*` for webhooks, submission status, verification links).

**Feature areas**: Profiles, OwnersAndControllers (connections), Questionnaires, Answers, Declarations, EvidenceCertification (incl. public/token-based endpoints for external certifiers who aren't logged-in users), EvidenceTypes, RiskAssessments, IdPal.

## 3. The sync mechanism (outbox → Service Bus → IDR saga)

Every domain entity implements `IHasSyncEvents`. When something changes (e.g. `Profile.CreateNewProfile`, `Connection.MarkAsDeleted`), the entity raises a `SyncEvent` with a DTO snapshot. These are persisted as `SyncDataOutboxMessage` rows in the **same DB transaction** as the actual change (transactional outbox pattern — never lost even if the publish step fails).

- **`KycSyncOutboxWorker`** (`Sync/KycSyncOutboxWorker.cs`) — background service, polls on `KycSyncOutboxOptions.Interval` (default 30s).
- **`KycSyncOutboxProcessor`** (`Sync/KycSyncOutboxProcessor.cs`) — claims a batch with an optimistic lock (`LockId`/`LockedAt`, expiry `LockExpiry` default 5 min), publishes each as an Azure Service Bus message, marks it `ProcessedAt` or records an `Error`.
- Sync is **disabled by default** (`KycSyncOutboxOptions.Enabled = false`) — check environment config before assuming it's live.

## 4. IDR side — "core KYC" and how it relates

IDR's `InvestorServices.Sync.Service` has a **Saga** (`EntityChangedNotificationHandler` / `SagaKycSyncOutboxNotificationHandler`) that consumes those Service Bus events and picks a direction:
- `LegacyToRebuild` — changes in old IDR propagate to the new TemplateAPI module.
- `RebuildToLegacy` — changes in the new module propagate back to legacy IDR tables.

This is the "core KYC being changed" — IDR's legacy KYC (`InvestorServices.Api/Controllers/V1/Admin/*Kyc*`, `InvestorServices.DatabaseConfiguration`) is kept in lockstep with the new module in both directions during the transition.

Separately, TemplateAPI's `Security/LegacyUserProfilePermissionResolver` calls **synchronously** into IDR (via `IInternalServiceClient`, cached in Redis) to resolve legacy entity IDs and permissions (Read / Full Control) for a profile — permissioning still lives in the legacy system for now. `Features/OwnersAndControllers/LegacyDiagnostics/LegacyPermissionDebugEndpoint.cs` exists purely to debug this bridge (marked `// TODO: remove in next ticket #26696`).

## 5. Recent/in-progress work (from branch history)

Active feature branches: evidence-document create/update sagas, certifier update saga, KYC actions, outbox worker configuration — consistent with this being a live, ongoing migration, not a finished module.

---

## 6. Improvement opportunities in the new standalone KYC module

Findings below are grounded in the actual source (file paths given), not generic advice.

### Reliability / correctness

1. **Failed outbox messages get permanently stuck.**
   `KycSyncOutboxProcessor.MarkNextProcessingBatch` (`Sync/KycSyncOutboxProcessor.cs:146-149`) only claims rows where `m.Error == null`. Once a message fails once (serialization error, oversized message, transient Service Bus failure) and `Error` is set, it is **excluded from every future batch** — there's no retry, backoff, or dead-letter path. A transient blip permanently drops that sync event, silently desyncing IDR and the new module.
   → Add a retry-count + backoff (clear `Error` after N attempts or route to a dead-letter table/queue with alerting).

2. **No monitoring/alerting on stuck or errored outbox messages.** Nothing surfaces "N messages have been failing for M hours" — worth a health check or metric, since silent drift between the two KYC stores is the exact failure mode a migration like this can't afford.

3. **Two-phase-commit-ish gap in `CreateProfileHandler`.** It calls `CreateSecurityProfileAsync` (external Security API) *before* the DB unit-of-work transaction (`Features/Profiles/CreateProfile/CreateProfileHandler.cs:87-101`). If the DB commit later fails/rolls back, the external security profile is already created and orphaned — no compensating action. Same class of risk wherever an external call precedes/isn't covered by the `IUnitOfWork`.

4. **62 `catch (Exception ex)` blocks across the module** (broad exception swallowing, e.g. throughout `LegacyUserProfilePermissionResolver.cs`). Several re-throw, but several just log and return null/empty — worth auditing which of those silent failures should instead surface as a degraded/error state to the caller rather than looking like "no permissions" or "no data".

### Design / tech debt

5. **`KycModuleService` is a stub.** `Services/KycModuleService.cs` has an unused `_apiClient` field and no methods, despite `IKycModuleService` being registered and injected. Either finish it or remove the abstraction — dead scaffolding is confusing for anyone new to the codebase.

6. **Legacy permission bridge is a long-term coupling point, not just migration scaffolding.** `LegacyUserProfilePermissionResolver` makes a live synchronous call to IDR for every permission check (Redis-cached, but cache miss = hard dependency on IDR being up). Worth a concrete plan/ticket for when permissions get owned natively by the new module, rather than this remaining "temporary" indefinitely — the codebase already treats it as temporary (`LegacyDiagnostics` folder, TODO to remove) but there's no visible target date/ticket tying it to a real cutover milestone.

7. **Open TODOs referencing unresolved tickets**, e.g.:
   - `Configuration/SerializerExtensions.cs:58` — dynamic validator discovery not implemented.
   - `Services/QuestionnaireMergingService.cs:271` — merge logic explicitly unfinished ("finalise the logic here").
   - `Services/QuestionnaireService.cs:181,198` — missing caching on frequently-read, infrequently-changing data (straightforward perf win).
   - `Features/EvidenceCertification/CreateCertifier/CreateCertifierHandler.cs:98` — a check was disabled pending ticket 26028; worth confirming it's still intentionally off.

8. **Minor hygiene**: `Features/OwnersAndControllers/GetChildProfiles/GetChildProfilesRequest .cs` has a trailing space in the filename.

### Test coverage

Test suite is reasonably proportioned (156 test files vs. 65 endpoint files), so this isn't a coverage gap in the aggregate — but worth spot-checking that the sync/outbox failure paths (point 1 above) and the legacy-bridge failure paths (point 6) specifically have tests for the *stuck/error* cases, not just the happy path, since those are the scenarios most likely to cause silent data drift in production.

---

## 7. "Profile" (old vs new) vs. "Passport"

### What a Profile is, old and new

**New (TemplateAPI, `S1.Module.Kyc`)**: `Domain/Profile.cs` is the aggregate root of the whole KYC domain — the record of the party being KYC'd (Individual, Joint Account, Partnership, LLC, Trust, Pension, etc. — see `ProfileType`). It carries identity fields (name, address, country), a `ProfileConfig` (privacy settings), `EidVDetails` (electronic ID verification result), `Visas` (see below), and is the anchor that `Connection`, `Answer`/`Questionnaire`, `Evidence`, and `RiskAssessment` all hang off of.

**Old (IDR / `InvestorServices`)**: the same real-world concept, but split across two linked structures — an `Entity` (`FK_EntityID`) plus a `DueDiligenceProfile` table (`InvestorServices.DD/Database/dbo/Tables/DueDiligenceProfile.cs`) holding the actual DD data as dozens of hardcoded columns (birth date, incorporation details, nationalities, tax residencies, sensitive activities, sources of wealth/funds, FATCA/CRS flags, etc.). The new module replaces most of those hardcoded columns with a dynamic Questionnaire/Answer model instead.

A **"profile check"** in either system means the same thing: look up a party and confirm their identity + due-diligence record — `GetProfileHandler`/`GetProfileSearch` in the new module; the Admin/Entities profile endpoints in IDR.

### How this differs from "Passport"

"Passport" is *not* a distinct domain concept parallel to Profile — it turns out to be two unrelated things:

1. **An abandoned earlier name for Profile itself.** Git history in TemplateAPI shows commit `da01e191` ("Rename Passport to Profile", 2025-09-10) — message: *"We no longer refer to Passports, but rather Profiles, as was done by the monolith."* The module originally modeled the KYC subject as "Passport" (`CreatePassportRequestValidator`, `GetPassportAnswersHandler`, etc.) and renamed it to align with IDR's terminology. A few remnants of the old name are still lying around as cosmetic debt:
   - Test files literally named `GetPassportAnswersHandlerTests.cs`/`GetPassportAnswersEndpointTests.cs` sitting inside the renamed `Features/Profiles/GetProfileAnswers` folder.
   - `Domain/Enums/PassportStatus.cs` (`Active`/`Inactive`) and the `LookupType.PassportStatus` lookup entry — worth checking whether anything still reads/writes these or whether they're dead leftovers from the rename (candidate cleanup item).
   - IDR's legacy front-end also has a screen literally titled **"Search Passport"** (`InvestorServices.RegressionTest/Features/Search/SearchPassport.feature.cs`) — it's just a UI label for "search for a Profile by Profile ID," same idea as the new module's `GetProfileSearch`, different marketing name.

2. **An unrelated, real concept in AML/sanctions screening.** `InvestorServices.DD/WorldCheck/` has `PassportCheckValidityType` (`VALID`/`INVALID`/`NOT_SET`) and `VerifyPassportAuditDetails` — these relate to matching a real physical passport's details (number, nationality) against a World-Check sanctions/PEP watchlist hit, to help confirm whether a screening match is genuinely the same person. This is a narrow field on the **screening** subsystem, not the KYC subject record — a Profile can have zero or many passport documents attached as *evidence* (`EvidenceType`/`EvidenceDocument`), separate from this World-Check validity flag.

**Bottom line**: *Profile* = the KYC subject/record itself (who they are, their due diligence, their connections). *Passport* is either (a) a fully-retired earlier name for that same Profile concept in the new module, or (b) in the unrelated sanctions-screening subsystem, the physical travel document used as one data point to confirm/deny a watchlist match. A "profile check" answers "do we know who this party is and has DD been done"; a "passport check" (screening) answers "does this specific travel document match a sanctioned/PEP individual."

---

## 8. Is the new KYC module actually an improvement over the old?

Short answer: **genuinely better in specific, structural ways — but also a real regression risk in one area, and not finished yet.** Not "the same thing renamed."

### Actually better (structural, not cosmetic)

- **Composable questionnaires vs. a fixed column list.** IDR's `DueDiligenceProfile` table is ~180 hardcoded nullable columns (`FundRaisingMethods`, `IsSeniorExecutive`, `HasBeneficialOwnersTenPercent`, ...) — every new DD question needs a migration + deploy. The new module models DD as `Questionnaire`/`Question`/`Answer`, with a `QuestionnaireStandard` enum (`Global`, `US`, `InvestmentKYC`, and even a fund-specific one, `HellmanFriedman`). A `Profile` can hold multiple `Visa` assignments (one per standard) that `GetMergedQuestionnaireHandler` + `IQuestionnaireMergingService` merge into one composite questionnaire at read time. The old flat-table design structurally cannot express "global standard + this fund's extra questions" — the new one is built for exactly that.
- **Generic ownership/control graph** (`Connection`: primary/secondary profile + type/role) vs. hardcoded single-purpose FK columns in the old table (`FK_ActingOnBehalfOfEntityID`, `FK_SubsidiaryOfListedEntityID`). New model extends to arbitrary relationship structures without schema changes.
- **First-class evidence/certification workflow** — `EvidenceCertifier`, `EvidenceCertification`, DocuSign integration, `EvidenceDocumentAuditEntry` — vs. whatever bolt-on document handling exists in the legacy monolith.
- **Transactional outbox + event sync** gives an auditable integration seam the legacy monolith never had natively (it needed the IDR Sync Service saga bolted on after the fact just to talk to anything else).
- **Test coverage**: 156 test files for 65 endpoints, vs. legacy's coverage being mostly Reqnroll UI regression scripts (e.g. `SearchPassport.feature.cs`) — slower and shallower than unit/handler-level tests.

### Real regression risk

- **Reporting gets harder.** IDR has direct SQL reporting off flat columns (`customreports/Stored Procedures/RPT_Summary_Kyc.sql`, `RPT_List_KycDetails.sql`, `RPT_GetMyProfilesKYCStaticDataProcedure.sql`) — trivial against a fixed schema. The new module's EAV-style `Answer` table (split across `BoolAnswer`/`IntAnswer`/`CurrencyAnswer`/`StringAnswer`/`EvidenceAnswer`) makes equivalent reporting much harder, and no replacement read-model/reporting layer is visible yet in `S1.Module.Kyc`. Confirm this is on the roadmap before any legacy reporting gets retired.
- **Feature surface is much narrower today.** Legacy IDR also covers WorldCheck sanctions screening, FATCA/CRS, 1042/1042s withholding, managed-KYC dashboard/chasers, NCR pack config — none of which exist yet in `S1.Module.Kyc`. Today's "new KYC" is a subset (profiles, connections, questionnaires/answers, evidence/certification, risk assessment, IdPal eIDV), not a full replacement.

### Not finished, so the improvement is partly aspirational

Everything flagged in Section 6 still applies here: sync is disabled by default, failed outbox messages get permanently stuck with no retry, `KycModuleService` is a stub, permissions are still resolved synchronously from legacy via `LegacyUserProfilePermissionResolver`, and `QuestionnaireMergingService.cs:271` has an explicit "finalise the logic here" TODO — on the exact merge feature that's the headline improvement above.

**Verdict**: the target design is a real step up, especially the composable-questionnaire model, which solves a problem the old schema genuinely can't solve. But today it's a partially-built replacement that still depends on the system it's meant to replace, with an open question mark over reporting parity.

---

## 9. How to improve the New KYC, and what the "best" implementation approach would have looked like

### 9a. Prioritized improvement roadmap

**P0 — data-integrity risks, fix before trusting sync in production**
1. **Outbox retry/DLQ** — `KycSyncOutboxProcessor.MarkNextProcessingBatch` permanently excludes any message once `Error` is set (see Section 6 #1). Add retry-with-backoff and a dead-letter path with alerting.
2. **Conflict resolution for bidirectional sync.** `EntityChangedNotificationHandler` in IDR routes events `LegacyToRebuild`/`RebuildToLegacy`, but nothing defines what happens if the same profile is edited in both systems near-simultaneously. Needs an explicit rule (version/timestamp check, last-writer-wins, or lock) — otherwise silent data loss is possible in either direction.
3. **Close the transaction gap in `CreateProfileHandler`** — external security-profile call happens before the DB unit-of-work commits, with no compensating action if the commit later fails (Section 6 #3).
4. **Monitoring on outbox health** — alert on "N messages stuck/erroring for M hours" (Section 6 #2). Silent drift is the worst failure mode for a regulated KYC record.

**P1 — hygiene / tech debt**
5. Finish or delete the `KycModuleService` stub (Section 6 #5).
6. Resolve ticket-linked TODOs: unfinished questionnaire-merge logic (`QuestionnaireMergingService.cs:271`), disabled certifier check pending #26028, missing caching in `QuestionnaireService.cs`.
7. Clean up leftover "Passport" naming (test files, dead `PassportStatus`/`LookupType.PassportStatus` from the 2025-09-10 rename — see Section 7).
8. Trivial: fix the trailing-space filename `GetChildProfilesRequest .cs`.

**P2 — closing the strategic gap identified in Section 8**
9. **Build a reporting/read-model projection.** Highest-value structural gap: legacy has flat-column SQL reports (`RPT_Summary_Kyc`, `RPT_List_KycDetails`); the EAV-style `Answer` table has no equivalent. Reuse the existing `SyncEvent`/outbox stream — add a second consumer that projects into a denormalized reporting schema. Keeps the flexibility of the questionnaire model *and* gets reporting parity, instead of trading one for the other.
10. **Publish an explicit per-bounded-context cutover plan** — screening (WorldCheck), FATCA/CRS, withholding, managed-KYC dashboards all still live only in IDR with no visible migration ticket.
11. **Give the legacy-permission bridge a committed decommission date.** `LegacyUserProfilePermissionResolver` is marked temporary (see `LegacyDiagnostics` debug endpoint, "remove in next ticket" TODO) but isn't tied to a real milestone — risks becoming permanent load-bearing coupling by default.

### 9b. What the "best" implementation approach would have looked like

Concrete choices that stand out as ones worth making differently from the start, based on what this migration has actually run into:

1. **CQRS from day one, not a retrofit.** Keep the flexible write model (composable questionnaires are a genuine win over the old rigid schema — Section 8) but pair it immediately with a purpose-built read model fed by the same event stream, instead of treating reporting as an afterthought once the write model was already EAV-shaped. This single change would have prevented the biggest open structural risk in the migration.

2. **Treat the sync layer as tier-1 infrastructure, not plumbing.** In a strangler-fig migration, the sync mechanism's correctness *is* the migration's correctness — a silently dropped message means the two "sources of truth" diverge without anyone noticing. Retry/DLQ/observability should have shipped with the outbox on day one.

3. **Prefer phased write-ownership handoff over permanent bidirectional sync.** Bidirectional sync (`LegacyToRebuild`/`RebuildToLegacy`) is powerful but requires solving conflict resolution, which doesn't appear to be solved here. A simpler model — legacy becomes read-only for a bounded context the moment that context cuts over — sidesteps that whole bug class, at the cost of needing an explicit cutover sequence per feature area.

4. **Own permissions natively early, or scope the bridge with a hard deadline.** A synchronous runtime dependency on the system being replaced is fine as a short-lived bridge, but should ship with an explicit decommission milestone, not a floating TODO.

5. **Lean into the event log as the canonical audit trail.** `SyncEvent` already captures every domain change with actor/timestamp/correlation ID. Given KYC's compliance weight, that's a strong foundation for a single audit mechanism (potentially full event sourcing per profile) rather than layering separate ad hoc audit-entry tables (`EvidenceDocumentAuditEntry`, etc.) per entity type.

---

## 10. Is KYC a task-triggered workflow that lets you see/time the staff doing the work?

**Yes** — confirmed in both systems as a deliberate assign/due-date/complete mechanism, not just a data record.

### New (TemplateAPI) — `S1.Module.AnalystAction`

`Domain/Operations/Action.cs` is a task record:
- `AssignedTo` — the staff member who owns it.
- `FundId` / `InvestorId` / `ServiceId` — what the task relates to.
- `ServiceLevelAgreementRef` → `ActionServiceLevelAgreement.DueDateHours` — an SLA tier (`ServiceLevelAgreement` enum: `Standard`/`Silver`/`Gold`) that sets how many hours after creation the task is due.
- `IsCompleted` / `CompletedBy` — closes the loop on who did it and when.

That's exactly the "see and time consultants" mechanism: assigned-to + SLA-derived due date + completed-by lets you measure workload, turnaround, and SLA breaches per person. A companion `KycStatus` enum (`Incomplete`/`Approved`/`ApprovedPendingReview`) likely reflects whether a profile's outstanding actions are resolved.

**Caveat — early-stage.** `GetAnalystActionsHandler` currently hardcodes placeholder values ("Investor", "ServiceName") for anything that should come from the legacy monolith (`// TODO: 21687 We need to get the username/fund/investor details from the monolith`), and the only seed data is literally `"Test action 1/2/3"` (`Operations/TestData/ActionType.sql`). Scaffolding, not yet feature-complete.

### Old (IDR) — the `Tasking` engine

The same idea already exists at much larger scale: `InvestorServices.General/Processors/Tasking/` is a rule-driven task engine (`TaskingManager`, `TriggerTaskOperation/TriggerTask.cs` + `Tasks/Admin/Rules/TaskRuleEvaluator`/`TaskConditionMatcher`) that auto-triggers dozens of concrete task types off conditions: `DueDiligenceTask`, `SanctionsScreeningTask`, `EvidenceExpiryTask`, `ScheduledReviewTask`, `HighRiskEscalationTask`, `MKycReportGenerationTask`, etc. Specifically on point: `Tasks/User/UserActionsTask.cs`, `Tasks/User/TeamLeadUserActionsTask.cs`, and `Tasks/SystemTasks/OutstandingTasksTask.cs` exist purely to surface a staff member's (or team lead's) outstanding workload — i.e. legacy already does "see and time the people doing the work," across a far broader trigger surface than the new module currently covers.

(Note: could not confirm the literal word "Consultant" is used as a role name anywhere in IDR — a search timed out on the large legacy tree — but the assign/due-date/complete machinery above is the real timing/attribution mechanism regardless of what the role is labeled.)

---

## Open questions / to explore next
*(add as they come up)*
