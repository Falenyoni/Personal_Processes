# What's in the Rebuild (TemplateAPI + ManagedServices) - and How It Happens

Diagrams first, minimal text. Full evidence: `../IDR-Rebuild/01-KYC-Explained-And-Access-Control.md §6`, `06-Glossary-BuyButton-And-ServiceLevels.md`, `09-KYC-Profile-Caching-And-Cross-User-Exposure-Risk.md`, `12-Profile-Visa-And-Profile-Creation-Flow.md`, `13-Questionnaire-Merging-Standard-Awareness-And-Terminology.md`.

---

## 1. "New KYC" today - two repos, two actors, one gap between them

```mermaid
graph LR
    subgraph TemplateAPIRepo["TemplateAPI repo"]
        KycModule["S1.Module.Kyc\n(Profile/Connection/Questionnaire/\nEvidence/RiskAssessment)\n= IKYC data + self-service flow"]
        AA["S1.Module.AnalystAction\n(generic task/SLA tracking)"]
    end
    subgraph ManagedServicesRepo["ManagedServices repo (separate service)"]
        Project["Project\n(analyst-run engagement)"]
        CP["CounterpartyProfile / CounterpartyUser\n(the MKYC subject)"]
        CR["CounterpartyRequest\n(DealStatus: AwaitingClientUpdate →\nWithCounterparty → QuestionPendingWithS1 → Complete)"]
    end

    Project --> CR
    CR --> CP
    CR -.->|"LinkedProfile — not yet wired\nto S1.Module.Kyc Profile"| KycModule
    Project -.->|assigned analysts, reads legacy user/fund data via IInternalServiceClient| AA
```

IKYC lives in `TemplateAPI → S1.Module.Kyc`; MKYC lives in its own live, actively-developed repo, `ManagedServices` - not a stub, not a gap. The one real gap: the two don't share a "who is this KYC subject" record yet (`../IDR-Rebuild/06-Glossary-BuyButton-And-ServiceLevels.md §3`, `05-Missing-Pieces-And-Target-Architecture.md §5` below).

## 2. Where KYC sits relative to the three journeys

```mermaid
graph LR
    subgraph KycPlatform["KYC Platform (shared data + rules)"]
        Core["Profile / Questionnaire /\nEvidence / RiskAssessment"]
    end

    InvestorJ["Investor Journey"] -->|"drives IKYC\n(self as subject)"| Core
    AnalystJ["Analyst Journey"] -->|"drives MKYC\n(counterparty as subject)"| Core
    FMJ["Fund Manager Journey"] -->|"reads classification/\nrisk status for gating"| Core
```

KYC isn't a fourth journey - it's a capability Investor and Analyst each drive (differently), sitting on shared data the Fund Manager journey only reads.

## 3. Buy Button - a per-domain self-service investing mode

```mermaid
flowchart TD
    Req["Investor request arrives"] --> Host["Read X-Forwarded-Host header"]
    Host --> Check{"Is host in\nBUY_BUTTON_DOMAINS?"}
    Check -->|Yes| BB["iAmOnABuyButtonDomain = true"]
    Check -->|No| NotBB["iAmOnABuyButtonDomain = false"]

    BB --> LoggedIn{"Logged in?"}
    NotBB --> LoggedIn

    LoggedIn -->|"Yes, investor,\non Buy Button domain"| ProfileCheck["haveILoadedAProfile"]
    ProfileCheck --> PreQual["amIPreQualified"]
    PreQual -->|"pass"| Dashboard["Redirect straight to\nINVESTOR_DASHBOARD\n(self-service buy flow)"]
    PreQual -->|"fail"| Blocked["Blocked / redirected elsewhere\nuntil qualification met"]

    LoggedIn -->|"Yes, investor,\nnot on Buy Button domain"| LegacyApp["Redirect to legacy Angular app\n(NEXT_PUBLIC_ANGULAR_APP_BASE_URL)"]
```

Some fund-manager clients' domains enable a direct, self-service "buy into this fund" flow in the new Next.js portal; others still route their investors to the legacy Angular app. Today the new frontend effectively only serves Buy Button domains for investors.

## 4. Profile creation → Visa assignment, one transaction

