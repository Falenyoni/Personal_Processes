# Module Boundary Patterns

## Overview

This document defines the approved architectural patterns for inter-module communication in TemplateAPI, specifically addressing the RulesEngine ↔ Kyc module boundary.

---

## Core Principle: Contract-Mediated Communication

**Modules communicate through shared contracts only, never through direct domain object references.**

### Pattern: Internal Service Client with Shared DTOs

Modules maintain:
1. **Shared DTOs** in `S1.Shared.Models.Contracts.{ModuleName}` — Single source of truth for inter-module contracts
2. **Module Service Adapter** in consuming module (e.g., `IKycModuleService` in RulesEngine) — Encapsulates remote calls
3. **Public & Internal Endpoints** in providing module (public for external clients, internal for peer modules via `IInternalServiceClient`)

**Benefits:**
- Decouples internal domain models from contract APIs
- Enables future provider/consumer patterns without ripple changes
- Supports versioning at contract layer only
- Allows either module to refactor internals independently

---

## RulesEngine ↔ Kyc Boundary

### Contracts (Shared DTOs)

Location: `S1.Shared.Models/Contracts/Kyc/` and `S1.Shared.Models/Contracts/RulesEngine/Risk/`

**Kyc → RulesEngine:**
- `CalculateRiskRequestDto` — Questions answered by user; triggers risk calculation
- `CalculateRiskResponseDto` — Risk level + applied conditions; returned to Kyc
- `AppliedRiskConditionDto` — Summary of which conditions triggered the risk level
- `AnsweredQuestionDto` — Single Q&A pair

**RulesEngine → Kyc:**
- `GetQuestionFieldsRequest` — (empty request)
- `GetQuestionFieldsResponse` — Question metadata (operators, valid answer options)
- `QuestionFieldDto` — Question structure with operators & value options
- `LookupOptionDto` — Valid answer choices

**RulesEngine (Internal Use):**
- `RiskConditionDto` — Full risk condition definition (name, logic, condition values)
- `RiskConditionValueDto` — Valid trigger values for a condition

### Provider Side: Kyc Module

**Public Endpoints:**
- `GET /kyc/questions` — List all active questions (public API)
- `POST /kyc/questionnaires/submit` — Submit answers & get risk assessment (public API)

**Internal Endpoints (for RulesEngine via `IInternalServiceClient`):**
- `GET /internal/kyc/question-fields` — Returns `QuestionFieldDto[]` (question operators & valid values)
  - Attribute: `[InternalRoute<GetQuestionFieldsResponse>]`
  - Handler: `GetQuestionFieldsHandler` queries active questions, builds structure
  - Used by RulesEngine to populate UI dropdowns/validations

### Consumer Side: RulesEngine Module

**Public Endpoints:**
- `GET /rules/risk/conditions` — List all active risk conditions (public API for admins)
- `POST /rules/risk/conditions` — Create/update conditions (admin operations)

**Internal Module Service:**
- `IKycModuleService.GetQuestionFieldsAsync()` → calls `/internal/kyc/question-fields`
  - Implementation: `KycModuleService` wraps `IInternalServiceClient.CallRequiredAsync<...()`
  - Location: `S1.Module.RulesEngine/Services/KycModule/`
  - Used by `GetQuestionFieldsHandler` to populate question metadata

**Risk Calculation (Bidirectional):**
- Kyc calls `CalculateRiskAsync()` via `IRiskService` → wraps IInternalServiceClient to `/internal/rules/calculate-risk`
- RulesEngine calculates and returns `CalculateRiskResponseDto`

---

## DTOs: Ownership & Location

