# The Missing Pieces, Part 1 - Target Architecture

What a clean-slate design looks like, and where the actual build already agrees with it. Diagrams first, minimal text. Full evidence: `../IDR-Rebuild/03-Greenfield-Architecture.md` and `07-Independent-Rebuild-Recommendation.md`.

---

## 1. Bounded context map

```mermaid
graph TB
    subgraph IdentityCtx["Identity & Access"]
        Users["Users"]
        Roles["Roles"]
        Perms["Permissions"]
        Rels["Relationships"]
    end

    subgraph OnboardingCtx["Onboarding"]
        Invite["Invitation"]
        Sessions["Sessions"]
        CddImport["CDD import"]
        Stages["Stages"]
    end

    subgraph ComplianceCtx["Compliance"]
        KycEngine["KYC Engine\n(IKYC / MKYC / CDD)"]
        FatcaCrs["FATCA / CRS"]
        DueDil["Due Diligence scheduling"]
        Screening["Screening (AML/WorldCheck)"]
    end

    subgraph FundCtx["Fund Management"]
        FundConfig["Fund config"]
        TA["Transfer Agency"]
        CapAcct["Capital Accounts"]
    end

    subgraph InvestorCtx["Investor Portal"]
        Profile["Profile"]
        Evidence["Evidence / Documents"]
        Questionnaire["Questionnaire"]
        Sign["DocuSign"]
    end

    subgraph OpsCtx["Operations"]
        Tasks["Task management"]
        AnalystWf["Analyst workflow / MKYC"]
        RiskEngine["RulesEngine / risk scoring"]
    end

    subgraph PlatformCtx["Platform Services"]
        Sync["Sync / messaging"]
        Notif["Notification"]
        Report["Reporting"]
        Gateway["API Gateway"]
    end

    OnboardingCtx --> IdentityCtx
    InvestorCtx --> ComplianceCtx
    OpsCtx --> ComplianceCtx
    FundCtx --> ComplianceCtx
    ComplianceCtx --> PlatformCtx
    InvestorCtx --> PlatformCtx
    OpsCtx --> PlatformCtx
    FundCtx --> PlatformCtx
```

## 2. KYC split into three cooperating services, not one module

```mermaid
graph TB
    subgraph KycCore["B1. KycCore Service — the shared data model"]
        Profile2["Profile (any EntityType)"]
        DD2["DueDiligence (per profile, per standard)"]
        Ev2["Evidence + EvidenceCertification"]
        SvcSub["ServiceSubscription (which services a profile is enrolled in)"]
        Risk2["RiskAssessment"]
    end

    subgraph InvestorKyc["B2. InvestorKyc Service (IKYC) — self-service"]
        Portal2["Investor-facing questionnaire UI API"]
        Flow2["Multi-step questionnaire flow"]
        Upload2["Evidence upload (investor-driven)"]
        Invite2["Invitation → register → onboard"]
        Chaser2["IKYC chaser schedule"]
        IdV2["ID verification (IdPal)"]
    end

    subgraph ManagedKyc["B3. ManagedKyc Service (MKYC) — ops-managed"]
        Dash3["Managed KYC dashboard"]
        Counterparty3["Counterparty request management"]
        Cert3["Certificate issuance"]
        BulkChaser3["Bulk chaser scheduling"]
        Workflow3["State machine:\nNOT_STARTED → WITH_COUNTERPARTY → COMPLETED"]
    end

    InvestorKyc -->|reads/writes via API| KycCore
    ManagedKyc -->|reads/writes via API| KycCore
```

`InvestorKyc` is realized today as `S1.Module.Kyc`; `ManagedKyc` as the standalone `ManagedServices` repo - **this split already happened**, independently, ahead of this reasoning. `KycCore` is the one piece not yet built (§5 below).

## 3. Database strategy - one schema per bounded context, no exceptions

```mermaid
graph LR
    Identity["Security schema"]:::db
    KycCore2["Kyc schema"]:::db
    InvKyc["InvestorPortal schema"]:::db
    MgdKyc["ManagedKyc schema"]:::db
    Compl["Compliance schema"]:::db
    Fund2["Fund schema"]:::db
    Invest["Investment schema"]:::db
    Ops2["Operations schema"]:::db
    RiskDb["Risk schema"]:::db
    ReportDb["Reports schema"]:::db
    SyncDb["Sync schema"]:::db

    classDef db fill:#2b6cb0,color:#fff,stroke:none;
```

A service owns its schema and is the only writer. Everyone else reads via API, never a direct DB join.

## 4. Event contracts, target state

