# KYC Module Architecture Recommendation
## For Team Breakdown & Development (Dec Deadline)

> Audience: 2 dev teams, ~4-5 months to build, starting from scratch or refactoring existing IDR KYC

---

## Executive Summary

**Recommend: 2 core modules + 1 shared library, split between 2 teams**

```
Team A: S1.Module.Kyc (investor + analyst workflows)
Team B: S1.Module.RulesEngine (triggers, compliance checks)

Shared (used by both):
  → S1.Shared.Kyc.Contracts
  → S1.Shared.Kyc.Domain
```

**No separate MKYC module.** MKYC is a workflow *within* Kyc module, not a separate entity.

---

## 1. Module Breakdown (What Gets Built)

### Module 1: `S1.Module.Kyc` (Team A Owns)

**Scope:** Everything KYC investors & analysts interact with

#### 1.1 Features (Features Folder)

```
Features/
├── Investor/
│   ├── CreateProfile/          investor self-service create/update profile
│   ├── GetProfile/
│   ├── SubmitKyc/              investor submits completed KYC
│   └── ViewKycStatus/          investor views their KYC status
│
├── Analyst/
│   ├── CreateManagedKyc/       analyst creates KYC on behalf of investor
│   ├── EditManagedKyc/         analyst updates investor's KYC
│   ├── RequestInvestorInfo/    analyst requests missing investor data
│   ├── ReviewProfile/          analyst reviews & approves
│   ├── BulkImportKyc/          bulk import for onboarding
│   └── AssignAnalyst/          assign analyst to a profile
│
├── Classification/
│   ├── GetClassification/      fetch US/CRS classification (tax domain)
│   ├── UpdateClassification/   update classification
│   └── GetClassificationStatus/ check if approved
│
├── Questionnaire/
│   ├── GetQuestionnaire/       fetch active questionnaire for fund
│   ├── SaveAnswers/            save investor answers
│   └── ValidateAnswers/        validate completeness
│
└── Shared/
    ├── GetProfileHistory/      audit trail of KYC changes
    └── ExportKyc/              export profile to PDF/Excel
```

#### 1.2 Domain (Business Logic)

```
Domain/
├── Entities/
│   ├── KycProfile              main entity (investor name, type, addresses, etc)
│   ├── KycAnswer               answers to questionnaire questions
│   ├── KycClassification       US + CRS classification per profile
│   ├── KycStatus               (Draft, InReview, Approved, Rejected, Expired)
│   ├── KycTimeline             SLA tracking (created date, due date, approved date)
│   └── KycAudit                change history (who changed what, when)
│
├── ValueObjects/
│   ├── KycStatusEnum           Draft, InProgress, PendingApproval, Approved, Rejected, Expired
│   ├── KycTypeEnum             IKYC (investor), MKYC (analyst-managed), CDD (extra due diligence)
│   ├── ReviewReason            enum: HighRisk, PEP, Sanctions, Insufficient, Other
│   └── ApprovalLevel           enum: Analyst, Manager, Compliance
│
├── Rules/
│   ├── KycCompletionRule       all required fields filled?
│   ├── KycExpiryRule           KYC older than 3 years? trigger renewal
│   ├── KycRiskRule             investor country/type → high risk? → escalate
│   └── ClassificationRule      require FATCA/CRS classification before approval?
│
└── Exceptions/
    ├── KycValidationException  incomplete/invalid KYC
    ├── KycStatusException      invalid status transition
    └── ClassificationRequiredException  classification missing
```

#### 1.3 Security (Permissions Module-Level)

```
Security/
├── KycPermissions
│   ├── View_Own_Profile        investor can view own only
│   ├── View_Assigned_Profiles  analyst can view assigned only
│   ├── Edit_Own_Profile        investor can edit own, draft only
│   ├── Edit_Assigned_Profiles  analyst can edit assigned
│   ├── Approve_Profiles        manager/compliance can approve
│   └── Admin_All_Profiles      admin can see all
│
└── KycAuthHandler            policy-based auth (user + profile → permission check)
```

#### 1.4 Data Access (Repositories)

```
Repositories/
├── IKycProfileRepository       CRUD for KycProfile
├── IKycAnswerRepository        get/save answers
├── IKycClassificationRepository get/save classification
└── IKycAuditRepository         readonly for history
```

#### 1.5 Services (Business Orchestration)

```
Services/
├── KycProfileService
│   ├── CreateProfile()
│   ├── UpdateProfile()
│   ├── ApproveProfile()
│   └── RejectProfile(reason)
│
├── KycQuestionnaireService
│   ├── GetActiveQuestionnaire(fundId)
│   ├── ValidateAnswers(answers)
│   └── SaveAnswers(profileId, answers)
│
├── KycClassificationService
│   ├── FetchClassification(profileId)       [calls Tax domain via contract]
│   ├── RequireClassification(profileId)     [validation rule]
│   └── UpdateClassification(profileId)      [calls Tax domain via contract]
│
└── KycTimelineService
    ├── GetProfileTimeline(profileId)
    ├── CalculateDueDate(profileId)
    └── CheckExpiry(profileId)
```