```mermaid
sequenceDiagram
    participant IDR as IDR (Legacy)
    participant EP as SyncCreateProfileEndpoint
    participant H as SyncCreateProfileHandler
    participant VisaSvc as VisaService
    participant DB as Kyc DB

    IDR-->>EP: POST kyc/sync/profile (saga.kyc.profile.synchronize)
    EP->>H: SyncCreateProfileRequest
    H->>DB: SELECT Profile WHERE GlobalId = X
    alt Already exists
        H-->>EP: return existing profile (idempotent)
    else New
        H->>H: BuildProfile → Profile.CreateFromLegacySync(...)
        H->>DB: BEGIN TRANSACTION
        H->>DB: UpsertProfileAsync → INSERT Kyc.Profile
        H->>DB: RegisterProfileDetailsAsync
        H->>VisaSvc: AssignVisaForProfileAsync(profile)
        VisaSvc->>DB: SELECT Questionnaire WHERE ProfileTypeRef = profile.ProfileType AND QuestionnaireStandardRef = Global AND IsActive
        alt 0 matches
            VisaSvc-->>H: Fail NotFoundError
            H->>DB: ROLLBACK — profile insert undone too
        else 2+ matches
            VisaSvc-->>H: Fail (ambiguous)
            H->>DB: ROLLBACK — profile insert undone too
        else exactly 1 match
            VisaSvc->>DB: INSERT Kyc.Visa (ProfileIdRef, QuestionnaireIdRef)
            VisaSvc-->>H: Ok(visa)
            H->>DB: COMMIT
        end
    end
```

A failed visa match rolls back the profile insert too - a profile is never left half-created with no questionnaire attached.

## 5. The full flow, question to database

```mermaid
flowchart TD
    A["Analyst (bulk onboarding) OR\ninvestor (self-registration)\ncreates profile in IDR"] --> B["IDR publishes saga.kyc.profile.synchronize"]
    B --> C["POST kyc/sync/profile\nSyncCreateProfileEndpoint"]
    C --> D["SyncCreateProfileHandler.Handle"]
    D --> E{"Profile with this\nGlobalId already exists?"}
    E -->|Yes| F["Return existing profile\n(idempotent, no-op)"]
    E -->|No| G["BuildProfile\nProfile.CreateFromLegacySync(...)"]
    G --> H["BEGIN TRANSACTION"]
    H --> I["UpsertProfileAsync\nINSERT Kyc.Profile"]
    I --> J["RegisterProfileDetailsAsync"]
    J --> K["AssignVisaAsync\nVisaService.AssignVisaForProfileAsync"]
    K --> L{"Questionnaire WHERE\nProfileType = X AND\nStandard = Global AND IsActive"}
    L -->|0 found| M["Fail: NotFoundError\nROLLBACK whole transaction"]
    L -->|2+ found| N["Fail: ambiguous match\nROLLBACK whole transaction"]
    L -->|exactly 1| O["INSERT Kyc.Visa\n(ProfileIdRef, QuestionnaireIdRef)"]
    O --> P["COMMIT"]
    P --> Q["Profile now has a Visa →\na Questionnaire → questions available"]
    Q --> R["User/analyst opens questionnaire:\nQuestionnaireMergingService reads JsonConfig\n+ Question rows + existing Answers"]
    R --> S["Answers submitted"]
    S --> T{"Source?"}
    T -->|"NewKyc UI"| U["SubmitQuestionnaire /\nSubmitMergedQuestionnaireSection\n(validated against section config)"]
    T -->|"Legacy resync"| V["SubmitAnswers\n(no validation, machine-to-machine)"]
    U --> W["Match existing Answer by\n(QuestionIdRef, FundIdRef, GroupId)"]
    V --> W
    W --> X{"Match found?"}
    X -->|"Yes, value changed"| Y["UPDATE Kyc.Answer"]
    X -->|"Yes, unchanged"| Z["No-op"]
    X -->|"No match"| AA["INSERT Kyc.Answer"]
```

Either an analyst's bulk onboarding import *or* an investor self-registering (`IDRFrontend`'s `create-profile` page, still posting to IDR's `POST entities` with an authenticated token) creates the profile - both land in IDR first, never directly in `S1.Module.Kyc`. A separate, real `S1.Module.Kyc`-native `CreateProfile` endpoint does exist, but its confirmed use is adding an owner/controller to an existing profile's connection graph, not primary investor registration - see `../IDR-Rebuild/12-Profile-Visa-And-Profile-Creation-Flow.md §4a` for the full trace.

## 6. Merging questions across visas - and where the gap actually is

```mermaid
flowchart TD
    Profile["Profile (one)"]
    Profile -->|"1 : many"| Visa1["Visa #1\nQuestionnaireIdRef = 2"]
    Profile -->|"1 : many"| Visa2["Visa #2\nQuestionnaireIdRef = 15"]

    Visa1 -->|"many : 1"| Q1["Questionnaire #2\nGlobal Standard, Individual\nQuestionnaireStandardRef = 1"]
    Visa2 -->|"many : 1"| Q2["Questionnaire #15\nCBRE Standard, Individual\nQuestionnaireStandardRef = 5"]

    Q1 --> S1["Sections / SubSections / Questions"]
    Q2 --> S2["Sections / SubSections / Questions"]

    S1 --> Merge
    S2 --> Merge

    Merge["QuestionnaireMergingService.GetMergedQuestionnaireAsync\nDedupes by QuestionnaireIdRef only.\nNo check anywhere on QuestionnaireStandard or Fund."]

    Merge --> Composite["QuestionnaireDto&lt;QuestionDto&gt;\ntransient, in-memory only - never saved"]

    Q1 -.->|"row untouched"| DB[("Kyc.Questionnaire\nQ#2 and Q#15 remain separate rows")]
    Q2 -.->|"row untouched"| DB

    classDef warn fill:#c05621,color:#fff,stroke:none;
    classDef transient fill:#4a5568,color:#fff,stroke:none;
    classDef persisted fill:#2b6cb0,color:#fff,stroke:none;
    class Merge warn;
    class Composite transient;
    class DB persisted;
```

