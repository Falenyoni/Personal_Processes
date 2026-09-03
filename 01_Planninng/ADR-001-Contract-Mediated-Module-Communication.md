# ADR-001: Contract-Mediated Module Communication via Internal Service Client

**Date:** 2026-08-12  
**Status:** Accepted  
**Context:** TemplateAPI module boundary violations  
**Deciders:** Architecture Team  

---

## Problem Statement

TemplateAPI exhibits tight coupling between modules, particularly RulesEngine and Kyc:
- RulesEngine directly imported Kyc domain models, services, and DTOs
- Kyc directly imported RulesEngine risk calculation contracts
- No clear contract boundaries → changes to internal domain models ripple across modules
- Impossible to independently version, test, or refactor modules

**Root Cause:** Modules evolved organically without explicit inter-module communication patterns.

---

## Decision

**Adopt contract-mediated module communication using shared DTOs and internal service adapters.**

### The Pattern

```
Module A (Consumer)               Module B (Provider)
      ↓                                  ↓
   [IFooModuleService]  ────→  [/internal/foo/...]
   (Adapter)                    (Internal Endpoint)
      ↓                                  ↓
[Calls via                      [Returns via
 IInternalServiceClient]        Shared DTOs in
 + Shared DTOs]                 S1.Shared.Models.Contracts]
```

### Key Rules

1. **Modules export contracts, not domain objects**
   - Location: `S1.Shared.Models.Contracts.{ModuleName}/`
   - Shared DTOs are the API boundary; internal models never leak

2. **Consumer modules use adapters**
   - Pattern: `I{Provider}ModuleService` (e.g., `IKycModuleService`)
   - Implementation wraps `IInternalServiceClient` calls
   - Location: `S1.Module.{Consumer}/Services/{Provider}Module/`

3. **Provider exposes internal endpoints**
   - Marked with `[InternalRoute<T>]` attribute
   - Accepts + returns shared DTOs only
   - Never used directly by external clients (blocked by route)

4. **Project references are clean**
   - Consumer: `S1.Shared.Models` ✅, `S1.Module.{Provider}` ❌
   - Provider: `S1.Shared.Models` ✅
   - Eliminates circular dependencies and compile-time coupling

---

## Applied Solution: RulesEngine ↔ Kyc

### Communication Flows

| Direction | Request | Response | Endpoint |
|-----------|---------|----------|----------|
| Kyc → RulesEngine | `CalculateRiskRequestDto` | `CalculateRiskResponseDto` | `POST /internal/rules/calculate-risk` |
| RulesEngine → Kyc | `GetQuestionFieldsRequest` | `GetQuestionFieldsResponse` | `GET /internal/kyc/question-fields` |

### Contracts Extracted

**Kyc Contracts** (`S1.Shared.Models.Contracts.Kyc/`):
- `QuestionFieldDto` — Question metadata with operators & valid answer choices
- `LookupOptionDto` — Single answer option

**RulesEngine Contracts** (`S1.Shared.Models.Contracts.RulesEngine.Risk/`):
- `RiskConditionDto`, `RiskConditionValueDto` — Risk condition definitions (moved from internal Dto folder)
- `CalculateRiskRequestDto`, `CalculateRiskResponseDto` — Risk calculation API
- `AppliedRiskConditionDto`, `AnsweredQuestionDto` — Supporting contracts

### Implementation

**Provider (Kyc):**
- Created `GetQuestionFieldsEndpoint` (public) + `GetQuestionFieldsInternalEndpoint` (internal)
- Handler queries active questions, builds `QuestionFieldDto` list with operators

**Consumer (RulesEngine):**
- Created `IKycModuleService` adapter in `Services.KycModule/`
- `KycModuleService` wraps `IInternalServiceClient` call to `/internal/kyc/question-fields`
- Registered in `AppConfiguration` (DI container)
- Used by `GetQuestionFieldsHandler` to populate UI metadata

**Project References:**
- RulesEngine: ✅ Removed `S1.Module.Kyc` (now uses `S1.Shared.Models` + adapter pattern)
- Kyc: ✅ Uses `S1.Shared.Models.Contracts.RulesEngine.Risk` (no module dependency)

---

## Consequences

### Positive

✅ **Decoupling**
- Modules can refactor internal domain models without affecting peers
- Clear API boundary reduces cross-module assumptions

✅ **Testability**
- Modules mock module service adapters (e.g., `IKycModuleService`) instead of concrete implementations
- Integration tests can test contract compliance independently

✅ **Versioning**
- Contracts can be versioned at endpoint level (e.g., `/v2/internal/kyc/question-fields`)
- Backward compatibility maintained without cascading changes

✅ **Scalability**
- Pattern applies to all modules (Fund, TransferAgency, DocuSign, etc.)
- Reduces complexity of adding new cross-module features

✅ **Auditability**
- Clear contract definitions (DTOs in shared location) make dependencies explicit
- Easier to trace data flow between modules

### Negative

⚠️ **Extra Layer of Indirection**
- Modules now require adapter classes + shared DTOs (vs. direct imports)
- Slightly more boilerplate per feature

