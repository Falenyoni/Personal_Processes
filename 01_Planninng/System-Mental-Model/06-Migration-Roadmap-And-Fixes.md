# The Missing Pieces, Part 2 - Migration Roadmap and Concrete Fixes

How to actually get from here to there. Diagrams first, minimal text. Full evidence: `../IDR-Rebuild/04-Target-Solution-And-Database-Structure.md`, `05-Migration-Strategy.md`, `06-Glossary-BuyButton-And-ServiceLevels.md`, `08-Sync-Strategy-ETL-Vs-Unidirectional-Cutover.md`, `10-Verified-Store-Proposal.md`.

---

## 1. IDR schema → new service mapping

```mermaid
graph LR
    subgraph IDR["IDR (single SQL Server DB, verified schemas)"]
        dbo["dbo\n(Entity, Profile, DueDiligence*, Billing, AdminTask)"]
        audit1["audit"]
        security1["security"]
        sync1["sync"]
        search1["search"]
        reports1["reports / customreports / export"]
        tasks1["tasks"]
        toolsmig["migration / tools"]
    end

    subgraph New["New services"]
        TAK["TemplateAPI: Kyc, Operations schemas"]
        TAC["TemplateAPI: Compliance schema (new)"]
        TAB["TemplateAPI: Billing schema (future)"]
        SOS["SonataOneSecurity: SonataOne.Security.Database"]
        SYN["Sync: SonataOne.Sync.Database"]
        SCR["SonataOneScreening: Screening schema"]
        REP["TemplateAPI: Reports schema (new)"]
    end

    dbo -->|Entity/Profile/DueDiligence| TAK
    dbo -->|FATCA/CRS/Classification| TAC
    dbo -->|Billing/FixedFee/Ledger| TAB
    dbo -->|AdminTask*| TAK
    dbo -->|WorldCheck*| SCR
    security1 --> SOS
    sync1 --> SYN
    reports1 --> REP
    audit1 -.->|"each service owns its own audit"| New
    toolsmig -.->|decommission post-migration| toolsmig
```

## 2. Migration approach - schema-first, code follows

```mermaid
flowchart LR
    P1["Phase 1\nSchema duplication\n(parallel write)"] --> P2["Phase 2\nRead cut-over"]
    P2 --> P3["Phase 3\nWrite cut-over"]
    P3 --> P4["Phase 4\nDecommission\nIDR tables"]

    P1_1["New service creates own schema alongside IDR"]:::note --> P1
    P1_2["Sync writes to both schemas"]:::note --> P1
    P2_1["Verify data parity"]:::note --> P2
    P2_2["New service reads from new schema only"]:::note --> P2
    P3_1["Route writes for feature area to new service"]:::note --> P3
    P3_2["IDR becomes read-only consumer via API"]:::note --> P3
    P4_1["Drop old tables once IDR reads only via API"]:::note --> P4

    classDef note fill:#4a5568,color:#fff,stroke:none;
```

## 3. Event-driven communication, target state

```mermaid
sequenceDiagram
    actor Investor
    participant Kyc as S1.Module.Kyc
    participant Sync
    participant AA as S1.Module.AnalystAction
    participant Rules as RulesEngine
    participant Notif as Notification

    Investor->>Kyc: Submit KYC questionnaire
    Kyc->>Sync: QuestionnaireSubmittedEvent (outbox)
    Sync->>AA: TaskCreated (review task)
    Sync->>Rules: RiskRecalculated
    Sync->>Notif: EmailQueued (notify analyst)
```

## 4. Strangler-fig migration timeline

```mermaid
gantt
    dateFormat  YYYY-MM-DD
    title Strangler Fig Migration Timeline (indicative)
    axisFormat %b %Y

    section Foundation
    Platform infra (Security, Sync, Core, Gateway)   :done, f1, 2026-01-01, 2026-08-13
    ApiGateway routing table finalized                :active, f2, 2026-08-13, 60d

    section Investor Journey
    S1.Module.Kyc (IKYC) parity          :active, i1, 2026-01-01, 2026-11-01
    Dual-write + parity validation       :i3, 2026-09-01, 150d

    section Analyst Journey
    ManagedServices (MKYC)               :active, a1, 2026-06-01, 2026-12-01
    S1.Module.Onboarding (new)           :a5, 2026-09-01, 120d
    S1.Module.Admin (task mgmt)          :a2, 2026-09-01, 120d
    S1.Module.Compliance (new)           :a3, 2026-10-01, 150d
    KycCore link: LinkedProfile -> Profile :crit, a4, 2026-09-01, 45d

    section Fund Manager Journey
    S1.Module.Fund / TransferAgency parity :active, fm1, 2026-01-01, 2027-01-01
    S1.Module.Compliance (FATCA/CRS)       :fm2, 2026-11-01, 150d
    S1.Module.Reporting (new)              :fm3, 2027-01-01, 120d

    section Decommission
    IDR read-only cutover per domain       :d1, 2027-02-01, 180d
    IDR decommission                       :d2, 2027-06-01, 90d
```

