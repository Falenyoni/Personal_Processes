# IDR System Rebuild — Architecture & Migration Strategy

**Date:** 2026-08-13  
**Author:** Architecture Review  
**Status:** Recommendation  
**Scope:** Full IDR system decomposition into bounded, journey-scoped services

---

## 1. Executive Summary

The IDR monolith (`InvestorServices.Api`) is a .NET Framework Web API with 300+ controllers and services covering every business domain in a single deployable unit, backed by a single multi-schema SQL Server database. The new platform (`TemplateAPI`) is already establishing the right patterns with `SonataOne.Core`, `SonataOneSecurity`, and the `Sync` service as supporting infrastructure.

This document recommends:
- **User journeys as service boundaries** — Investor, Analyst, Fund Manager scope their own bounded contexts
- **A phased strangler-fig migration** — no big-bang rewrite; traffic is progressively re-routed
- **Database per domain** — each new service owns its schema, migrated from the IDR monolith database
- **Retain IDR as legacy until journey parity** — run both in parallel per feature area

---

## 2. Current Landscape

### 2.1 The IDR Monolith

| Artefact | Tech | Notes |
|----------|------|-------|
| `InvestorServices.Api` | .NET Framework Web API (MVC 5) | 300+ controllers in a single project |
| `InvestorServices.General` | Shared logic library | Cross-cutting domain logic |
| `InvestorServices.Models` | Shared models | DTOs, domain models mixed together |
| `InvestorServices.DD` | Due Diligence logic | Heavily intertwined with main API |
| `InvestorServices.DatabaseConfiguration` | SQL Server DACPAC | 1 database, 11+ schemas: `dbo`, `audit`, `tasks`, `sync`, `security`, `search`, `reports`, `export`, `customreports`, `migration`, `tools` |
| `InvestorServices.Sync.Service` | Legacy sync worker | Being replaced by `Sync` service |
| `InvestorServices.Web` | Legacy web frontend | Being replaced by `IDRFrontend` |
| `InvestorServices.SSRS` | Reporting | SQL Server Reporting Services |

### 2.2 New Platform (Already Exists)

| Service | Repo | Purpose |
|---------|------|---------|
| `TemplateAPI` | `TemplateAPI` | Core KYC, Fund, TransferAgency, RulesEngine, DocuSign, DocumentManagement |
| `SonataOneSecurity` | `SonataOneSecurity` | Identity, permissions, roles, relationships, service configuration |
| `SonataOne.Core` | `SonataOne.Core` | Shared service hosting infrastructure (DomainRepository, InternalServiceClient, Transit, etc.) |
| `Sync` | `Sync` | Async saga/outbox message processing |
| `ApiGateway` | `ApiGateway` | Entry point / BFF routing |
| `Identity` | `Identity` | OIDC / Entra integration |
| `Notification` | `Notification` | Email/SMS dispatch |
| `PublicAPI` | `PublicAPI` | External partner-facing API |
| `TransferAgency` | `TransferAgency` | Subscription & fund operations |

### 2.3 TemplateAPI Module Inventory

Current modules already rebuilt (or in progress):

```
S1.Module.Kyc             — Investor KYC, questionnaires, risk assessment, evidence
S1.Module.RulesEngine     — Risk conditions, calculation engine
S1.Module.Fund            — Fund management, NAV, fund configuration
S1.Module.TransferAgency  — Subscriptions, commitments, transfers
S1.Module.DocuSign        — Document signing workflows
S1.Module.DocumentManagement — Document storage, retrieval
S1.Module.Admin           — Admin actions, configuration
S1.Module.AnalystAction   — Analyst workflow actions
S1.Module.Security        — Module-level authorization
S1.Module.IdPal           — ID verification integration
S1.Shared.*               — Shared contracts, models, utilities
```

---

## 3. User Journey Scoping

### 3.1 The Three Primary Journeys

```
┌─────────────────────────────────────────────────────────────────┐
│                         IDR SYSTEM                              │
├──────────────────┬──────────────────┬───────────────────────────┤
│   INVESTOR       │    ANALYST       │    FUND MANAGER           │
│   JOURNEY        │    JOURNEY       │    JOURNEY                │
├──────────────────┼──────────────────┼───────────────────────────┤
│ Self-service KYC │ Review & approve │ Fund setup & config       │
│ Document upload  │ Risk assessment  │ Investor relationships     │
│ Questionnaires   │ Due diligence    │ Subscription management    │
│ Evidence submit  │ Screening alerts │ Transfer agency ops        │
│ Onboarding       │ Task management  │ Reporting & compliance     │
│ Profile mgmt     │ Escalations      │ Capital account mgmt       │
│ Declarations     │ FATCA/CRS filing │ Fee management            │
└──────────────────┴──────────────────┴───────────────────────────┘
```

### 3.2 Journey → Module Mapping

| Journey | Core Modules | Supporting Modules |
|---------|-------------|-------------------|
| **Investor** | `Kyc`, `DocumentManagement`, `DocuSign`, `IdPal` | `RulesEngine`, `Notification`, `Security` |
| **Analyst** | `AnalystAction`, `RulesEngine`, `Admin` | `Kyc` (read), `DocumentManagement`, `Notification`, `Sync` |
| **Fund Manager** | `Fund`, `TransferAgency`, `DocumentManagement` | `Kyc` (read), `Notification`, `Security` |

### 3.3 Cross-Journey Shared Capabilities

These are **platform capabilities** — not journey-specific:

| Capability | Service | Notes |
|-----------|---------|-------|
| Identity & auth | `SonataOneSecurity` | Roles, permissions, relationships |
| Event messaging | `Sync` (SonataOne.Sync) | Outbox/inbox saga pattern |
| Notifications | `Notification` | Email, SMS |
| Document signing | `DocuSign` module | Shared across journeys |
| Risk/compliance rules | `RulesEngine` | All journeys consume |
| External data | `SonataOneScreening` | WorldCheck, AML screening |
| Reporting | Dedicated service (TBD) | SSRS → modern replacement |

---

## 4. Recommended Solution Structure

### 4.1 Repository & Solution Layout

