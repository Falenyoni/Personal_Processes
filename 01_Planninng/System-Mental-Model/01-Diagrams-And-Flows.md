# System Mental Model - Diagrams and Flows

Diagrams first, minimal text. For evidence, corrections, and deep investigation behind any of this, see `../IDR-Rebuild/`.

---

## 1. The landscape

```mermaid
flowchart TD
    IDR["IDR (Legacy)\none monolith"]
    MS["ManagedServices\n(MKYC)"]
    Sec["SonataOneSecurity"]

    subgraph SyncLayer["Sync boundary"]
        direction LR
        Sync["Sync"]
        SyncExch["SyncExchange"]
    end

    subgraph TemplateAPI["TemplateAPI - modular monolith"]
        direction LR
        Kyc["S1.Module.Kyc\n(IKYC)"]
        Fund["S1.Module.Fund"]
        TA["S1.Module.TransferAgency"]
        Rules["S1.Module.RulesEngine"]
    end

    IDR <--> SyncLayer
    SyncLayer <--> TemplateAPI
    MS --> IDR
    MS --> TemplateAPI
    TemplateAPI <--> Sec
    MS <--> Sec
```

Investors use TemplateAPI (self-service, IKYC). Analysts use IDR (their tools still live there) and manage counterparties via ManagedServices (MKYC). Data flows both ways through Sync/SyncExchange.

## 2. Who drives the process - IKYC vs MKYC

```mermaid
flowchart LR
    subgraph Actors["Who drives the process"]
        Investor["Investor\n(self-service)"]
        Analyst["Analyst / Ops team\n(managed service)"]
    end

    subgraph Modes["KYC Service Modes (Service enum)"]
        IKYC["IKYC / CDD\nService = 20 / 8\nInvestor fills own KYC"]
        MKYC["MKYC\nService = 18\nAnalyst fills KYC on behalf\nof a counterparty"]
    end

    subgraph Data["Shared KYC Data Model"]
        Profile["Profile / Entity"]
        DD["DueDiligenceProfile"]
        Evidence["Evidence / Documents"]
    end

    Investor -->|drives| IKYC
    Analyst -->|drives| MKYC
    IKYC --> Profile
    MKYC --> Profile
    Profile --> DD
    Profile --> Evidence
```

Same underlying Profile/data model either way - what differs is who's driving data entry, not what gets stored.

## 3. Core domain relationship

```mermaid
flowchart LR
    Profile["Profile\n(the KYC subject)"] -->|"1 : many"| Visa["Visa\n(entitlement/obligation)"]
    Visa -->|"many : 1"| Questionnaire["Questionnaire\n(structure + text)"]
    Questionnaire --> Sections["Sections / SubSections / Questions"]
    Profile -->|"1 : many"| Answer["Answer\n(a response, optionally per-Fund, per-Group)"]
    Sections -.->|"answered via"| Answer
```

A profile can hold several visas. Each visa points at exactly one questionnaire. Merging (when more than one visa is involved) combines *questions*, never questionnaires themselves.

## 4. Profile creation → visa assignment

```mermaid
sequenceDiagram
    participant IDR
    participant NewKyc as S1.Module.Kyc
    participant DB as Kyc DB

    IDR-->>NewKyc: sync: create profile
    NewKyc->>DB: INSERT Profile
    NewKyc->>DB: find Questionnaire for (ProfileType, Global standard)
    alt found exactly one
        NewKyc->>DB: INSERT Visa
    else zero or multiple found
        NewKyc-->>NewKyc: rollback - profile insert undone too
    end
```

Today, every profile gets exactly one visa, always Global standard. This is deliberate for now - see §7.

## 5. Answering questions

```mermaid
sequenceDiagram
    actor User
    participant NewKyc as S1.Module.Kyc
    participant DB as Kyc DB

    User->>NewKyc: open questionnaire
    NewKyc->>DB: load Questionnaire structure + Question text + existing Answers
    NewKyc-->>User: assembled questionnaire

    User->>NewKyc: submit answers
    NewKyc->>DB: load existing Answers for this profile
    NewKyc->>NewKyc: match/insert/update by (QuestionId, FundId, GroupId)
    NewKyc->>NewKyc: reconcile grouped answers - remove any no longer submitted
    NewKyc->>DB: save
```

One shared reconciliation step handles matching, updating, and removing - used by every answer-submission path.

## 6. Sync between IDR and NewKyc, today

```mermaid
flowchart LR
    IDR["IDR"] -->|"full DD payload, every submit"| SyncExch["SyncExchange"]
    SyncExch -->|"transformed"| NewKyc["S1.Module.Kyc"]
    NewKyc -->|"changed fields only"| SyncExch
    SyncExch -->|"POST - Legacy treats as full replace"| IDR
```

Legacy always sends its full current state. NewKyc's outbound leg today sends only what changed - a mismatch worth knowing about before relying on either side's data being complete.

## 7. Where the platform is headed - one visa per fund

```mermaid
flowchart TD
    subgraph Today["Today - Legacy's model, mirrored deliberately"]
        P1["Profile"] --> V1["One Visa\nalways Global standard"]
    end
    subgraph Future["Target - one visa per fund"]
        P2["Profile"] --> VA["Visa A\nstandard from Fund A's lead profile"]
        P2 --> VB["Visa B\nstandard from Fund B's lead profile"]
    end
    Today -.->|"deliberately deferred until Legacy is further retired"| Future
```

Legacy gives a profile one overall approval, pinned to the highest standard demanded by any connected fund (standards stack: US ⊆ Global). Rebuild's `Visa` model exists to remove that limitation - a profile connects to a fund, and that fund's lead profile's jurisdiction determines the standard for that specific visa. Not built yet, on purpose - keeping question sets identical to Legacy's for now keeps sync simple while both systems run.

---

Next: `02-Monolith-Today.md` - the same system, in more depth: what's in the Monolith, what's in the Rebuild, and the concrete missing pieces between them.
