# Diagram and Artifact Guide

**Terms used throughout:** [`Glossary.md`](./Glossary.md).

**Purpose:** TOGAF's Architecture Content Framework defines roughly three dozen standard artifact types (catalogs, matrices, diagrams) across the ADM phases. Producing all of them for a first pass is how EA initiatives die of documentation weight before they deliver anything. This guide picks the small, high-value subset worth actually producing for this initiative, explains *why* each one earns its place, and gives a real mermaid template for each — grounded in Apex/SonataOne's actual landscape, not a generic example.

**The three artifact types, and what each is for:**
- **Catalog** — a structured list (e.g. every application, every business capability). The raw inventory everything else is built from.
- **Matrix** — a cross-reference between two catalogs, showing relationships (e.g. which applications support which business capability). Where gaps and duplication become visible.
- **Diagram** — a visual representation for communication — of a catalog, a matrix, or a flow. What you'd actually show a room of stakeholders.

---

## Phase A — Architecture Vision

### Stakeholder Map (matrix)

Who cares about this initiative, and how much power/interest do they have. Standard TOGAF power/interest grid.

Reconciled against `03-Phase-B-Business-Architecture.md §1`'s Organization/Actor Catalog, drafted after this map originally was - two points below were quietly conflating things Phase B later showed are not the same. "Investors and Fund Managers" bundled a confirmed external self-service actor (Investor) with one whose external-vs-staff attribution is explicitly unconfirmed (Fund Manager, Phase B §4) - collapsing them into one point hid the exact ambiguity Phase B took pains to surface. "Compliance and Onboarding Ops" mixed a real, distinct actor (Compliance Officer) with a capability Phase B confirmed is performed by Analysts, not a separate group. Both are split below.

```mermaid
quadrantChart
    title Stakeholder Map - KYC Platform Modernization
    x-axis Low Interest --> High Interest
    y-axis Low Power --> High Power
    quadrant-1 Manage Closely
    quadrant-2 Keep Satisfied
    quadrant-3 Monitor
    quadrant-4 Keep Informed
    "Engineering Leadership": [0.8, 0.9]
    "Compliance Officer": [0.85, 0.6]
    "Investors - confirmed external, self-service": [0.5, 0.3]
    "Fund Manager - attribution unconfirmed, position provisional": [0.4, 0.3]
    "Analysts - internal, incl. onboarding and MKYC": [0.9, 0.4]
    "IDR Legacy Maintainers": [0.6, 0.5]
```

### Solution Concept (diagram)

The one-slide "what are we actually doing" — deliberately non-technical, this is the diagram that goes in front of people who've never heard of `S1.Module.Kyc`.

```mermaid
flowchart LR
    Legacy["IDR — legacy monolith\n(current system of record)"] -->|"strangled out, journey by journey"| Target["TemplateAPI — modular platform\n(target system)"]
    Target --> IKYC["IKYC — investor self-service\nS1.Module.Kyc"]
    Target --> MKYC["MKYC — analyst-managed\nManagedServices"]
    Target --> Fund["Fund administration\nS1.Module.Fund"]
    Target --> TA["Transfer Agency\nS1.Module.TransferAgency"]
```

---

## Phase B — Business Architecture

### Business Capability Map (catalog + diagram)

The capabilities the business needs, independent of which system currently provides them — this is the artifact that lets you ask "do we have this capability at all" before asking "which system has it." Grounded in the journeys already scoped in `02-Journey-Scoping.md`.

```mermaid
flowchart TD
    subgraph Onboarding["Client Onboarding"]
        KYC["KYC / Due Diligence"]
        EDD["Enhanced Due Diligence"]
        EIDV["Identity Verification (eID&V)"]
    end
    subgraph Ops["Ongoing Operations"]
        FundAdmin["Fund Administration"]
        TAOps["Transfer Agency"]
        TaxComp["Tax Compliance (FATCA/CRS/W-Forms)"]
        RiskMon["Risk Monitoring"]
    end
    subgraph Support["Supporting Capabilities"]
        Sync["Cross-System Data Sync"]
        Security["Identity & Access"]
        Notify["Notifications"]
    end
```

### Actor/Role Matrix

Who's allowed to do what, against which capability — the business-layer counterpart of the access-control work already done in `01-KYC-Explained-And-Access-Control.md`.

```mermaid
flowchart LR
    subgraph Actors
        Investor
        Analyst
        FundManager["Fund Manager"]
        ComplianceOfficer["Compliance Officer"]
    end
    Investor -->|self-serve| KYC["KYC / Due Diligence"]
    Analyst -->|manages on behalf of| KYC
    Analyst -->|manages| EDD["Enhanced Due Diligence"]
    FundManager -.->|"status unclear — see 02-Journey-Scoping.md §6"| FundAdmin["Fund Administration"]
    ComplianceOfficer -->|reviews| RiskMon["Risk Monitoring"]
```

---

## Phase C — Data Architecture

### Conceptual Data diagram

The entities the enterprise cares about, technology-agnostic — this is the level Zachman's "What/Business Model" row lives at.

```mermaid
erDiagram
    PROFILE ||--o{ ANSWER : "has"
    PROFILE ||--o{ VISA : "assigned"
    VISA }o--|| QUESTIONNAIRE : "references"
    QUESTIONNAIRE ||--o{ QUESTION : "structures via JsonConfig"
    PROFILE ||--o{ CONNECTION : "owns/controls"
    PROFILE ||--o{ EVIDENCE_DOCUMENT : "submits"
```

