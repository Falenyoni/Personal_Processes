# What's in the Monolith (IDR) - and How It Happens

Diagrams first, minimal text. Full evidence and file/line citations: `../IDR-Rebuild/01-KYC-Explained-And-Access-Control.md` and `../IDR-Rebuild/02-Journey-Scoping.md`.

---

## 1. Three journeys, one monolith

```mermaid
graph TD
    subgraph Investor["INVESTOR JOURNEY"]
        I1["Self-service KYC (IKYC)"]
        I2["Document / evidence upload"]
        I3["Questionnaires"]
        I5["Profile management"]
        I6["Declarations, e-signing"]
    end

    subgraph Analyst["ANALYST JOURNEY"]
        A1["Review & approve"]
        A2["Risk assessment"]
        A3["Managed KYC (MKYC)"]
        A4["Screening alerts (WorldCheck)"]
        A5["Task management"]
        A6["Escalations"]
        A7["FATCA/CRS filing"]
        A8["Onboarding - bulk data\nimport/staging + invitations"]
    end

    subgraph FundMgr["FUND MANAGER JOURNEY"]
        F1["Fund setup & config"]
        F2["Investor relationships"]
        F3["Subscription management"]
        F4["Transfer agency ops"]
        F5["Reporting & compliance"]
        F6["Capital account mgmt"]
        F7["Fee management"]
    end

    Investor -. "submits" .-> Analyst
    Analyst -. "escalates / reports to" .-> FundMgr
    FundMgr -. "configures rules that gate" .-> Investor
```

IDR's 300+ controllers organize around what Investor, Analyst, and Fund Manager are each trying to accomplish - not around a table list. Whether "Fund Manager" is really a self-service actor, or work Apex staff perform on their behalf, is an open, unresolved question - see `../IDR-Rebuild/02-Journey-Scoping.md §6`.

## 2. Platform capabilities every journey shares

```mermaid
graph TB
    subgraph Platform["Platform (shared, journey-agnostic)"]
        Sec["SonataOneSecurity\n(identity/permissions)"]
        Sync["Sync\n(outbox/saga messaging)"]
        Notif["Notification"]
        Rules["RulesEngine"]
        Screen["SonataOneScreening"]
        Report["Reporting"]
    end

    InvJourney["Investor Journey"] --> Platform
    AnaJourney["Analyst Journey"] --> Platform
    FMJourney["Fund Manager Journey"] --> Platform
```

## 3. Who can actually be a KYC subject - not just "the investor"

```mermaid
graph TD
    KYC["KYC Subject\n(Profile / Entity)"]
    KYC --> Individual["Individual\n(private investor, beneficial owner)"]
    KYC --> Listed["Listed Entity\n(publicly listed company)"]
    KYC --> Regulated["Regulated Entity\n(bank, insurer)"]
    KYC --> PrivateCo["Private Company"]
    KYC --> LP["Limited Partnership\n(fund vehicle itself)"]
    KYC --> Trust["Trust"]
    KYC --> EBT["Pension / EBT / IRA"]
    KYC --> Uni["University\n(endowment)"]
    KYC --> Gov["Public Body / Government"]
    KYC --> Found["Foundation / Non-Profit"]
    KYC --> SWF["Sovereign Wealth Entity"]
    KYC --> LLC["Limited Liability Company"]
    KYC --> Joint["Joint Account"]
```

13 possible KYC subjects, one of which is "Individual investor." A fund vehicle, a trust, or a counterparty entity managed entirely by ops staff can equally be the thing being KYC'd.

## 4. IKYC - investor-driven, self-service

```mermaid
stateDiagram-v2
    [*] --> NOT_STARTED
    NOT_STARTED --> IN_PROGRESS: investor opens questionnaire
    IN_PROGRESS --> PARKED: investor pauses / missing info
    PARKED --> IN_PROGRESS: investor resumes
    IN_PROGRESS --> COMPLETED: all questions + evidence submitted
    COMPLETED --> [*]
```

```mermaid
sequenceDiagram
    actor Investor
    participant Portal as Investor Portal
    participant DD as DueDiligenceController (IDR)
    participant Auth as AuthorizationService
    participant Chaser as IKYCChaserEmail job

    Investor->>Portal: Log in, open KYC
    Portal->>DD: GET /entities/{id}/dueDiligence
    DD->>Auth: AuthorizeOrReadOnlyKYCOrManagedKyc(investorId, entityId)
    Auth-->>DD: Full (investor owns this entity)
    DD-->>Portal: Editable profile + questionnaire
    Investor->>Portal: Fill answers, upload evidence
    Portal->>DD: PUT answers / evidence
    Note over Chaser: If incomplete after N days
    Chaser-->>Investor: Reminder email
```

## 5. MKYC - analyst-driven, on behalf of a counterparty

```mermaid
stateDiagram-v2
    [*] --> NOT_STARTED
    NOT_STARTED --> IN_PROGRESS: analyst begins managed KYC
    IN_PROGRESS --> AWAITING_CLIENT_UPDATE: analyst requests info
    AWAITING_CLIENT_UPDATE --> WITH_COUNTERPARTY: sent to counterparty for response
    WITH_COUNTERPARTY --> QUESTION_PENDING_WITH_IDR: counterparty raises a query back
    QUESTION_PENDING_WITH_IDR --> WITH_COUNTERPARTY: query answered
    WITH_COUNTERPARTY --> IN_PROGRESS: counterparty responds, analyst continues
    IN_PROGRESS --> COMPLETED: certificate issued
    COMPLETED --> [*]
```

```mermaid
sequenceDiagram
    actor Analyst
    actor Counterparty as Counterparty (client, no portal login)
    participant Dash as ManagedKycDashboard
    participant DD as DueDiligenceController (IDR)
    participant Auth as AuthorizationService
    participant Cert as ManagedKycCertificate

    Analyst->>Dash: Open assigned MKYC case
    Dash->>DD: GET /entities/{id}/dueDiligence
    DD->>Auth: AuthorizeOrReadOnlyKYCOrManagedKyc(analystId, entityId)
    Auth-->>DD: ReadOnly or Full via HasManagedKYCPermissionFunction
    DD-->>Dash: Profile + questionnaire (analyst-editable)
    Analyst->>Dash: Fill known data, flag missing items
    Dash->>Counterparty: Bulk chaser email (MkycBulkChaserEmail)
    Counterparty-->>Dash: Confirms / provides documents (offline or portal-lite)
    Analyst->>Dash: Mark complete
    Dash->>Cert: Issue ManagedKycCertificate
```

IKYC and MKYC run through the *same* `DueDiligenceController` and the *same* permission method in IDR, despite having different actors, state machines, and SLAs - the root of that controller's complexity, and the first thing the Rebuild split apart (see `03-Rebuild-Today.md §1`).

## 6. The permission model, as a decision

```mermaid
flowchart TD
    Start["Analyst requests profile X"] --> Full{"Full permission\non entity X?"}
    Full -->|Yes| GrantFull["Access: Full\n(view + edit)"]
    Full -->|No| RO{"Read-Only KYC\npermission granted?"}
    RO -->|Yes| GrantRO["Access: Read-only"]
    RO -->|No| MK{"Managed KYC\npermission on entity X?"}
    MK -->|Yes| GrantMK["Access: Read-only\n(MKYC role)"]
    MK -->|No| Deny["403 Access Denied"]
```

Same decision tree backs IKYC, MKYC, and CDD endpoints alike: explicit grant on that entity, or denied. No "view all" role exists anywhere in IDR.

---

Next: `03-Rebuild-Today.md` - the same domain, as TemplateAPI and ManagedServices actually build it today.