```
C:\Code\
├── TemplateAPI/                     ← Core domain service (all modules)
│   ├── src/
│   │   ├── S1.Module.Kyc/           ← Investor KYC journey (PRIMARY)
│   │   ├── S1.Module.RulesEngine/   ← Risk calculation (shared)
│   │   ├── S1.Module.Fund/          ← Fund Manager journey (PRIMARY)
│   │   ├── S1.Module.TransferAgency/← Fund Manager journey (PRIMARY)
│   │   ├── S1.Module.DocuSign/      ← Shared signing capability
│   │   ├── S1.Module.DocumentManagement/ ← Shared document store
│   │   ├── S1.Module.Admin/         ← Analyst journey (admin ops)
│   │   ├── S1.Module.AnalystAction/ ← Analyst journey (PRIMARY)
│   │   ├── S1.Module.Security/      ← Auth integration
│   │   ├── S1.Module.IdPal/         ← ID verification
│   │   ├── S1.Shared.Models/        ← All cross-module contracts
│   │   ├── S1.Shared.Common/        ← Shared enums, interfaces
│   │   └── TemplateApi/             ← Host/entry point
│   ├── database/
│   │   ├── schema/
│   │   │   ├── Kyc/                 ← KYC domain tables
│   │   │   ├── Risk/                ← RulesEngine tables
│   │   │   ├── Fund/                ← Fund domain tables
│   │   │   ├── DocuSign/            ← DocuSign tables
│   │   │   ├── Investment/          ← Investment/subscription tables
│   │   │   ├── Documents/           ← Document management tables
│   │   │   ├── Operations/          ← Admin/analyst operations tables
│   │   │   ├── Security/            ← Module-level security tables
│   │   │   ├── HangFire/            ← Recurring job tables
│   │   │   └── User/                ← User identity tables (AspNet*)
│   │   └── DataUpdateScripts/       ← Migration/seed scripts
│   └── test/
│
├── SonataOneSecurity/               ← Identity, permissions, relationships
│   ├── src/
│   │   ├── SonataOne.Security.Domain/
│   │   ├── SonataOne.Security.Application/
│   │   ├── SonataOne.Security.Infrastructure/
│   │   ├── SonataOne.Security.Host/
│   │   └── SonataOne.Security.Models/
│   └── database/                    ← Security-specific schema
│       └── SonataOne.Security.Database/
│           ├── Permissions/
│           ├── ServiceConfiguration/
│           └── Users/
│
├── Sync/                            ← Async messaging, saga, outbox
│   ├── src/
│   │   ├── SonataOne.Sync.Domain/
│   │   ├── SonataOne.Sync.Application/
│   │   ├── SonataOne.Sync.Infrastructure/
│   │   ├── SonataOne.Sync.Host/
│   │   └── SonataOne.Sync.Models/
│   └── database/                    ← Outbox/inbox tables
│       └── SonataOne.Sync.Database/
│
├── SonataOne.Core/                  ← Shared infrastructure (NuGet packages)
│   └── src/
│       ├── SonataOne.Utils/         ← DomainRepository, Http, Transit, Identity
│       ├── SonataOne.Models/        ← ICommand, IEvent, IQuery interfaces
│       ├── SonataOne.Service/       ← ServiceRunner, hosting patterns
│       └── SonataOne.Tests/         ← Test server base infrastructure
│
├── ApiGateway/                      ← BFF routing & aggregation
├── Identity/                        ← OIDC / Entra B2C integration
├── Notification/                    ← Email/SMS service
├── PublicAPI/                       ← Partner-facing external API
├── TransferAgency/                  ← Transfer agency specific ops
└── SonataOneScreening/              ← AML/sanctions screening
```

### 4.2 New Module to Add: `S1.Module.Reporting`

The IDR monolith has rich reporting (`SSRS`, `customreports` schema, `ExcelExportService`, `FatcaCrsBulkXmlGeneration`). This should become its own module or service:

```
S1.Module.Reporting/
├── Features/
│   ├── FATCA/
│   ├── CRS/
│   ├── CapitalAccount/
│   ├── ExcelExport/
│   └── TieReporting/
└── Services/
    └── ReportGenerationService
```

### 4.3 New Module to Add: `S1.Module.Onboarding`

Investor onboarding is a complex flow in IDR (150+ onboarding import service files). It warrants its own module:

```
S1.Module.Onboarding/
├── Features/
│   ├── InviteUser/
│   ├── OnboardingSession/
│   ├── ImportCdd/         ← CDD entity types: Individual, Corporate, Foundation, Trust, etc.
│   ├── OnboardingStage/
│   └── ValidateOnboarding/
└── Services/
    └── OnboardingImportService
```

### 4.4 New Module to Add: `S1.Module.Compliance`

FATCA/CRS classification, screening, due diligence scheduling:

```
S1.Module.Compliance/
├── Features/
│   ├── FatcaCrs/
│   │   ├── Classification/
│   │   ├── Registration/
│   │   ├── Investigation/
│   │   ├── Reporting/
│   │   └── WForms/
│   ├── DueDiligence/
│   │   ├── Schedule/
│   │   ├── EvidenceCertification/
│   │   └── Archive/
│   └── Screening/
│       └── WorldCheck/
└── Services/
```

---

## 5. Database Strategy

### 5.1 IDR Monolith Schema → New Service Mapping

The IDR database has a single SQL Server instance with multiple schemas. The mapping to new bounded contexts:

| IDR Schema | Tables / Domain | New Service | New Schema |
|-----------|-----------------|-------------|------------|
| `dbo` (core entity) | Entity, Profile, Address, Relationship, EntityEvidence, DueDiligence* | `TemplateAPI` | `Kyc`, `Operations` |
| `dbo` (billing) | Billing, FixedFee, LedgerBankPayment | `TemplateAPI` → `S1.Module.Billing` (future) | `Billing` |
| `dbo` (task) | AdminTask, AdminTaskRule, AdminTaskType, AdminTaskPriority | `TemplateAPI` | `Operations` |
| `dbo` (WorldCheck) | WorldCheck*, SanctionsScreening* | `SonataOneScreening` | `Screening` |
| `dbo` (subscription) | Subscription*, InterestOwnership*, InterestTransfer* | `TransferAgency` | `Investment` |
| `dbo` (FATCA/CRS) | FatcaCrs*, Classification*, Jurisdiction* | `TemplateAPI` | `Compliance` (new) |
| `audit` | Audit history tables | All services (own audit) | `audit` per service |
| `tasks` | Scheduled tasks, rule types | `TemplateAPI` | `Operations` |
| `sync` | Sync message queue, outbox | `Sync` | `SonataOne.Sync.Database` |
| `security` | User roles, permissions | `SonataOneSecurity` | `SonataOne.Security.Database` |
| `search` | Search indexes, full-text | `TemplateAPI` | `search` (or Elastic) |
| `reports` / `customreports` | Report definitions, config | `S1.Module.Reporting` | `Reports` |
| `export` | Export jobs, file metadata | `S1.Module.Reporting` | `Reports` |
| `migration` | Migration tracking | Infrastructure | Decommission post-migration |
| `tools` | Utility/maintenance | Infrastructure | Decommission post-migration |

### 5.2 Database Migration Approach

**Principle:** Schema-first, code follows.

**Phase 1 — Schema duplication (parallel write)**
1. New service creates its own schema tables alongside IDR (`TemplateAPI.Kyc.*` vs `IDR.dbo.DueDiligence*`)
2. Sync service (`SonataOne.Sync`) writes to both old and new schemas during transition
3. New API reads from new schema; IDR reads from old schema
4. Both schemas stay in sync via event-driven sync