```mermaid
flowchart LR
    Phase1["Phase 1\nFoundation\n(now)"] --> Phase2["Phase 2\nInvestor journey parity\n(+3-6mo)"]
    Phase2 --> Phase3["Phase 3\nAnalyst + Fund Manager\njourney parity\n(+6-12mo)"]
    Phase3 --> Phase4["Phase 4\nDecommission IDR\n(+12-18mo)"]
```

## 5. Team structure & ownership

```mermaid
graph TB
    Platform["PLATFORM TEAM\nSonataOne.Core, SonataOneSecurity,\nSync, ApiGateway"]
    Investor["INVESTOR TEAM\nS1.Module.Kyc\nS1.Module.IdPal\nS1.Module.DocuSign"]
    AnalystFM["ANALYST / FUND MANAGER TEAM\nManagedServices (MKYC)\nS1.Module.Onboarding\nS1.Module.Admin\nS1.Module.AnalystAction\nS1.Module.Compliance\nS1.Module.Fund\nS1.Module.TransferAgency\nS1.Module.Reporting"]

    Platform --> Investor
    Platform --> AnalystFM
    AnalystFM -. "Onboarding invitation hand-off" .-> Investor
    Investor -. "Kyc read contract" .-> AnalystFM
```

`ManagedServices` already exists as a going concern with its own repo, CI/CD, and cadence - treat it as an existing team asset to extend, not a greenfield deliverable.

## 6. Why a counterparty can't hold ownership/control data yet

```mermaid
flowchart LR
    subgraph IDR["IDR (always could)"]
        Entity1["Entity\n(corporate counterparty)"]
        Entity2["Entity\n(its director/UBO)"]
        Entity1 -- "Relationship type 63\nCounterpartyReceivingKYC" --> Entity3["Entity\n(SonataOne/analyst side)"]
        Entity2 -- "Relationship type 15/16\nDirectorOrController" --> Entity1
    end
    subgraph New["New platform (blocked today)"]
        CP["CounterpartyProfile\n(ManagedServices)"]
        LP["LinkedProfile — stub,\nno typed FK"]
        Prof["Profile\n(S1.Module.Kyc)"]
        Conn["Connection\n(ownership/control)"]
        CP -.-> LP
        LP -.->|"not wired"| Prof
        Prof --- Conn
        CP -. "cannot reach Connection\nuntil LinkedProfile is fixed" .-> Conn
    end
```

IDR's single `Entity`/`Relationship` model always let one Entity hold a `CounterpartyReceivingKYC` tag *and* a `DirectorOrController` relationship simultaneously - a corporate counterparty could always have its own recorded directors/UBOs. In the new platform, `CounterpartyProfile` (ManagedServices) and `Profile` (S1.Module.Kyc) are structurally different types, so this isn't possible until `LinkedProfile` gets a real typed link - a functional blocker for MKYC due-diligence completeness, not just a data-hygiene concern.

## 7. The actual fix for sync: unidirectional, per-domain ownership

```mermaid
graph LR
    subgraph Before["Before cutover — IDR owns this domain"]
        IDR1["IDR"]:::writer
        New1["S1.Module.Kyc"]:::reader
        IDR1 -- "writes" --> Data1[("Date of Birth /\nDate of Incorporation")]
        Data1 -- "read-only,\nprojected via sync" --> New1
    end

    subgraph After["After cutover — S1.Module.Kyc owns this domain"]
        IDR2["IDR"]:::reader
        New2["S1.Module.Kyc"]:::writer
        New2 -- "writes" --> Data2[("Date of Birth /\nDate of Incorporation")]
        Data2 -- "read-only,\nprojected via sync" --> IDR2
    end

    Before -. "explicit cutover event\nper field/domain" .-> After

    classDef writer fill:#2b6cb0,color:#fff,stroke:none;
    classDef reader fill:#4a5568,color:#fff,stroke:none;
```

The rule: at any point in time, exactly one system owns the write for a given field. The other is read-only, kept current via one-directional projection. This makes the conflict-resolution question disappear by construction - there's no "who wins" when only one side can write.

## 8. Applying the cutover concretely

```mermaid
sequenceDiagram
    participant Analyst
    participant NewKyc as S1.Module.Kyc
    participant Bus as Sync (one-directional now)
    participant IDR as IDR (read-only for this field)

    Note over NewKyc,IDR: Date of Birth / Date of Incorporation formally cut over:<br/>S1.Module.Kyc is now the sole writer

    Analyst->>NewKyc: Submit new date
    NewKyc->>NewKyc: Kyc.Answer upsert<br/>(bug fixed: proper match key + unique DB constraint)
    NewKyc-. AnswerUpdatedEvent .->Bus
    Bus-.->IDR: Project into DueDiligenceProfile (read-only apply)

    Note over IDR: IDR never writes this field again.<br/>Any inbound "kyc.questionnaires.submit" for this<br/>field is rejected by NewKyc as not-my-source-of-truth.<br/>No reverse event ever gets applied — no echo loop possible.
```