---

### Module 2: `S1.Module.RulesEngine` (Team B Owns)

**Scope:** Rules that trigger KYC workflows, classify risk, auto-escalate

#### 2.1 Features

```
Features/
├── EvaluateInvestment/         when new investment → run rules
├── EvaluateProfile/            when profile data changes → re-evaluate
├── GetApplicableRules/         get rules for fund/investor type
├── CreateRule/                 admin creates rule
├── TestRule/                   dry-run rule on test data
└── GetRuleResults/             see what rule triggered & why
```

#### 2.2 Domain (Rule Engine Logic)

```
Domain/
├── Entities/
│   ├── Rule                    condition → action definition
│   ├── RuleCondition           "if investor country = SanctionsList"
│   ├── RuleAction              "then: TriggerKyc, EscalateToCompliance, RequestDocuments"
│   ├── RuleExecution           log of rule run (when, inputs, outputs)
│   └── RuleVersion             versioning for rule changes
│
├── RuleConditionTypes/
│   ├── InvestmentAmount > $X
│   ├── InvestorCountry in [list]
│   ├── EntityType = PEP/Trust/Corp
│   ├── InvestorAge < 18 or > 65
│   ├── HighNetWorth criteria
│   ├── Previous sanctions hit
│   ├── Investor has no tax forms
│   └── Custom: [admin-defined]
│
├── RuleActions/
│   ├── TriggerKyc(kycType: IKYC | MKYC | CDD)          calls KYC module
│   ├── TriggerClassification(type: US | CRS)           calls Tax module
│   ├── EscalateToCompliance(reason, priority)          creates task
│   ├── RequestDocuments(docType[], deadline)           KYC module creates task
│   ├── SendAlert(recipient, message)                   notify admin/analyst
│   ├── CreateAnalystAction(actionType, sla)           AnalystAction module
│   └── ApplyRiskFlag(level: Low | Med | High | Critical)
│
├── RuleChains/
│   ├── CompoundRule             AND/OR multiple conditions
│   └── SequentialRule           condition1 → action1 → condition2 → action2
│
└── Services/
    ├── RuleEvaluationService
    │   ├── Evaluate(investment) → List<RuleExecution>
    │   └── EvaluateProfile(profileId) → List<RuleExecution>
    │
    ├── RuleExecutionService
    │   ├── ExecuteActions(List<Rule>)    [calls KYC, Tax, AnalystAction modules]
    │   └── LogExecution(execution)
    │
    └── RuleManagementService
        ├── CreateRule()
        ├── UpdateRule()
        ├── DeactivateRule()
        └── GetRuleHistory(ruleId)
```

#### 2.3 Integration Points (Contracts from Shared)

RulesEngine **calls other modules** via contracts:

```
Services/
├── IKycModuleService (contract from S1.Shared.Kyc.Contracts)
│   └── TriggerKyc(investorId, fundId, kycType, reason)
│
├── ITaxModuleService (contract from S1.Shared.Tax.Contracts)
│   └── TriggerClassification(entityId, type)
│
├── IAnalystActionService (contract from S1.Shared.AnalystAction.Contracts)
│   └── CreateAction(investorId, fundId, actionType, dueDate)
│
└── (IInternalServiceClient adapter wrapping above)
```

---

### Shared Layer: `S1.Shared.Kyc.Contracts` & `S1.Shared.Kyc.Domain` (Both Teams Use)

**Team A & B both depend on this. Build first.**

#### Shared Contracts (DTOs for cross-module calls)

```
S1.Shared.Kyc.Contracts/

├── CreateKycRequest
│   ├── InvestorId, FundId
│   ├── KycType (IKYC | MKYC | CDD)
│   ├── TriggerReason
│   └── DueDate
│
├── KycStatusDto
│   ├── ProfileId, Status
│   ├── LastUpdated, UpdatedBy
│   └── ApprovalLevel
│
├── KycProfileDto
│   ├── InvestorName, InvestorType, InvestorCountry
│   ├── Address, TaxResidences, TIN
│   ├── KycType, Status
│   └── ClassificationStatus (US/CRS: Approved | Pending | Rejected)
│
├── IKycModuleService (internal API contract)
│   ├── CreateKyc(CreateKycRequest)
│   ├── GetProfile(profileId)
│   ├── UpdateStatus(profileId, newStatus)
│   ├── RequireClassification(profileId, type)
│   └── CompleteKyc(profileId)
│
└── KycEvents (async messaging)
    ├── KycCreatedEvent
    ├── KycSubmittedEvent
    ├── KycApprovedEvent
    ├── KycRejectedEvent
    ├── KycExpiringEvent
    └── ClassificationRequiredEvent
```