**Phase 2 — Read cut-over**
1. Verify data integrity between schemas
2. Switch new service to read-only from new schema (old schema becomes the source of truth for writes still going to IDR)
3. New service begins all new writes to its own schema

**Phase 3 — Write cut-over**
1. Route all writes for a feature area to the new service
2. IDR becomes a read-only consumer of new service API for that area
3. Remove IDR write paths for cut-over features

**Phase 4 — Decommission IDR tables**
1. Once IDR routes all reads via new service API, drop old tables
2. Decommission IDR controllers/services for cut-over domain

### 5.3 New Service Database Pattern

Each service follows this established pattern (already in `TemplateAPI` and `SonataOneSecurity`):

```
service/
└── database/
    └── ServiceName.DatabaseConfiguration/  ← .sqlproj DACPAC
        ├── schema/
        │   └── {DomainName}/
        │       └── Tables/
        │           └── *.sql                ← Table definitions
        ├── DataUpdateScripts/               ← One-off migration scripts
        ├── PostDeploymentScript.sql         ← Seed / reference data
        └── PreDeploymentScript.sql          ← Safety checks
```

**New schemas to create in TemplateAPI:**
- `Compliance` — FATCA, CRS, DueDiligence (from IDR `dbo`)
- `Onboarding` — Onboarding sessions, imports (from IDR `dbo`)
- `Billing` — Billing, fees, ledger (from IDR `dbo.Billing*`)
- `Reporting` — Report definitions, export jobs (from IDR `reports`, `export`, `customreports`)
- `Screening` — WorldCheck, AML (→ `SonataOneScreening`)

---

## 6. Work Breakdown — Journey Scoping

### 6.1 Investor Journey

**Scope:** Everything an investor does themselves — onboard, submit documents, answer questionnaires, sign.

| Feature Area | IDR Origin | New Module | Priority |
|-------------|------------|------------|----------|
| Investor profile creation | `BasicInformationController` | `S1.Module.Kyc` | ✅ In progress |
| KYC questionnaire | `DueDiligenceController` | `S1.Module.Kyc` | ✅ In progress |
| Document evidence upload | `EntityEvidenceController` | `S1.Module.Kyc` | ✅ In progress |
| ID verification (IdPal) | `KycApiService` | `S1.Module.IdPal` | ✅ In progress |
| Electronic signing | `DocusignAccountController` | `S1.Module.DocuSign` | ✅ In progress |
| Investor onboarding session | `OnboardingSessionsController` | `S1.Module.Onboarding` (NEW) | 🔶 Planned |
| Onboarding import (CDD types) | `OnboardingImportCdd*Service` | `S1.Module.Onboarding` (NEW) | 🔶 Planned |
| Declaration submission | `DeclarationController` | `S1.Module.Kyc` | 🔶 Planned |
| Invitation management | `OnboardingInvitationController` | `S1.Module.Onboarding` (NEW) | 🔶 Planned |
| Contact information | `ContactInformationController` | `S1.Module.Kyc` | 🔶 Planned |

**Team allocation:** 1-2 developers  
**Dependencies:** `SonataOneSecurity` (auth), `Notification` (invite emails), `DocuSign`, `IdPal`

### 6.2 Analyst Journey

**Scope:** Internal staff reviewing, approving, escalating — everything that happens after an investor submits.

| Feature Area | IDR Origin | New Module | Priority |
|-------------|------------|------------|----------|
| Task management | `AdminTaskController` | `S1.Module.Admin` | 🔶 Planned |
| Task rules & priorities | `AdminTaskRuleController` | `S1.Module.Admin` | 🔶 Planned |
| Risk assessment & scoring | `EntityRiskRatingController` | `S1.Module.RulesEngine` | ✅ In progress |
| Entity review workflow | `EntityReviewController` | `S1.Module.AnalystAction` | 🔶 Planned |
| Due diligence triggers | `DueDiligenceTriggerController` | `S1.Module.Compliance` (NEW) | 🔶 Planned |
| Evidence certification | `EvidenceCertificationController` | `S1.Module.Kyc` | 🔶 Planned |
| Sanctions screening | `SanctionsScreeningController` | `SonataOneScreening` | 🔶 Planned |
| WorldCheck integration | `WorldCheckController` | `SonataOneScreening` | 🔶 Planned |
| Escalations | `EntityHighRiskReviewController` | `S1.Module.AnalystAction` | 🔶 Planned |
| Client services | `ClientServicesController` | `S1.Module.Admin` | 🔶 Planned |
| Managed KYC dashboard | `ManagedKycDahboardController` | `S1.Module.AnalystAction` | 🔶 Planned |
| Admin actions | `AdminActionController` | `S1.Module.Admin` | 🔶 Planned |

**Team allocation:** 2-3 developers  
**Dependencies:** `RulesEngine`, `Kyc` (read), `SonataOneScreening`, `Notification`, `DocumentManagement`

### 6.3 Fund Manager Journey

**Scope:** Fund setup, investor relationship management, reporting, transfer agency operations.

| Feature Area | IDR Origin | New Module | Priority |
|-------------|------------|------------|----------|
| Fund configuration | `FundClosingController` | `S1.Module.Fund` | ✅ In progress |
| Subscription management | `SubscriptionDashboardController` | `S1.Module.TransferAgency` | ✅ In progress |
| Transfer of interest | `InterestTransferController` | `S1.Module.TransferAgency` | 🔶 Planned |
| Capital account statement | `CapitalAccountStatement` (SSRS) | `S1.Module.TransferAgency` | 🔶 Planned |
| Fund emails | `FundEmailsController` | `S1.Module.Fund` | 🔶 Planned |
| Relationship management | `EntityRelationshipController` | `S1.Module.Fund` | 🔶 Planned |
| FATCA/CRS classification | `FatcaCrsClassificationController` | `S1.Module.Compliance` (NEW) | 🔶 Planned |
| FATCA/CRS reporting | `FatcaCrsReportingController` | `S1.Module.Compliance` (NEW) | 🔶 Planned |
| W-Forms management | `WFormsService` | `S1.Module.Compliance` (NEW) | 🔶 Planned |
| Withholding dashboard | `WithholdingDashboardController` | `S1.Module.Compliance` (NEW) | 🔶 Planned |
| Excel export | `ExcelExportController` | `S1.Module.Reporting` (NEW) | 🔶 Planned |
| TIE reporting | `TieReportingController` | `S1.Module.Reporting` (NEW) | 🔶 Planned |
| SAR management | `SARsController` | `S1.Module.Compliance` (NEW) | 🔶 Planned |

**Team allocation:** 2-3 developers  
**Dependencies:** `TransferAgency`, `Fund`, `Notification`, `DocumentManagement`, `Kyc` (read)

---

## 7. Migration Strategy — Strangler Fig Pattern

### 7.1 Overview