Once cut over, IDR's copy becomes read-only, populated only by one-way events, and `S1.Module.Kyc` refuses to accept incoming changes for it from the sync inbox - closing the echo loop from `04-Sync-And-Known-Defects.md §3` without needing any correlation-ID plumbing for that field.

## 9. Interim fix (before full cutover): the Verified Store, reshaping the read side

```mermaid
graph LR
    subgraph Today["Today — SyncDataOutboxMessage"]
        Row1["Row: Created, Profile #42"]:::ev
        Row2["Row: Updated, Answer #883"]:::ev
        Row3["Row: Updated, Answer #883"]:::ev
        Note1["Each row = one event.<br/>No row answers 'what is true right now.'"]
    end
    subgraph Proposed["Proposed — Verified Store"]
        Doc["One row per GlobalId,<br/>upserted on every owned change,<br/>version + last-sent correlation IDs"]:::doc
    end
    Today -. "same DTOs, different lifecycle" .-> Proposed
    classDef ev fill:#4a5568,color:#fff,stroke:none;
    classDef doc fill:#2b6cb0,color:#fff,stroke:none;
```

Not a green-field idea - `S1.Module.Kyc` already has almost every ingredient (the same sync DTOs), just organized as a queue instead of a store. The proposal: upsert one row per `GlobalId` on every owned change, instead of only ever appending new event rows.

## 10. How the Verified Store closes the echo loop, without touching IDR

```mermaid
sequenceDiagram
    actor User
    participant NewKyc as S1.Module.Kyc
    participant Store as Verified Store (internal to NewKyc)
    participant Proxy as Sync proxy (unchanged)
    participant IDR as IDR: DueDiligenceController (unchanged)

    User->>NewKyc: Edit Date of Birth
    NewKyc->>Store: Upsert profile.dateOfBirth,<br/>record correlationId = C1, version+1
    NewKyc-->>Proxy: saga.investor-services.questionnaires.submit (correlationId C1)
    Proxy->>IDR: POST entities/{id}/dueDiligence (unchanged, verbatim forward)
    IDR-->>IDR: DueDiligenceProfile.DateOfBirth updated correctly
    IDR-->>NewKyc: saga.kyc.questionnaires.submit (re-broadcast, still carries correlationId C1)
    NewKyc->>Store: Look up lastOutboundCorrelationIds["profile.dateOfBirth"]
    Note over NewKyc: Incoming correlationId C1 matches what I just sent<br/>for this exact field → this is my own echo, drop it.
    NewKyc-->>NewKyc: No-op — edit is NOT overwritten
```

Fixes the confirmed Date of Birth echo loop with zero IDR changes - NewKyc simply stops reapplying its own round-tripped write. A genuine concurrent edit made directly in IDR would carry IDR's own correlation ID, not `C1`, so it still applies normally. This is an interim fix for the echo specifically - it doesn't replace the full unidirectional cutover in §7-§8, and it never carries permission/role data (that stays live in each system's own auth layer, per the cross-user cache exposure risk in `03-Rebuild-Today.md §8`).

---

## Summary - the honest gap list

| Gap | Where it's tracked | Priority signal |
|---|---|---|
| `LinkedProfile` has no real typed link to `S1.Module.Kyc.Profile` | §6 above | Functional blocker for MKYC ownership/control data, not just hygiene |
| No conflict resolution / loop detection on bidirectional sync | `04-Sync-And-Known-Defects.md §1, §3` | Confirmed live bug (DOB echo loop) |
| Tax Residence / Source of Funds / Wealth / Sensitive Activities duplication | `04-Sync-And-Known-Defects.md §5` | Confirmed, precisely traced; fixes designed, one not yet committed |
| No `Compliance`, `Onboarding`, or `Reporting` module in TemplateAPI | `05-Target-Architecture.md §1, §6` | Full gap - IDR's `Admin`-namespaced controllers are the only implementation today |
| Service-tier SLA granularity regresses vs. IDR (tier-only, not tier × task-type) | `../IDR-Rebuild/06-Glossary-BuyButton-And-ServiceLevels.md §2` | Real regression risk if shipped as-is |
| "Fund Manager journey" self-service status unconfirmed | `../IDR-Rebuild/02-Journey-Scoping.md §6` | Affects Business Architecture and this whole roadmap's Fund Manager sections |

This is the honest state as of the source investigation - not a committed plan. See `../IDR-Rebuild/README.md` for the full documents these diagrams were drawn from, and `../../2027_Plan/Enterprise_Architecture/Apex-SonataOne-Initiative/` for how this maps onto TOGAF/Zachman.