```mermaid
sequenceDiagram
    participant KycCore
    participant InvestorKyc
    participant ManagedKyc
    participant RulesEngine
    participant AnalystWf as AnalystWorkflow
    participant Notif as Notification
    participant Compliance

    InvestorKyc->>KycCore: InvestorRegisteredEvent
    KycCore->>AnalystWf: ProfileCreatedEvent
    InvestorKyc->>KycCore: IkycCompletedEvent
    KycCore->>AnalystWf: KycSubmittedEvent (review task created)
    KycCore->>RulesEngine: (profile data changed)
    RulesEngine-->>KycCore: RiskCalculatedEvent
    KycCore->>AnalystWf: RiskLevelChangedEvent (escalation check)
    KycCore->>Notif: EvidenceUploadedEvent (certifier notified)
    ManagedKyc->>Notif: MkycRequestCreatedEvent (counterparty invited)
    ManagedKyc-->>ManagedKyc: CounterpartyRespondedEvent (state advanced)
    ManagedKyc->>KycCore: MkycCertificateIssuedEvent (ServiceSubscription complete)
    Compliance->>Notif: FatcaClassificationChangedEvent
    Compliance->>AnalystWf: ScreeningMatchFoundEvent (escalation task)
```

## 5. The one real gap - two independent subject records instead of one shared core

```mermaid
graph TB
    subgraph Today["Current state — two independent subject records"]
        direction LR
        KycProfile["S1.Module.Kyc.Profile\n(IKYC subject)"]
        CPProfile["ManagedServices.CounterpartyProfile\n+ LinkedProfile (stub)\n(MKYC subject)"]
    end
    subgraph Target["Recommended — one shared subject record"]
        direction LR
        Core2["KycCore.Profile\n(shared, either service reads/writes via API)"]
        Kyc2["S1.Module.Kyc\n(IKYC surface)"]
        MS2["ManagedServices\n(MKYC surface)"]
        Kyc2 --> Core2
        MS2 --> Core2
    end
    Today -.->|"recommended refactor:\ngive LinkedProfile a real typed link,\nor extract KycCore"| Target
```

Everything else in the greenfield design (compliance, chasers, event-driven-by-default) is a full gap - not yet built anywhere. This one is the highest-value fix: without it, the same real-world counterparty can silently drift into two different records across the two services.

## 6. The full independent service map, judged with no reference to what's already built

```mermaid
graph TB
    Gateway["ApiGateway"]

    subgraph Identity["Identity"]
        IdentityAccess["IdentityAccess"]
    end

    subgraph KycDomain["KYC Domain"]
        KycCore["KycCore"]
        InvestorKyc["InvestorKyc"]
        ManagedKyc["ManagedKyc"]
    end

    subgraph ComplianceDomain["Compliance Domain"]
        Compliance["Compliance"]
        Screening["Screening"]
    end

    subgraph FundDomain["Fund Domain"]
        FundManagement["FundManagement"]
        TransferAgency["TransferAgency"]
        SubscriptionDocuments["SubscriptionDocuments"]
    end

    subgraph OpsDomain["Operations"]
        Operations["Operations"]
        RulesEngine["RulesEngine"]
        Onboarding["Onboarding\n(staff-driven bulk import + invite)"]
    end

    subgraph SharedCapabilities["Shared Capabilities"]
        DocumentManagement["DocumentManagement"]
        ESignature["ESignature"]
        IdVerification["IdVerification"]
        Notification["Notification"]
        Reporting["Reporting"]
    end

    Bus(["Sync / Event Bus"])

    Gateway --> InvestorKyc
    Gateway --> ManagedKyc
    Gateway --> FundManagement
    Gateway --> TransferAgency
    Gateway --> SubscriptionDocuments
    Gateway --> Operations
    Gateway --> Onboarding
    Gateway --> Reporting

    IdentityAccess -.auth.- Gateway
    IdentityAccess -.authz.- KycCore
    IdentityAccess -.authz.- ManagedKyc
    IdentityAccess -.authz.- Operations

    InvestorKyc -->|"read/write Profile, DueDiligence"| KycCore
    InvestorKyc -->|"verify identity"| IdVerification
    InvestorKyc -->|"sign declarations"| ESignature
    ManagedKyc -->|"read/write counterparty Profile,\nrecord ownership/control via Connection"| KycCore
    ManagedKyc -->|"issue certificate"| ESignature

    Onboarding -->|"stage bulk-imported\nProfile/DueDiligence data"| KycCore
    Onboarding -->|"create user account\nfor invited investor"| IdentityAccess
    Onboarding -->|"validate staged batch"| RulesEngine

    KycCore -->|"is this profile screened?"| Screening
    Compliance -->|"sanctions check"| Screening
    Compliance -->|"reportability rules"| RulesEngine

    TransferAgency -->|"fund config"| FundManagement
    TransferAgency -->|"confirm KYC/qualification status\nbefore allowing subscription"| KycCore
    SubscriptionDocuments -->|"pre-fill fields from KYC data"| KycCore
    SubscriptionDocuments -->|"e-sign subscription docs"| ESignature
    TransferAgency -->|"attach documents"| DocumentManagement
    SubscriptionDocuments -->|"attach documents"| DocumentManagement

    Operations -->|"task context"| KycCore
    Operations -->|"task context"| ManagedKyc
    Operations -->|"task context"| Compliance

    KycCore -. events .-> Bus
    InvestorKyc -. events .-> Bus
    ManagedKyc -. events .-> Bus
    RulesEngine -. events .-> Bus
    Compliance -. events .-> Bus
    TransferAgency -. events .-> Bus
    Onboarding -. "InvitationSentEvent" .-> Bus
    Bus -. "task created,\nescalation" .-> Operations
    Bus -. "email triggers" .-> Notification
    Bus -. "risk recalculation" .-> RulesEngine
    Bus -. "denormalized projections" .-> Reporting
    Bus -. "invitation accepted\n-> begin questionnaire" .-> InvestorKyc
```