```
Phase 1          Phase 2          Phase 3          Phase 4
(Now)            (3-6 months)     (6-12 months)    (12-18 months)

[IDR Monolith]   [IDR Monolith]   [IDR Monolith]   [IDR (legacy)]
[runs fully]     [KYC delegated]  [only Compliance] [read-only]
                 [to TemplateAPI] [+ Billing left]  [then decommission]
                  ↗ [TemplateAPI  ↗ [TemplateAPI    ↗ [TemplateAPI
                    partial]        most journeys]    full parity]
```

### 7.2 Phase 1 — Foundation (0-3 months) ✅ In Progress

**Goal:** Ensure infrastructure is solid before feature migration.

- [ ] ✅ Establish module boundary patterns (complete)
- [ ] ✅ `SonataOneSecurity` — full permissions/roles/relationships service
- [ ] ✅ `Sync` service — outbox/saga messaging infrastructure  
- [ ] ✅ `SonataOne.Core` — hosting patterns, DomainRepository, InternalServiceClient
- [ ] Finalize `ApiGateway` routing strategy (old vs new URLs)
- [ ] Set up dual-write sync from IDR database to TemplateAPI database for Kyc entities
- [ ] Establish contract versioning policy (ADR-001)
- [ ] Dev environment: both IDR and TemplateAPI running locally side-by-side

**DB:** No IDR schema changes. New schemas are additive only.

### 7.3 Phase 2 — Investor Journey Parity (3-6 months)

**Goal:** An investor can complete full onboarding through TemplateAPI (without needing IDR).

Features to complete:
- `S1.Module.Kyc` — all KYC questionnaire flows, evidence, profile management
- `S1.Module.Onboarding` (NEW) — sessions, invitations, CDD import types
- `S1.Module.IdPal` — ID verification full integration
- `S1.Module.DocuSign` — all signature workflows
- `S1.Module.DocumentManagement` — all investor document needs

**Routing strategy:**  
`ApiGateway` routes `/api/investor/*` to TemplateAPI.  
IDR continues serving `/api/analyst/*` and `/api/fund/*`.

**DB migration for Investor Journey:**
1. Dual-write: Investor onboarding data written to both IDR and TemplateAPI
2. Validate parity via data comparison scripts
3. Cut TemplateAPI as source of truth for investor data
4. IDR reads investor data via TemplateAPI internal API

### 7.4 Phase 3 — Analyst & Fund Manager Journey Parity (6-12 months)

**Goal:** Internal analysts and fund managers can operate fully from TemplateAPI.

Features to complete (Analyst):
- `S1.Module.Admin` — task management, admin actions, client services
- `S1.Module.AnalystAction` — entity review, escalations, managed KYC
- `S1.Module.Compliance` (NEW) — due diligence, evidence certification

Features to complete (Fund Manager):
- `S1.Module.Fund` — relationship management, fund emails
- `S1.Module.TransferAgency` — transfer of interest, capital accounts
- `S1.Module.Compliance` (NEW) — FATCA/CRS classification, reporting, W-Forms

Features to complete (Shared):
- `SonataOneScreening` — WorldCheck, AML screening full integration
- `S1.Module.Reporting` (NEW) — Excel export, TIE reporting

**DB migration for Analyst + Fund Manager:**
1. Dual-write for task management, screening, compliance tables
2. Validate parity
3. Cut over to TemplateAPI as source of truth
4. IDR begins proxying through TemplateAPI for these domains

### 7.5 Phase 4 — Decommission IDR (12-18 months)

**Goal:** IDR is retired; TemplateAPI serves all journeys.

- All IDR routes redirected to TemplateAPI equivalents in `ApiGateway`
- IDR database: archived, then decommissioned schema by schema
- Remaining IDR-only features (Billing, custom reports) migrated or deferred
- `InvestorServices.Sync.Service` fully replaced by `Sync` service
- SSRS reports replaced by `S1.Module.Reporting` or external BI tool

---

## 8. Team & Work Splitting

### 8.1 Recommended Team Structure

```
┌─────────────────────────────────────────────────────────┐
│                   PLATFORM TEAM                          │
│  SonataOne.Core, SonataOneSecurity, Sync, ApiGateway    │
│  Shared contracts, database migration tooling            │
└───────────────┬─────────────────┬───────────────────────┘
                │                 │
    ┌───────────▼──────┐ ┌───────▼────────────────────────┐
    │  INVESTOR TEAM   │ │  ANALYST / FUND MANAGER TEAM   │
    │  S1.Module.Kyc   │ │  S1.Module.Admin               │
    │  S1.Module.      │ │  S1.Module.AnalystAction        │
    │   Onboarding     │ │  S1.Module.Compliance           │
    │  S1.Module.IdPal │ │  S1.Module.Fund                 │
    │  S1.Module.      │ │  S1.Module.TransferAgency       │
    │   DocuSign       │ │  S1.Module.Reporting            │
    └──────────────────┘ └────────────────────────────────┘
```

### 8.2 Team Ownership

| Team | Owns | Consumes |
|------|------|---------|
| **Platform** | `SonataOne.Core`, `SonataOneSecurity`, `Sync`, `ApiGateway`, `Identity`, `Notification` | — |
| **Investor** | `S1.Module.Kyc`, `S1.Module.Onboarding`, `S1.Module.IdPal`, `S1.Module.DocuSign`, `S1.Module.DocumentManagement` | Platform, `RulesEngine` |
| **Analyst/FM** | `S1.Module.Admin`, `S1.Module.AnalystAction`, `S1.Module.Compliance`, `S1.Module.Fund`, `S1.Module.TransferAgency`, `S1.Module.Reporting` | Platform, Investor (read), `RulesEngine`, `SonataOneScreening` |

### 8.3 Sprint Work Allocation (Suggested per Phase 2)

**Investor Team Sprint Focus:**
1. Sprint 1-2: `S1.Module.Onboarding` — invitation + session
2. Sprint 3-4: `S1.Module.Onboarding` — CDD import individual/corporate types
3. Sprint 5-6: `S1.Module.Kyc` — declaration, contact information
4. Sprint 7-8: `S1.Module.DocuSign` — remaining signing workflows
5. Sprint 9-10: `ApiGateway` routing + dual-write setup, parity validation

**Analyst/FM Team Sprint Focus:**
1. Sprint 1-2: `S1.Module.Admin` — task management core
2. Sprint 3-4: `S1.Module.Compliance` — due diligence scheduling
3. Sprint 5-6: `S1.Module.AnalystAction` — entity review workflow
4. Sprint 7-8: `S1.Module.Compliance` — FATCA/CRS classification
5. Sprint 9-10: `S1.Module.Reporting` — Excel export, TIE reporting foundation

---

## 9. Cross-Cutting Concerns

### 9.1 Permissions & Authorization

**All permissions are owned by `SonataOneSecurity`.**

Journey-scoped permission model:

```
Connection Type → Role → Permission Category → Permission
                                              ↘ Feature Flag (via FeatureGate)
```