### Data Dissemination Diagram

Where does a given data entity actually live and get replicated — directly informed by the sync investigation. This is exactly the diagram that would have made the tax-residence duplication bug's root cause visible *before* it shipped.

```mermaid
flowchart LR
    IDR[("IDR\nDueDiligenceProfile\n+ TaxResidence, SourceOfFunds...")]
    SyncExch["SyncExchange\n(sync-exchange-tr / -tl)"]
    Kyc[("S1.Module.Kyc\nProfile, Answer")]

    IDR -->|"full DD payload, every submit"| SyncExch
    SyncExch -->|"transformed, GroupId assigned"| Kyc
    Kyc -->|"delta or full, per §10-Verified-Store"| SyncExch
    SyncExch -->|"POST dueDiligence — full-replace"| IDR
```

---

## Phase C — Application Architecture

### Application Portfolio Catalog

Every deployable system in scope — the flat inventory everything else references.

| Application | Repo | Language/Runtime | Role |
|---|---|---|---|
| IDR | `IDR` | .NET (legacy) | System of record being strangled out |
| TemplateAPI | `TemplateAPI` | .NET 8/10, modular monolith | Target platform (IKYC, Fund, TransferAgency, RulesEngine) |
| ManagedServices | `ManagedServices` | .NET | MKYC engine, analyst-driven |
| Sync | `Sync` | .NET | Generic outbound proxy (Rebuild→Legacy) |
| SyncExchange | `SyncExchange` | Python | Bidirectional transform layer (`-tl`/`-tr`) |
| SonataOneSecurity | `SonataOneSecurity` | .NET | Roles, permissions, security tree |
| Identity | `Identity` | .NET | Auth |
| ApiGateway | `ApiGateway` | .NET | Ingress |
| Notification | `Notification` | .NET | Notifications |
| PublicAPI | `PublicAPI` | .NET | External-facing API |
| Subscription | `Subscription` | .NET | Subscription mgmt |

### Application Communication Diagram

How the systems actually talk to each other today — reuses the trace already built in `08-Sync-Strategy...md` and `10-Verified-Store-Proposal.md §6`, this is the single most load-bearing diagram this initiative has produced so far, because it's grounded in exact file/line evidence rather than an assumed architecture.

```mermaid
flowchart LR
    IDR["IDR"] <-->|"Event Grid + REST"| Sync["Sync\n(generic proxy)"]
    Sync <-->|"REST"| SyncExch["SyncExchange\n(tl / tr)"]
    SyncExch <-->|"REST + direct DB"| Kyc["S1.Module.Kyc"]
    ManagedServices["ManagedServices"] -->|"IInternalServiceClient"| IDR
    ManagedServices -->|"IInternalServiceClient"| Kyc
    Kyc <-->|"IInternalServiceClient"| RulesEngine["S1.Module.RulesEngine"]
    Kyc <-->|"IInternalServiceClient"| Fund["S1.Module.Fund"]
```

---

## Phase D — Technology Architecture

### Technology Portfolio Catalog

Kept deliberately thin until Phase D is actually walked in full — placeholder structure only:

| Layer | Technology | Notes |
|---|---|---|
| Runtime | .NET 8/10 | Primary platform stack |
| Runtime | Python | SyncExchange only |
| Data | SQL Server | Confirmed via direct schema reads across IDR/TemplateAPI |
| Messaging | Azure Event Grid | Confirmed — `infrastructure/aeg/appSubscriptions.json` |
| Caching | Redis | Confirmed — `RedisCachedQuery<T>` in `User` module |
| Hosting | Docker (confirmed for Sync, SyncExchange) | IDR/TemplateAPI hosting model not yet confirmed |

---

## Phase E/F — Opportunities & Migration

### Gap Analysis (matrix)

Baseline vs. target, one row per capability, status pulled directly from findings already made this session rather than re-assessed.

| Capability | Baseline (today) | Target | Gap | Source |
|---|---|---|---|---|
| DD Questionnaire sync | Bidirectional, no ownership rule, confirmed echo loop | Unidirectional per-field ownership | Open | `08-Sync-Strategy...md` |
| Grouped-answer reconciliation | Fixed for sync path (`SubmitAnswersHandler`) | Same for `SyncExchange`'s GroupId bug | Partially closed | `10-Verified-Store-Proposal.md §6-7` |
| Lookup service coupling | RulesEngine depended on Kyc's concrete `LookupService` | Decoupled via `S1.Shared.Common` | Closed (Phase 0 done) | This session's implementation work |
| Multi-standard questionnaire assignment | Code exists (merge logic) but unreachable — no path creates a 2nd distinct visa | Either remove dead code or build the missing trigger | Open, undecided | `12-Profile-Visa-...md §3a/§3b` |

---

## Tooling note

Every diagram in this initiative is mermaid, inline in markdown, same as [`01_Planninng/IDR-Rebuild/`](../../../01_Planninng/IDR-Rebuild/). No separate diagramming tool (Visio, Lucidchart, ArchiMate modeling tools) is in use — deliberate, not a gap: it keeps every artifact diffable, greppable, and co-located with the analysis that produced it. If this initiative is ever formally pitched, the one thing worth adding at that point is exporting the key diagrams to a presentation format — the source of truth should stay markdown+mermaid regardless.