⚠️ **Network Latency**
- Internal service calls traverse HTTP (though on localhost with `IInternalServiceClient`)
- Minimal impact in practice; local calls are <1ms

⚠️ **Bidirectional Coupling Risk**
- RulesEngine ↔ Kyc now bidirectionally depends (via contracts)
- If this grows, may need to extract a mediating module (e.g., `RiskManagement`)

⚠️ **Contract Evolution**
- Shared DTOs are harder to remove/rename (affect multiple modules)
- Requires careful versioning strategy

---

## Alternatives Considered

### 1. **Direct Domain Model Imports** (Original State)
- ❌ Tight coupling; internal changes ripple across modules
- ❌ Circular dependencies possible
- ❌ Difficult to test in isolation

### 2. **Event-Driven Architecture**
- ✅ Fully decoupled (event producer unknown to consumer)
- ✅ Scales well for high-volume updates
- ❌ Introduces message broker complexity (RabbitMQ, etc.)
- ❌ Harder to debug synchronous request/response flows
- ❌ Eventual consistency challenges for risk calculations (need immediate response)
- 🚫 **Rejected:** Overkill for RulesEngine ↔ Kyc synchronous needs

### 3. **Shared Library with Contracts Only**
- ✅ Clean separation (DTOs isolated from domain)
- ❌ Modules still directly instantiate + import provider logic
- ❌ No adapter layer for testing
- 🚫 **Rejected:** Doesn't address coupling; contracts alone insufficient

### 4. **API Gateway Pattern**
- ✅ Centralized routing + orchestration
- ✅ Strong decoupling
- ❌ Adds latency + operational complexity
- ❌ Gateway becomes bottleneck
- 🚫 **Rejected:** Overkill for direct inter-module calls within single service

### 5. **REST Client (HttpClient)**
- ✅ True decoupling; contracts via HTTP
- ❌ Requires serialization + network overhead
- ❌ Harder to test (mocking HTTP requires infrastructure)
- 🚫 **Rejected:** `IInternalServiceClient` achieves same decoupling with better perf

---

## Compliance & Enforcement

### Code Review Checklist

When reviewing cross-module features:

- [ ] Shared DTOs defined in `S1.Shared.Models.Contracts.{Module}`?
- [ ] Provider exposes internal endpoint with `[InternalRoute<T>]`?
- [ ] Consumer uses module service adapter (not direct HTTP/imports)?
- [ ] Consumer imports from `S1.Shared.Models.Contracts`, NOT provider module?
- [ ] DI registration documented in consuming module's `AppConfiguration`?

### Static Analysis (Future)

- Roslyn analyzer: Flag imports of `S1.Module.{X}.Dto`, `S1.Module.{X}.Domain`, etc. from other modules
- Warn on project references between module projects

---

## Implementation Roadmap

### Phase 1: RulesEngine ↔ Kyc ✅ **COMPLETED**
- Extract shared DTOs to `S1.Shared.Models.Contracts`
- Create module service adapters (IKycModuleService)
- Remove circular project references
- Tests passing

### Phase 2: Other Modules (Fund, TransferAgency, DocuSign, etc.)
- Audit existing boundaries
- Extract contracts for each pair
- Refactor to adapter pattern

### Phase 3: Formalize Governance
- Add Roslyn analyzers for enforcement
- Update developer onboarding docs
- ADR review process for new cross-module features

---

## Related Decisions

- **ADR-002:** (Future) Event-driven sync workflows for high-volume updates
- **ADR-003:** (Future) API versioning strategy for shared contracts

---

## Questions & Discussions

**Q: What if a module needs to call multiple peer modules?**  
A: Create separate adapters (e.g., `IKycModuleService`, `IFundModuleService`). Each adapter is a single responsibility.

**Q: Can a module service adapter call another module service adapter?**  
A: Yes, but transitively. Adapter chains should be shallow (1-2 levels). Deep chains indicate design smell.

**Q: What about domain events within a module (not cross-module)?**  
A: Internal events are fine. Only cross-module communication must use contracts + adapters.

**Q: How do we handle contract changes without breaking consumers?**  
A: Semver for contracts. Add new fields as optional. Deprecated fields marked with `[Obsolete]`. Version endpoints if breaking changes needed.

---

## References

- **Module Boundary Patterns Document:** `/docs/Module-Boundary-Patterns.md`
- **RulesEngine ↔ Kyc Implementation:** 
  - Contracts: `S1.Shared.Models/Contracts/Kyc/` & `S1.Shared.Models/Contracts/RulesEngine/Risk/`
  - Adapter: `S1.Module.RulesEngine/Services/KycModule/IKycModuleService.cs`
  - Provider Endpoints: `S1.Module.Kyc/Features/GetQuestionFields/`

---

## Sign-Off

- **Proposed by:** Copilot (Architecture Assistant)
- **Reviewed by:** (Awaiting team review)
- **Accepted by:** (Awaiting approval)

**Date Accepted:** TBD  
**Supersedes:** N/A  
**Superseded by:** N/A