| Journey | Connection Type | Key Roles |
|---------|----------------|-----------|
| Investor | Investor | `INVESTOR_SUBMIT`, `INVESTOR_SIGN`, `INVESTOR_VIEW` |
| Analyst | Internal Staff | `ANALYST_REVIEW`, `ANALYST_APPROVE`, `ANALYST_ESCALATE` |
| Fund Manager | Fund Manager | `FM_FUND_MANAGE`, `FM_REPORT_VIEW`, `FM_SUBSCRIPTION_MANAGE` |

### 9.2 Event-Driven Communication

Use `SonataOne.Sync` for all cross-service state changes:

```
Investor submits KYC questionnaire
  → KYC.QuestionnaireSubmittedEvent
  → Sync delivers to:
      AnalystAction.TaskCreated (creates review task)
      RulesEngine.RiskRecalculated (triggers risk scoring)
      Notification.EmailQueued (notifies analyst)
```

Adopt the `Outbox` pattern (already in `SonataOne.Core/DomainRepository/Outbox`):
- All events persisted to outbox before commit
- Sync service picks up and routes
- Dead letter handling in `Sync.Database.MessageFailure`

### 9.3 Feature Flags

All new features gated behind `[FeatureGate(FeatureOptions.{FeatureName})]`.  
IDR fallback until TemplateAPI reaches parity for a given feature.  
Feature flags managed in Azure App Configuration.

### 9.4 API Gateway Routing

```
ApiGateway routes:
  /api/investor/**  → TemplateAPI (Phase 2+)
  /api/analyst/**   → TemplateAPI (Phase 3+)  [IDR until then]
  /api/fund/**      → TemplateAPI (Phase 3+)  [IDR until then]
  /api/admin/**     → TemplateAPI (Phase 3+)  [IDR until then]
  /api/public/**    → PublicAPI

IDR legacy routes (until full parity):
  All other /api/** → IDR (fallback)
```

---

## 10. Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Data divergence during dual-write | High | Automated parity checks; reconciliation scripts per domain |
| IDR complexity — 300+ controllers | High | Strangle by journey, not by feature; don't over-parallelize |
| Missing business logic in migration | High | IDR regression tests run against TemplateAPI during parallel period |
| Team knowledge silos | Medium | Rotate analysts on both IDR and TemplateAPI PRs for 2 sprints |
| Database migration rollback | Medium | All schema changes are backward-compatible additive during migration; never destructive until parity confirmed |
| FATCA/CRS regulatory deadlines | High | Compliance module prioritized independently of other Phase 3 work |
| SSRS report replacement | Medium | Defer until Phase 4; IDR SSRS runs until replacement ready |
| WorldCheck API dependencies | Medium | `SonataOneScreening` wraps IDR's WorldCheck calls; can reuse IDR as internal proxy initially |

---

## 11. Technology Decisions

| Concern | Decision | Rationale |
|---------|----------|-----------|
| Service framework | .NET 8 (`SonataOne.Core` pattern) | Consistent with existing new platform |
| Database | Azure SQL (per service) | Matches existing; DACPAC for schema management |
| Messaging | Azure Service Bus + Azure Event Grid | Already established in `Sync` and `SonataOne.Core` |
| Caching | Redis (`DistributedCacheService` in `SonataOne.Utils`) | Consistent with existing |
| Auth | Azure Entra + `SonataOneSecurity` | Identity ownership clear |
| API docs | OpenAPI/Swagger (FeatureGate-aware) | Already in TemplateAPI |
| Testing | xUnit + Integration test server pattern (`SonataOne.Tests`) | Established pattern |
| Schema management | DACPAC (.sqlproj) | Already used in all services |
| Search | Consider Elastic Search for `dbo.search` schema replacement | MSSQL full-text search is limiting |
| Reporting | Consider Power BI Embedded or FastReport for SSRS replacement | Evaluate in Phase 3 |

---

## 12. Immediate Next Steps (Next 4 Weeks)

1. **Finalize `ApiGateway` routing table** — map every IDR route to either TemplateAPI target or "IDR legacy" fallback
2. **Create `S1.Module.Onboarding`** project scaffolding in TemplateAPI
3. **Create `S1.Module.Compliance`** project scaffolding in TemplateAPI
4. **Set up dual-write sync** — Investor KYC data flows to both IDR and TemplateAPI databases
5. **Data parity tooling** — script that compares IDR entities with TemplateAPI entities (by GlobalId)
6. **Team kick-off** — align Investor Team and Analyst/FM Team on sprints 1-2 deliverables
7. **ADR for event contracts** — define events published per journey transition (ADR-002)
8. **Decommission plan for IDR tables** — map each IDR dbo.* table to new schema owner

---

---

## 13. Understanding KYC in IDR — IKYC vs MKYC vs KYC

> **Critical:** KYC in IDR is not one thing. It is a service type — a mode of operation applied to an entity going through due diligence. Misunderstanding this leads to collapsed module designs. This section must inform how `S1.Module.Kyc` is scoped.

### 13.1 The Three KYC Service Types (from `Service` enum)

| Code | Service ID | Description | Who uses it |
|------|-----------|-------------|-------------|
| **KYC / CDD** | 8 | Standard Client Due Diligence — investor completes themselves via the portal | Investor (self-service) |
| **IKYC** | 20 | Investment KYC — KYC for investment entities/funds, or investor registration flow | Investors registering; also fund vehicles as KYC subjects |
| **MKYC** | 18 | Managed KYC — Apex/operations staff manage KYC on behalf of the client; the client is the counterparty | Operations team (Analyst journey) |

These are fundamentally **different operational modes** for the same underlying KYC data model.

### 13.2 What Each Mode Means

#### IKYC (Investment KYC / Registration KYC)
- Triggered when an **investor self-registers** through the portal
- The investor drives the process — fills in own data, uploads own documents
- Has its own `DueDiligenceStandard`: `InvestmentKycStandard = 4`
- Has its own service fields: `ServiceSubscriptionIKYC`, `DueDiligenceIKYC`, `EvidenceIKYC`, etc.
- Status tracked via `IkycDashboardRequestOverallStatusEnum` (NOT_STARTED → IN_PROGRESS → PARKED → COMPLETED)
- Chaser emails sent via `IKYCChaserEmail` schedule (configured per occurrence/day)
- The investor is both the **actor** and the **KYC subject**