Solid arrows = synchronous, a real runtime dependency. Dashed = async, via the event bus. This independent exercise and the actual build agree on more than they disagree on - the main substantive gap is the same `KycCore` unification called out in §5.

## 7. End-to-end target flow: staff onboarding → investor self-service KYC → subscription

```mermaid
sequenceDiagram
    actor Analyst
    actor Investor
    participant Onb as Onboarding
    participant Gateway as ApiGateway
    participant IdA as IdentityAccess
    participant IKyc as InvestorKyc
    participant Core as KycCore
    participant IdV as IdVerification
    participant Rules as RulesEngine
    participant SubDocs as SubscriptionDocuments
    participant TA as TransferAgency
    participant ESign as ESignature
    participant Bus as Event Bus
    participant Ops as Operations
    participant Notif as Notification

    Analyst->>Onb: Stage bulk-imported investor batch
    Onb->>Core: Create staged Profile + DueDiligence per record
    Onb->>Rules: Validate staged batch
    Analyst->>Onb: Confirm & trigger invitations
    Onb->>IdA: Create user account per invited investor
    Onb-. InvitationSentEvent .->Bus
    Bus-.->Notif: send invitation email

    Investor->>Gateway: Accept invitation, log in
    Gateway->>IdA: Resolve permissions
    Investor->>IKyc: Complete/confirm staged KYC questionnaire
    IKyc->>Core: Create/update Profile + DueDiligence
    IKyc->>IdV: Submit for identity verification
    IdV-->>IKyc: Verification result
    Core-. KycSubmittedEvent .->Bus
    Bus-.->Rules: recalculate risk
    Bus-.->Ops: create review task
    Investor->>SubDocs: Start subscription questionnaire
    SubDocs->>Core: Pre-fill fields from KYC data
    Investor->>TA: Submit commitment
    TA->>Core: Confirm KYC/qualification status (sync, blocking)
    Core-->>TA: Qualified
    TA->>SubDocs: Finalize subscription document
    SubDocs->>ESign: Send for signature
    ESign-->>Investor: Sign
    TA-. SubscriptionCreatedEvent .->Bus
    Bus-.->Ops: create fund-ops onboarding task
```

## 8. End-to-end target flow: analyst-managed KYC (MKYC) engagement

```mermaid
sequenceDiagram
    actor Analyst
    actor Counterparty as Counterparty (no login)
    participant MKyc as ManagedKyc
    participant Core as KycCore
    participant Screen as Screening
    participant ESign as ESignature
    participant Notif as Notification
    participant Bus as Event Bus
    participant Ops as Operations

    Analyst->>MKyc: Open/create engagement (Project)
    MKyc->>Core: Create counterparty Profile + Connection (ownership/control)
    Core->>Screen: Screen counterparty and its owners/controllers
    Screen-->>Core: Screening result
    MKyc-. MkycRequestCreatedEvent .->Bus
    Bus-.->Notif: notify counterparty
    Notif-->>Counterparty: Request for information
    Counterparty-->>MKyc: Response (offline/portal-lite)
    Analyst->>MKyc: Mark complete
    MKyc->>ESign: Issue certificate
    MKyc-. MkycCertificateIssuedEvent .->Bus
    Bus-.->Core: ServiceSubscription complete
    Bus-.->Ops: close related tasks
```

Notice this target flow gives the counterparty a `Connection` (ownership/control graph) at engagement time - something the actual build cannot do today until the `LinkedProfile` gap (§5) closes. See `06-Migration-Roadmap-And-Fixes.md §7`.

---

Next: `06-Migration-Roadmap-And-Fixes.md` - how to actually get from here to there, and the specific fixes already designed for the defects in `04-Sync-And-Known-Defects.md`.