| DTO | Owner | Location | Usage |
|-----|-------|----------|-------|
| `RiskConditionDto` | RulesEngine | `S1.Shared.Models/Contracts/RulesEngine/Risk/` | GetRiskConditions endpoint response (can be consumed by any module) |
| `RiskConditionValueDto` | RulesEngine | `S1.Shared.Models/Contracts/RulesEngine/Risk/` | Part of RiskConditionDto structure |
| `CalculateRiskRequestDto` | RulesEngine | `S1.Shared.Models/Contracts/RulesEngine/Risk/` | Kyc → RulesEngine API contract |
| `CalculateRiskResponseDto` | RulesEngine | `S1.Shared.Models/Contracts/RulesEngine/Risk/` | RulesEngine → Kyc response |
| `AppliedRiskConditionDto` | RulesEngine | `S1.Shared.Models/Contracts/RulesEngine/Risk/` | Part of CalculateRiskResponseDto |
| `AnsweredQuestionDto` | RulesEngine | `S1.Shared.Models/Contracts/RulesEngine/Risk/` | Part of CalculateRiskRequestDto |
| `QuestionFieldDto` | Kyc | `S1.Shared.Models/Contracts/Kyc/` | Kyc → RulesEngine question metadata contract |
| `LookupOptionDto` | Kyc | `S1.Shared.Models/Contracts/Kyc/` | Part of QuestionFieldDto |

**Principle:** Shared contracts live in `S1.Shared.Models`, never in module-specific projects.

---

## Request/Response Payload Example

### Risk Calculation Flow

**Kyc → RulesEngine** (`POST /internal/rules/calculate-risk`)
```json
{
  "answeredQuestions": [
    {
      "questionId": 5,
      "answerType": "Boolean",
      "answerValue": { "boolValue": true }
    }
  ]
}
```

**RulesEngine → Kyc** (Response)
```json
{
  "riskLevel": "Medium",
  "appliedConditions": [
    {
      "conditionId": 12,
      "name": "High-Risk Country",
      "description": null,
      "questionId": 5,
      "riskLevel": "Medium"
    }
  ]
}
```

---

## Violation Prevention Checklist

✅ **Before exposing data to another module, ask:**

- [ ] Is this information needed across module boundaries?
- [ ] Have I defined a shared DTO in `S1.Shared.Models.Contracts.{Module}`?
- [ ] Does the consuming module reference the shared contract (not the provider's internal DTOs)?
- [ ] Is the provider exposing this via a public or internal endpoint?
- [ ] Does the consumer use `IInternalServiceClient` for internal calls (not direct HTTP)?
- [ ] Are both request and response wrapped in shared contracts?

✅ **Before writing a cross-module call:**

- [ ] Am I importing from `S1.Shared.Models.Contracts`, not the provider's `S1.Module.{X}.Dto`?
- [ ] Am I using a module service adapter (e.g., `IKycModuleService`)?
- [ ] Have I added the adapter to my module's DI container?
- [ ] Is the provider endpoint tagged with `[InternalRoute<T>]` or marked `[Authorize]`?

---

## Adding a New Cross-Module Feature

1. **Define the contract** → Create request/response DTOs in `S1.Shared.Models.Contracts.{ProviderModule}`
2. **Expose provider endpoint** → Add handler + endpoint (public or internal route) in provider module
3. **Create consumer adapter** → Add `I{Provider}ModuleService` + implementation in consumer module
4. **Wire DI** → Register adapter in consumer's AppConfiguration
5. **Update project references** → Consumer references `S1.Shared.Models`, **not** the provider's module project

---

## Current State: RulesEngine ↔ Kyc Coupling

**Communication Paths:**
- RulesEngine → Kyc: `IKycModuleService.GetQuestionFieldsAsync()` → `/internal/kyc/question-fields`
- Kyc → RulesEngine: `IRiskService.CalculateRiskLevelAsync()` → `/internal/rules/calculate-risk`

**Project References:**
- RulesEngine: ✅ Removed `S1.Module.Kyc` reference (now uses `S1.Shared.Models` + `IKycModuleService`)
- Kyc: ✅ Removed `S1.Module.RulesEngine` reference (now uses `S1.Shared.Models` + `IRiskService`)

**Assessment:** Bidirectional but contract-mediated—acceptable for now; monitor if scope grows.

---

## Future Considerations

**Reducing Coupling:**
- If RulesEngine ↔ Kyc coupling grows significantly, consider extracting a third `RiskManagement` module that both consume
- Alternatively, establish one-way dependency (e.g., only Kyc depends on RulesEngine for read operations)

**Extending to Other Modules:**
- Fund, TransferAgency, DocuSign, InvestorServices follow the same pattern
- All new cross-module interactions should use shared contracts + module service adapters

**Versioning:**
- When contracts change, maintain backward compatibility or bump version in endpoint route (e.g., `/v2/internal/kyc/question-fields`)
- Use feature flags to gate new contract versions during rollout