Illustrative, not observed - every real multi-visa profile found in production points both visas at the *same* questionnaire (a race-condition bug, not genuine multi-standard usage). The orange node is real code with no standard/fund check today - deliberately unfinished, not dead: it's built for the one-visa-per-fund target design (`05-Missing-Pieces-And-Target-Architecture.md`), which Rebuild is intentionally deferring while it mirrors Legacy's simpler one-standard-per-profile behavior for now.

## 7. How an answer actually lands in the database

```mermaid
sequenceDiagram
    participant Src as "User (SubmitQuestionnaire) OR Legacy (SubmitAnswers, unvalidated)"
    participant H as Answer handler
    participant DB as Kyc DB

    Src->>H: submitted answers (QuestionId, FundId, AnswerValue incl. GroupId)
    H->>DB: SELECT Question WHERE Id IN (submitted question ids)
    H->>H: Validate AnswerType matches Question.AnswerType
    H->>DB: SELECT existing Answer WHERE ProfileIdRef = X AND IsActive
    H->>H: match by (QuestionIdRef, FundIdRef, GroupId)
    alt match found, value differs
        H->>DB: UPDATE Answer (AnswerValue, DateModified)
    else match found, value same
        H->>H: skip — no DB write
    else no match
        H->>DB: INSERT new Answer row
    end
```

This exact match-by-`GroupId` logic is what the sync bugs in `04-Sync-And-Known-Defects.md` trace back to.

## 8. Known defect: cross-user KYC profile cache exposure

```mermaid
sequenceDiagram
    actor UserA as User A
    actor UserB as User B
    participant Next as Next.js Server<br/>(fetch Data Cache)
    participant Kyc as S1.Module.Kyc<br/>(kyc/profiles/me)

    UserA->>Next: GET page (Bearer token A)
    Next->>Next: Cache miss for "GET /kyc/profiles/me"
    Next->>Kyc: fetch (Authorization: Bearer A)
    Kyc-->>Next: [Profile IDs belonging to A]
    Next->>Next: Store in Data Cache, keyed by URL only
    Next-->>UserA: [Profile IDs belonging to A]

    UserB->>Next: GET page (Bearer token B)
    Next->>Next: Cache HIT for "GET /kyc/profiles/me"<br/>(auth header not part of the key)
    Next-->>UserB: [Profile IDs belonging to A] ⚠️ wrong user's data
```

Code-confirmed, not yet reproduced live: Next.js's fetch Data Cache keys on URL only, and `/kyc/profiles/me` carries no identifying parameter - the auth header is the only thing distinguishing users, and it's exactly what the cache ignores.

## 9. Known defect: stale permission cache, self-resolving but confusing

```mermaid
sequenceDiagram
    actor User
    participant IDR
    participant Redis as Redis (LegacyPermissions cache, 900s TTL)
    participant Kyc as S1.Module.Kyc (kyc/profiles/me)

    User->>IDR: Create profile / establish links
    Note over IDR: Permission computation is momentarily<br/>unsettled right after the mutation
    User->>Kyc: GET /kyc/profiles/me
    Kyc->>Redis: Cache miss
    Kyc->>IDR: GetUserPermissionProfilesRequest
    IDR-->>Kyc: Over-inclusive list (transient state)
    Kyc->>Redis: SET, 900s TTL
    Kyc-->>User: Expected 4 + extra not-yet-settled profiles

    Note over Redis: Frozen for 15 minutes regardless of<br/>what IDR's actual state does next

    User->>Kyc: GET /kyc/profiles/me (minutes later, still within TTL)
    Kyc->>Redis: Cache HIT — same stale over-inclusive list
    Kyc-->>User: Still showing the extra profiles

    Note over IDR: IDR settles to the correct,<br/>final permission set

    User->>Kyc: GET /kyc/profiles/me (TTL has now expired)
    Kyc->>Redis: Cache miss — must refetch
    Kyc->>IDR: GetUserPermissionProfilesRequest
    IDR-->>Kyc: Correct, settled list (just the 4)
    Kyc->>Redis: SET, 900s TTL
    Kyc-->>User: Back to the expected 4
```

Confirmed live: this cache is correctly scoped per-user (not a cross-user leak like §8) - the risk here is staleness, caught at exactly the wrong moment right after a profile mutation, self-resolving once the 15-minute TTL expires.

---

Next: `04-Sync-And-Known-Defects.md` - how IDR and the Rebuild actually talk to each other, and where that's confirmed to break.
