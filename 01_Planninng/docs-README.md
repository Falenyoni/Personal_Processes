# TemplateAPI Architecture Documentation

Welcome to the TemplateAPI architecture documentation. This folder contains design patterns, decisions, and guidelines for maintaining the codebase.

## Contents

### 📋 [Module Boundary Patterns](./Module-Boundary-Patterns.md)
**What:** Approved patterns for inter-module communication  
**When to read:** Before implementing cross-module features  
**Key topics:**
- Contract-mediated communication principle
- Shared DTO locations & ownership
- Module service adapter pattern
- Violation prevention checklist
- Adding new cross-module features

### 📝 [Architecture Decision Records (ADR)](./adr/)
**What:** Formal decisions on architectural approaches  
**When to read:** To understand the "why" behind patterns  
**Current ADRs:**
- [ADR-001: Contract-Mediated Module Communication](./adr/ADR-001-Contract-Mediated-Module-Communication.md) — How modules communicate via shared contracts + adapters

---

## Quick Start for Developers

### Adding a Cross-Module Feature?

1. Read: **Module Boundary Patterns** (take 5 min)
2. Follow: "Adding a New Cross-Module Feature" section (step-by-step)
3. Reference: **ADR-001** for the architectural rationale
4. Code review: Use **Violation Prevention Checklist**

### Proposing a New Architecture Pattern?

1. Create a new ADR using the template in `adr/README.md`
2. Set status to `Proposed`
3. Circulate for team discussion
4. Update to `Accepted` after approval

---

## Module Boundary Map

```
RulesEngine ←→ Kyc
│
├─ RulesEngine → Kyc:  /internal/kyc/question-fields
│                      via IKycModuleService
│
└─ Kyc → RulesEngine:  /internal/rules/calculate-risk
                       via IRiskService

Fund ↔ TransferAgency
├─ (Following same pattern)
└─ Extend as needed

[Other modules follow same contract-mediated pattern]
```

---

## Glossary

- **Shared DTO:** Data transfer object in `S1.Shared.Models.Contracts.{Module}` used for inter-module API contracts
- **Module Service Adapter:** Interface (e.g., `IKycModuleService`) wrapping `IInternalServiceClient` calls to peer modules
- **Internal Endpoint:** Route tagged with `[InternalRoute<T>]`; only callable via `IInternalServiceClient`, not external clients
- **Internal Service Client:** Service for calling internal endpoints between modules; handles authentication & serialization
- **Contract-Mediated:** Communication through defined APIs (contracts) rather than direct domain object imports

---

## Resources

- **Code Review Checklist:** See ADR-001 "Compliance & Enforcement" section
- **Roslyn Analyzers:** Planned for ADR-001 Phase 3 (future)
- **Developer Onboarding:** See `docs/adr/` for pattern onboarding guide (future)

---

## Questions?

Refer to ADR-001 "Questions & Discussions" section, or raise a discussion in code review.