#### Shared Domain Models

```
S1.Shared.Kyc.Domain/

├── Enums/
│   ├── KycStatus               Draft, InProgress, PendingApproval, Approved, Rejected, Expired
│   ├── KycType                 IKYC, MKYC, CDD
│   ├── KycEntityType           Individual, Trust, Corporation, Partnership, Pension, Government
│   ├── ApprovalLevel           Analyst, Manager, Compliance, Admin
│   └── RiskLevel               Low, Medium, High, Critical
│
└── ValueObjects/
    ├── KycProfileId
    ├── QuestionnaireVersion
    ├── ClassificationStatus
    └── ApprovalHistory (who approved when)
```

---

## 2. Recommended Team Split

### Team A: KYC Module (Investor + Analyst Workflows)
- Dev Lead: Owns Kyc module architecture
- Responsibilities:
  - Investor self-service features (create/submit/view)
  - Analyst managed KYC (create/edit/approve on behalf)
  - Questionnaire & answer handling
  - Profile history & audit
  - SLA tracking & reminders

**Deliverables (by Dec):**
- ✅ Kyc module with Features, Domain, Services
- ✅ Database schema (KycProfile, KycAnswer, KycClassification, KycAudit)
- ✅ REST APIs (GET/POST/PUT profile, questionnaire, status)
- ✅ Kyc-to-Tax integration (via contract for classification)
- ✅ Kyc-to-RulesEngine integration (handle triggered KYC)
- ✅ Unit tests + integration tests
- ✅ Basic UI/API docs

### Team B: RulesEngine Module (Compliance Triggers)
- Dev Lead: Owns RulesEngine architecture
- Responsibilities:
  - Rule definition & management
  - Rule evaluation on investment/profile changes
  - Action execution (call KYC/Tax/AnalystAction)
  - Audit logging of rule executions
  - SLA escalation logic

**Deliverables (by Dec):**
- ✅ RulesEngine module with Features, Domain, Services
- ✅ Database schema (Rule, RuleCondition, RuleAction, RuleExecution)
- ✅ REST APIs (GET/POST rules, test rule, get results)
- ✅ Internal service client calls to Kyc/Tax modules
- ✅ Unit tests + integration tests
- ✅ Pre-built common rules (PEP, Sanctions, HighRisk, High Value)
- ✅ Rule testing/dry-run capability

---

## 3. Shared Infrastructure (Coordination Between Teams)

**Assign 1 person (part-time) to own:**

1. **Shared Contracts** (`S1.Shared.Kyc.Contracts`)
   - Define DTOs, interfaces, events
   - Both teams contribute & agree on APIs

2. **Database Design** (cross-module)
   - KycProfile table schema
   - Integration with existing IDR tables (Entity, Investor, Fund)

3. **Internal Service Client** (`IInternalServiceClient` adapter)
   - How RulesEngine calls Kyc module
   - How Kyc calls Tax/AnalystAction modules
   - Error handling, retry logic

4. **Event Bus / Messaging** (async communication)
   - KycCreatedEvent → RulesEngine re-evaluates?
   - ClassificationRequiredEvent → sent from Kyc to Tax
   - Both teams understand the flow

---

## 4. Development Timeline (Dec Deadline = 4-5 Months)

### Month 1 (Sep): Design & Setup
- **Week 1-2:**
  - Design shared contracts & domain models
  - Database schema design (Entity Relationship Diagram)
  - Define APIs (OpenAPI spec for Kyc & RulesEngine)
  - Create Project structure

- **Week 3-4:**
  - Implement shared contracts & domain
  - Database migrations
  - Both teams ready to start feature development

**Deliverable:** Shared library published to internal NuGet, DB deployed

### Month 2 (Oct): Core Features
**Team A:**
- Investor profile create/update/view
- Basic questionnaire answer handling
- Status tracking (Draft → InProgress → Approved)

**Team B:**
- Rule CRUD operations
- Basic rule evaluation (single condition)
- Rule execution logging

**Integration:** Kyc module callable from external (Team B can see it via contract)

### Month 3 (Nov): Advanced Features & Integration
**Team A:**
- Analyst managed KYC (create on behalf)
- Approval workflow (Analyst → Manager → Compliance)
- Classification requirements from Tax domain
- Bulk import from IDR legacy

**Team B:**
- Complex rules (AND/OR conditions)
- Multi-action workflows
- Escalation to AnalystAction
- Pre-built rule library (20-30 common rules)

**Integration:** RulesEngine successfully calls Kyc module APIs, Tax module APIs