#### MKYC (Managed KYC)
- Triggered when **Apex operations team manages KYC on behalf of a client** (typically institutional investors who don't self-serve)
- The client is the **counterparty** — they receive and respond to requests but don't log into the portal
- Has its own `ManagedKycDashboard` with status: NOT_STARTED → IN_PROGRESS → **AWAITING_CLIENT_UPDATE** → **WITH_COUNTERPARTY** → **QUESTION_PENDING_WITH_IDR** → COMPLETED
- Has counterparty-specific actions: `CounterpartyconfirmsacceptanceofEIC`, `Counterpartyconfirmsacceptanceofdocuments`
- Uses `MkycBulkChaserEmail` and `MKycReportGenerator` (separate from IKYC)
- `CounterPartyRequestsRelatedProfile` and `CounterPartyRequestsRelatedProfileEvidenceDocument` are key data tables
- `RelationshipType.CounterpartyReceivingKYC = 63` defines the counterparty relationship
- `ServiceField.ProfileTypeMKYC = 104` — profile types are tracked per service
- Has `ManagedKycCertificate` — signed certificate confirming KYC completion
- The **analyst is the actor**; the client/counterparty is the **KYC subject**

#### KYC / CDD (Standard Due Diligence)
- General CDD standard (service 8)
- Applies to standard investor self-service onboarding
- `DueDiligenceStandardType`: `HellmanFriedman`, `SonataOneUs`, `SonataOneGlobal` — different rule sets per fund manager client
- `AuthorizeOrReadOnlyKYCOrManagedKyc` — the same `DueDiligenceController` handles both KYC and MKYC but with different permission paths

### 13.3 Who Can Be a KYC Subject

KYC is **not only for investors**. The `EntityType` enum shows **13 entity types** that can all be subjects of KYC:

| Entity Type | Example |
|------------|---------|
| Individual | Private investor, beneficial owner |
| Listed Entity | Publicly listed company investing |
| Regulated Entity | Bank, insurance company |
| Private Company | Private equity firm |
| Limited Partnership | LP / fund vehicle itself |
| Trust | Family trust, charitable trust |
| Pension / EBT / IRA | Employee benefit trust |
| University | Endowment fund |
| Public Body / Government | Sovereign wealth entity |
| Foundation / Non-Profit | Family foundation |
| Sovereign Wealth | Sovereign wealth fund |
| Limited Liability Company | LLC investing in fund |
| Joint Account | Joint investment account |

**Implication for architecture:** `S1.Module.Kyc` must support KYC for **all entity types**, not just individual investors. The KYC subject is a `Profile` entity, which has a `ProfileType` (aligned to `EntityType`) and a `ServiceType` (IKYC or MKYC or CDD).

### 13.4 How This Maps to the Recommended Architecture

```
KYC Service Type → Module Ownership
─────────────────────────────────────────────────────────────────
IKYC (self-service)     → S1.Module.Kyc
                            ├── KYC questionnaire (all EntityTypes)
                            ├── Evidence upload
                            ├── IKYC chaser email schedule
                            └── InvestmentKycStandard due diligence rules

MKYC (managed service)  → S1.Module.AnalystAction + S1.Module.Kyc
                            ├── ManagedKycDashboard (AnalystAction)
                            ├── CounterPartyRequests management (AnalystAction)
                            ├── ManagedKycCertificate (AnalystAction)
                            ├── MkycBulkChaserEmail (AnalystAction)
                            └── Shared KYC data model (Kyc)

KYC/CDD (standard)      → S1.Module.Kyc
                            ├── DueDiligence standard evaluation
                            ├── SonataOneUs/Global/HellmanFriedman standards
                            └── Profile completion tracking
```

**Key design rule:**
- `S1.Module.Kyc` owns the **KYC data model** (Profile, DueDiligence, Evidence, DueDiligenceStandard)
- `S1.Module.AnalystAction` owns the **MKYC workflow** (ManagedKycDashboard, counterparty coordination, certificate management)
- `S1.Module.Kyc` exposes shared KYC data to `S1.Module.AnalystAction` via `IKycModuleService` contract (following ADR-001)

### 13.5 DueDiligence Standards as Configuration

The `DueDiligenceStandardType` should become **configuration**, not hardcoded logic:

```
DueDiligenceStandard (entity per fund)
  ├── HellmanFriedman = 1       → specific questionnaire fields + evidence requirements
  ├── SonataOneUs = 2           → US-specific DD rules
  ├── SonataOneGlobal = 3       → Global standard
  └── InvestmentKycStandard = 4 → Investment KYC specific rules
```

In the new architecture, each standard maps to a **RulesEngine rule set** — the `S1.Module.RulesEngine` evaluates which questions and evidence are required based on the standard applied to the fund/profile combination.

### 13.6 Services Beyond KYC (Same Entity, Multiple Services)

An entity can be subscribed to **multiple services simultaneously**:

| Service | Code | Description |
|---------|------|-------------|
| CDD | 8 | Due diligence |
| FATCA | 7 | US tax compliance classification |
| KPMG | 9 | KPMG-specific review standard |
| USQ | 11 | US questionnaire |
| MANAGEDKYC | 18 | Managed KYC (MKYC) |
| IKYC | 20 | Investment KYC registration |
| CLIENTSUBSCRIPTION | 22 | Fund subscription workflow |
| MFATCA | 27 | Managed FATCA |
| OM | 28 | Ongoing Monitoring |
| WFORM | 32 | W-Form tax withholding |
| WITHHOLDING | 33 | Withholding tax |
| MLRO | 30 | Money Laundering Reporting Officer review |

Each service has its own `ServiceField` progress tracking (e.g., `DueDiligenceMKYC`, `EvidenceIKYC`, `OverallOM`) — these are **completion status fields per service per profile**.

**Architecture implication:** The `S1.Module.Kyc` profile entity must support **service subscriptions** — a profile's completion state is tracked per service, not just once. This is the `ServiceField` concept that must be preserved in the new data model.

---

## 14. Greenfield Architecture — If Nothing Were Built Yet

> This section describes how the system would be designed from scratch, with no legacy constraints, no existing TemplateAPI code, and free choice of structure. It is intentionally aspirational — use it to evaluate whether current decisions are moving in the right direction.

### 14.1 Starting Principles (Greenfield)

1. **Domain-driven boundaries** — each bounded context owns its data, logic, and API
2. **Journey-first, not feature-first** — design starts from user goals, not database tables
3. **Shared platform, not shared code** — common infrastructure as NuGet packages (already done via `SonataOne.Core`)
4. **Event-driven by default** — state changes emit events; modules subscribe rather than call each other
5. **One database schema per bounded context** — no shared tables across services
6. **KYC as a platform capability** — not a single module but a set of composable services

### 14.2 Bounded Context Map

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                           IDR PLATFORM DOMAIN                                  │
│                                                                                │
│  ┌────────────────┐   ┌────────────────┐   ┌──────────────────────────────┐  │
│  │  IDENTITY &    │   │   ONBOARDING   │   │   COMPLIANCE CONTEXT         │  │
│  │  ACCESS        │   │   CONTEXT      │   │                              │  │
│  │                │   │                │   │  ┌──────────┐  ┌──────────┐  │  │
│  │ SonataOne      │   │ S1.Module.     │   │  │  KYC     │  │  FATCA/  │  │  │
│  │ Security       │   │ Onboarding     │   │  │  Engine  │  │   CRS    │  │  │
│  │                │   │                │   │  │(IKYC/    │  │          │  │  │
│  │ - Users        │   │ - Invitation   │   │  │ MKYC/    │  │          │  │  │
│  │ - Roles        │   │ - Sessions     │   │  │  CDD)    │  │          │  │  │
│  │ - Permissions  │   │ - CDD import   │   │  └──────────┘  └──────────┘  │  │
│  │ - Relationships│   │ - Stages       │   │                              │  │
│  └────────────────┘   └────────────────┘   │  ┌──────────┐  ┌──────────┐  │  │
│                                             │  │ DueDili- │  │Screening │  │  │
│  ┌────────────────┐   ┌────────────────┐   │  │  gence   │  │  (AML/   │  │  │
│  │  FUND          │   │  INVESTOR      │   │  │          │  │ WorldChk)│  │  │
│  │  MANAGEMENT    │   │  PORTAL        │   │  └──────────┘  └──────────┘  │  │
│  │  CONTEXT       │   │  CONTEXT       │   └──────────────────────────────┘  │
│  │                │   │                │                                      │
│  │ S1.Module.Fund │   │ Profile,       │   ┌──────────────────────────────┐  │
│  │ TransferAgency │   │ Evidence,      │   │   OPERATIONS CONTEXT         │  │
│  │                │   │ Documents,     │   │                              │  │
│  │ - Fund config  │   │ Questionnaire  │   │ S1.Module.Admin              │  │
│  │ - Subscriptions│   │ DocuSign       │   │ S1.Module.AnalystAction      │  │
│  │ - TOI          │   │                │   │ S1.Module.RulesEngine        │  │
│  │ - Capital A/C  │   └────────────────┘   │                              │  │
│  └────────────────┘                        │ - Task management            │  │
│                                             │ - MKYC managed service       │  │
│  ┌────────────────────────────────────┐    │ - Risk scoring               │  │
│  │         PLATFORM SERVICES          │    │ - Analyst workflows          │  │
│  │  Sync | Notification | Reporting   │    └──────────────────────────────┘  │
│  │  ApiGateway | Identity | PublicAPI  │                                      │
│  └────────────────────────────────────┘                                      │
└────────────────────────────────────────────────────────────────────────────────┘
```

### 14.3 Greenfield Service Design

#### A. Identity & Access Service (`SonataOneSecurity`) ✅ Exists
No changes needed. Already cleanly separated.

```
SonataOneSecurity
├── Users (UserIdentity, GlobalId)
├── Roles (role assignments per connection type)
├── Permissions (granular feature permissions)
├── Relationships (user ↔ entity relationships)
└── Service Configuration (which services enabled per environment)
```

#### B. KYC Platform — Three Separate Services (Greenfield split)

In a true greenfield, **KYC would be three distinct services**, not one module:

**B1. `KycCore` Service — The KYC Data Model**
```
KycCore/
├── Profile (the KYC subject — any EntityType)
├── DueDiligence (questionnaire state per profile per standard)
├── Evidence (documents uploaded as proof)
├── EvidenceCertification (third-party certifier workflow)
├── ServiceSubscription (which services a profile is enrolled in)
└── RiskAssessment (current risk level + applied conditions)
```
- Owned by a **Platform KYC team**
- All other services read/write via API
- One database schema: `Kyc`

**B2. `InvestorKyc` Service (IKYC) — Self-Service Portal**
```
InvestorKyc/
├── InvestorPortal (investor-facing questionnaire UI API)
├── QuestionnaireFlow (multi-step form logic)
├── EvidenceUpload (investor uploads own documents)
├── InvitationFlow (invite → register → onboard)
├── IkycChaserSchedule (automated chaser emails)
└── IdVerification (IdPal integration)
```
- Investor is the actor
- Calls `KycCore` for data persistence
- Calls `SonataOneSecurity` for auth
- Owned by an **Investor Experience team**

**B3. `ManagedKyc` Service (MKYC) — Operations Managed Service**
```
ManagedKyc/
├── ManagedKycDashboard (analyst view of all MKYC cases)
├── CounterPartyManagement (requests sent to clients)
├── CertificateManagement (issued KYC completion certificates)
├── BulkChaserEmail (scheduled reminders to counterparties)
├── MkycReporting (MKYC-specific reports)
└── WorkflowEngine (state machine: NOT_STARTED → WITH_COUNTERPARTY → COMPLETED)
```
- Analyst is the actor; client/counterparty is subject
- Calls `KycCore` for underlying KYC data
- Owned by an **Operations / Managed Services team**

**Why split?** IKYC and MKYC have completely different actors, workflows, state machines, notifications, SLAs, and dashboards. In IDR they are tangled — `DueDiligenceController` handles both via `AuthorizeOrReadOnlyKYCOrManagedKyc`. This single controller is the root cause of complexity.

#### C. Compliance Service
```
Compliance/
├── FatcaCrs/ (classification, registration, investigation, reporting)
├── DueDiligence/ (scheduling, triggers, ongoing monitoring)
├── Screening/ (WorldCheck, AML, sanctions — or delegate to SonataOneScreening)
├── WForms/ (W-8, W-9, W-series management)
└── Withholding/ (1042-S, withholding tax dashboard)
```

#### D. Fund Management Service
```
FundManagement/
├── FundConfiguration/ (fund setup, closing, email config)
├── Relationships/ (fund ↔ investor relationship management)
├── Subscriptions/ (subscription documents, questionnaire)
└── Reporting/ (fund-level reporting)
```

#### E. Transfer Agency Service ✅ Exists (TransferAgency)
Extend with:
```
TransferAgency/
├── Commitments/ (capital commitments)
├── TransferOfInterest/ (TOI workflow, stages)
├── CapitalAccount/ (statements, NAV)
└── SubscriptionDocuments/ (side letters, fund docs)
```

#### F. Operations Service
```
Operations/
├── TaskManagement/ (AdminTask, rules, priorities, categories)
├── AnalystWorkflow/ (entity review, escalations, high-risk)
├── OngoingMonitoring/ (OM service, scheduled reviews)
└── ClientServices/ (billing contact, client dashboard)
```

#### G. RulesEngine Service ✅ Exists (as module)
In greenfield, this would be a standalone service:
```
RulesEngine/
├── RiskConditions/ (configurable risk rules)
├── CalculateRisk/ (evaluates answers against conditions)
├── QuestionFields/ (metadata for UI)
└── DueDiligenceStandards/ (which standards apply per fund)
```
- Publishes: `RiskLevelCalculatedEvent`
- Consumed by: KycCore, Compliance, AnalystWorkflow

### 14.4 Greenfield Database Strategy

**One schema per bounded context. No cross-schema joins ever.**

| Bounded Context | Database Schema | Key Tables |
|----------------|-----------------|------------|
| Identity | `Security.*` | `UserIdentity`, `Role`, `Permission`, `ProfileConnection` |
| KycCore | `Kyc.*` | `Profile`, `DueDiligence`, `Evidence`, `ServiceSubscription`, `RiskAssessment` |
| InvestorKyc | `InvestorPortal.*` | `OnboardingSession`, `Questionnaire`, `InvitationFlow` |
| ManagedKyc | `ManagedKyc.*` | `ManagedKycDashboard`, `CounterPartyRequest`, `ManagedKycCertificate` |
| Compliance | `Compliance.*` | `FatcaCrsClassification`, `DueDiligenceSchedule`, `Withholding`, `WForm` |
| FundManagement | `Fund.*` | `Fund`, `FundClosing`, `FundEmail` |
| TransferAgency | `Investment.*` | `Subscription`, `TransferOfInterest`, `CapitalAccount` |
| Operations | `Operations.*` | `AdminTask`, `TaskRule`, `EntityReview`, `OngoingMonitoring` |
| RulesEngine | `Risk.*` | `RiskCondition`, `CountryRisk`, `CalculateRiskAudit` |
| Reporting | `Reports.*` | `ReportDefinition`, `ExportJob`, `ReportSchedule` |
| Sync/Messaging | `Sync.*` | `MessageOutbox`, `MessageFailure` |

**Data access rules:**
- A service **owns** its schema — it is the only writer
- Other services **read** via API calls, never via direct DB joins
- Reporting service may use **read replicas** or **projections** for query optimization

### 14.5 Greenfield Event Contracts

Key domain events (all via `SonataOne.Sync`):

```
KycCore publishes:
  ProfileCreatedEvent          → triggers: Invitation, TaskCreation
  KycSubmittedEvent            → triggers: AnalystWorkflow.ReviewTaskCreated
  RiskLevelChangedEvent        → triggers: AnalystWorkflow.EscalationCheck, Notification
  EvidenceUploadedEvent        → triggers: EvidenceCertification.CertifierNotified
  EvidenceCertifiedEvent       → triggers: KycCore.CompletionCheck

InvestorKyc publishes:
  InvestorRegisteredEvent      → triggers: KycCore.ProfileCreated, Security.UserCreated
  IkycCompletedEvent           → triggers: Operations.TaskCreated (review task)

ManagedKyc publishes:
  MkycRequestCreatedEvent      → triggers: Notification.CounterpartyInvited
  CounterpartyRespondedEvent   → triggers: ManagedKyc.StateAdvanced
  MkycCertificateIssuedEvent   → triggers: KycCore.ServiceSubscription.Complete

RulesEngine publishes:
  RiskCalculatedEvent          → triggers: KycCore.RiskAssessmentUpdated

Compliance publishes:
  FatcaClassificationChangedEvent → triggers: Notification.FundManagerAlerted
  ScreeningMatchFoundEvent        → triggers: Operations.EscalationTaskCreated

TransferAgency publishes:
  SubscriptionCreatedEvent     → triggers: InvestorKyc.KycRequired
  TransferOfInterestInitiated  → triggers: Operations.ToiTaskCreated
```

### 14.6 Greenfield vs Current State — Gap Analysis

| Area | Greenfield Design | Current TemplateAPI | Gap |
|------|------------------|---------------------|-----|
| KYC split (IKYC/MKYC) | 3 separate services | 1 module (`S1.Module.Kyc`) | MKYC workflow not separated |
| Compliance | Dedicated service | Not built yet | Full gap |
| IKYC chaser schedule | `InvestorKyc` service | Not built | Full gap |
| MKYC dashboard | `ManagedKyc` service | Not built | Full gap |
| KYC entity types (13) | All in `KycCore` | Partially (individual-focused) | Partial gap |
| Service subscriptions | Explicit model | Partially modelled | Partial gap |
| DueDiligence standards | Config-driven | Hardcoded logic | Gap |
| RulesEngine | Standalone service | Module (partially right) | Small gap |
| Event-driven | Default | Partial (via Sync) | Growing parity |

### 14.7 Pragmatic Recommendation

Given TemplateAPI already exists and is live:

> **Do not split `S1.Module.Kyc` into three services today.** Instead, introduce **internal sub-contexts** within the module that respect the IKYC/MKYC boundary — and track whether they grow into independent services in Phase 3-4.

```
S1.Module.Kyc/
├── Features/
│   ├── InvestorKyc/          ← IKYC sub-context (self-service)
│   │   ├── Questionnaires/
│   │   ├── EvidenceUpload/
│   │   └── IkycChaser/
│   ├── ManagedKyc/           ← MKYC sub-context (NEW — currently missing)
│   │   ├── ManagedKycDashboard/
│   │   ├── CounterPartyRequests/
│   │   ├── ManagedKycCertificate/
│   │   └── MkycChaser/
│   └── Core/                 ← Shared KYC data (Profile, Evidence, DueDiligence)
│       ├── Profiles/
│       ├── Evidence/
│       ├── EvidenceCertification/
│       └── ServiceSubscriptions/
└── Domain/
    ├── Profile.cs            ← All 13 EntityTypes
    ├── ServiceSubscription.cs← Per-service tracking
    └── DueDiligenceStandard.cs← Standard config
```

**When to split into separate services:**
- MKYC team grows beyond 2 developers
- MKYC deployment frequency diverges from IKYC
- MKYC has its own SLA/availability requirements

---

## Appendix A — IDR Schema to New Service Quick Reference

```
IDR dbo.*Entity*              → TemplateAPI Kyc schema
IDR dbo.*DueDiligence*        → TemplateAPI Kyc + Compliance schema
IDR dbo.*Onboarding*          → TemplateAPI Onboarding schema (NEW)
IDR dbo.*FatcaCrs*            → TemplateAPI Compliance schema (NEW)
IDR dbo.*AdminTask*           → TemplateAPI Operations schema
IDR dbo.*WorldCheck*          → SonataOneScreening
IDR dbo.*Subscription*        → TransferAgency
IDR dbo.*Interest*Transfer*   → TransferAgency
IDR dbo.*Billing*             → TemplateAPI Billing schema (future)
IDR dbo.*Document*            → TemplateAPI Documents schema
IDR sync.*                    → Sync (SonataOne.Sync.Database)
IDR security.*                → SonataOneSecurity (SonataOne.Security.Database)
IDR audit.*                   → Each service owns its own audit tables
IDR reports.* / export.*      → TemplateAPI Reporting schema (NEW)
IDR search.*                  → TemplateAPI (or Elastic)
IDR tasks.*                   → TemplateAPI Operations schema
IDR tools.*                   → Decommission post-migration
IDR migration.*               → Decommission post-migration
```

## Appendix B — Module Contract Boundary Rules

All cross-journey module communication must follow ADR-001:
- Shared contracts in `S1.Shared.Models.Contracts.{Module}/`
- Module service adapters (e.g., `IKycModuleService`) for inter-module calls
- Internal endpoints tagged `[InternalRoute<T>]`
- No direct project-to-project references between domain modules
- Events for state change notifications (via `Sync`)
- Request/response for synchronous queries (via `IInternalServiceClient`)

See: `C:\Code\ADR-001-Contract-Mediated-Module-Communication.md`