### Month 4 (Dec): Polish, Testing, Migration
- **Both Teams:**
  - Unit test coverage (>80%)
  - Integration tests with other modules
  - Performance testing & optimization
  - Security review (permissions, data access)
  - Data migration from IDR (if applicable)

- **Team A:**
  - Questionnaire templates finalized
  - Kyc expiry job scheduled (3-year renewal)
  - Bulk approval workflows

- **Team B:**
  - Rule conflict detection (overlapping rules)
  - Audit reports on rule execution
  - Admin dashboard for rule management

**Release:** MVP to staging environment

---

## 5. What's Shared vs. Separate

### SHARED (Both Teams Use)

✅ **`S1.Shared.Kyc.Contracts`**
- IKycModuleService interface
- CreateKycRequest, KycStatusDto, KycProfileDto
- KycStatus, KycType enums
- Events (KycCreatedEvent, etc.)

✅ **`S1.Shared.Kyc.Domain`**
- Enums (KycStatus, KycType, RiskLevel, ApprovalLevel)
- Value objects (KycProfileId, ClassificationStatus)
- Exceptions (KycValidationException, etc.)

✅ **Database Tables** (shared schema)
- `Kyc.Profile` — core investor profile
- `Kyc.Answer` — questionnaire answers
- `Kyc.Classification` — US/CRS classification status
- `Kyc.Audit` — change history
- `RulesEngine.Rule` — rule definitions
- `RulesEngine.Execution` — rule run logs

✅ **IInternalServiceClient** (shared infrastructure)
- How modules call each other
- Error handling, logging

### SEPARATE (Team A Only)

✅ **`S1.Module.Kyc`**
- Features (investor/analyst/questionnaire endpoints)
- Domain (KycProfile, KycAnswer entities)
- Services (KycProfileService, KycQuestionnaireService)
- Repositories
- Security (KycPermissions)

### SEPARATE (Team B Only)

✅ **`S1.Module.RulesEngine`**
- Features (rule CRUD, evaluation endpoints)
- Domain (Rule, RuleCondition, RuleAction entities)
- Services (RuleEvaluationService, RuleExecutionService)
- Repositories
- Pre-built rule library

---

## 6. External Dependencies (Don't Build, Integrate)

| Dependency | Module | Interface | Notes |
|---|---|---|---|
| **Tax Classification** | S1.Module.Tax (exists) | ITaxModuleService | Call to get US/CRS classification status |
| **Investor/Fund Data** | IDR monolith (legacy) | REST API or database view | Sync investor metadata |
| **Analyst Actions** | S1.Module.AnalystAction | IAnalystActionService | RulesEngine creates tasks here |
| **Questionnaires** | TBD (Kyc.Templates? or Kyc.Domain?) | Need to clarify | Fetch active questionnaire per fund |
| **Document Management** | S1.Module.DocumentManagement | IDocumentService | Upload/store KYC supporting docs |

---

## 7. Risk Mitigation

| Risk | Mitigation |
|---|---|
| **Scope creep** | Strictly scope "v1" to IKYC + MKYC only. CDD is "future" |
| **Teams blocking each other** | Finalize shared contracts by end of Month 1 |
| **Database schema conflicts** | Assign DBA/architect to own schema design early |
| **Tax integration unclear** | Schedule kickoff with Tax module team week 1 |
| **Questionnaire source unknown** | Clarify: is it in Kyc.Domain or separate Kyc.Templates module? |
| **No legacy data migration plan** | Identify "v1" can skip legacy migration (greenfield) OR plan Oct-Nov for it |
| **Performance bottlenecks** | Load test rule evaluation in Oct, optimize in Nov |

---

## 8. Success Criteria (December)

- ✅ KYC module: create/update/approve investor profiles
- ✅ RulesEngine: evaluate rules & trigger KYC/tasks
- ✅ Both teams: unit test coverage >80%
- ✅ Integration: Kyc ↔ RulesEngine ↔ Tax working end-to-end
- ✅ Staging: can onboard 50 test investors
- ✅ Performance: <200ms response time for profile operations
- ✅ Documentation: API docs, architecture diagram, run-book
- ✅ Security: permissions tested, data access validated

---

## 9. Kickoff Checklist

**Before Development Starts:**

- [ ] Team assignments finalized (Team A lead, Team B lead, shared infrastructure person)
- [ ] Shared contracts & domain library scaffolded
- [ ] Database schema reviewed & approved
- [ ] OpenAPI specs written & agreed
- [ ] Development environment setup (local, CI/CD pipeline)
- [ ] Source control branching strategy defined
- [ ] Code review process agreed
- [ ] Definition of "done" (unit tests, integration tests, docs)
- [ ] Communication cadence (daily standup? weekly sync with other modules?)
- [ ] Escalation path (if Team A needs from Team B, who decides priority?)

---

*Document intended for team leads & project managers. Ready to build.*
