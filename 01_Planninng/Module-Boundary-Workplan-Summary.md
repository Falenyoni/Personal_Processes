# Module Boundary Fix - Work Plan Summary

**Work Item**: 27735 (Spike Investigation) → Module Architecture Follow-up  
**Date**: 2026-08-11  
**Owner**: Bongani Nyoni  
**Status**: PHASE 1 COMPLETE ✅ - Ready for PHASE 2

---

## Executive Summary

The `S1.Module.RulesEngine` module has **4 critical boundary violations** with `S1.Module.Kyc`. The most severe issue is `GetQuestionFieldsHandler.cs`, which directly couples RulesEngine to Kyc's domain models, enums, and services.

**Estimated Fix Effort**: ~12 hours  
**Priority**: HIGH - Blocks proper module deployment and testing  
**Recommended Approach**: Extract shared contracts → Refactor handlers → Remove project reference

---

## Violations at a Glance

| File | Type | Severity | Issue | Fix Time |
|------|------|----------|-------|----------|
| GetQuestionFieldsHandler.cs | Domain + Service | 🔴 CRITICAL | Queries Kyc models directly | 2-3h |
| CountryRisk.cs | Domain Inheritance | 🟠 HIGH | Base class from Kyc | 1h |
| AppConfiguration.cs | Service Dependency | 🟠 HIGH | LookupService coupling | 1h |
| QuestionFieldDto.cs | DTO Usage | 🟡 MEDIUM | Uses OptionDto from Kyc | 1h |
| .csproj | Project Reference | 🔴 CRITICAL | Hard Kyc dependency | 0.5h |

**Total**: 6+ hours implementation + 3 hours testing/validation = ~12 hours

---

## Detailed Findings

### 🔴 CRITICAL: GetQuestionFieldsHandler.cs

**Current Code**:
```csharp
using S1.Module.Kyc.Domain;
using S1.Module.Kyc.Domain.Enums;
using S1.Module.Kyc.Services;

public class GetQuestionFieldsHandler(
    IReadOnlyRepository<Question> questionRepository,      // Kyc entity
    LookupService lookupService                            // Kyc service
) : IRequestHandler<...>
{
    // Uses Question.QuestionType (Kyc enum)
    // Uses Question.AnswerType (Kyc enum)
    // Calls lookupService.GetLookupOptions() (Kyc method)
}
```

**Issues**:
- ❌ Can't work without Kyc's DbContext
- ❌ Can't test independently
- ❌ Tight coupling to Kyc domain structure
- ❌ Possible design smell: Should this be in Kyc module?

**Fix Strategy**:
1. Extract `IQuestion` interface to `S1.Shared.Models`
2. Extract `QuestionType`, `AnswerType` enums to shared
3. Create `ILookupProvider` interface in shared
4. Make handler accept interfaces instead of concrete types

---

### 🟠 HIGH: CountryRisk.cs

**Current Code**:
```csharp
using S1.Module.Kyc.Domain;  // Inherits from Kyc's AuditEntity

public class CountryRisk : AuditEntity, IEntity  // ← AuditEntity from Kyc?
{
    public string CountryCode { get; private set; }
    public RiskLevel RiskLevel { get; private set; }
}
```

**Issue**:
- ❌ Inherits from Kyc's base class
- Needs investigation: Is `AuditEntity` in Kyc or Shared?

**Fix**:
- Move/verify `AuditEntity` is in shared
- Create `ICountryRisk` interface in shared
- Remove Kyc import

---

### 🟠 HIGH: AppConfiguration.cs

**Current Code**:
```csharp
using S1.Module.Kyc.Services;

public static IServiceCollection AddRulesEngineModule(...)
{
    services.AddScoped(typeof(LookupService));  // Hard dependency!
    return services;
}
```

**Issue**:
- ❌ Registers Kyc's `LookupService` in RulesEngine's DI container
- ❌ Creates circular DI setup

**Fix**:
- Create `ILookupProvider` interface in shared
- Register abstraction instead of concrete type
- Move registration to composition root if needed

---

### 🟡 MEDIUM: QuestionFieldDto.cs

**Current Code**:
```csharp
using S1.Module.Kyc.Dto;

public class QuestionFieldDto
{
    public required HashSet<OptionDto> ValueOptions { get; set; } = [];  // ← From Kyc
}
```

**Issue**:
- ❌ Depends on Kyc's `OptionDto`

**Fix**:
- Move `OptionDto` to `S1.Shared.Models`
- Rename to more neutral name (e.g., `LookupOptionDto`)

---

## Solution Architecture

### Before (Current - Coupled)
```
RulesEngine
├── imports Kyc.Domain.Question
├── imports Kyc.Domain.QuestionType
├── imports Kyc.Services.LookupService
├── imports Kyc.Dto.OptionDto
└── depends on Kyc.csproj
```

### After (Desired - Decoupled)
```
Shared.Models
├── Contracts/RulesEngine/Questions/
│   ├── IQuestion.cs
│   ├── QuestionType.cs (enum)
│   ├── AnswerType.cs (enum)
│   └── LookupType.cs
├── Contracts/RulesEngine/Lookups/
│   ├── ILookupProvider.cs
│   └── LookupOptionDto.cs
└── Contracts/RulesEngine/Risk/
    └── ICountryRisk.cs

RulesEngine → imports only from Shared.Models
Kyc → imports only from Shared.Models
```

---

## Implementation Roadmap

### Phase 2: Create Shared Contracts (3-4 hours)

**Files to Create**:
1. `S1.Shared.Models/Contracts/RulesEngine/Questions/IQuestion.cs`
2. `S1.Shared.Models/Contracts/RulesEngine/Questions/QuestionType.cs`
3. `S1.Shared.Models/Contracts/RulesEngine/Questions/AnswerType.cs`
4. `S1.Shared.Models/Contracts/RulesEngine/Lookups/ILookupProvider.cs`
5. `S1.Shared.Models/Contracts/RulesEngine/Lookups/LookupOptionDto.cs`
6. `S1.Shared.Models/Contracts/RulesEngine/Risk/ICountryRisk.cs`

### Phase 3: Refactor RulesEngine (4-5 hours)

**Critical**: GetQuestionFieldsHandler.cs
- Accept `IReadOnlyRepository<IQuestion>` instead of `Question`
- Accept `ILookupProvider` instead of `LookupService`
- Use shared enums

**Other Files**:
- CountryRisk.cs - Update inheritance
- QuestionFieldDto.cs - Use shared OptionDto
- AppConfiguration.cs - Register interfaces

### Phase 4: Update Kyc (1-2 hours)

- Implement shared interfaces
- Add Shared.Models reference to .csproj

### Phase 5: Remove Project Reference (0.5 hours)

- Delete `<ProjectReference>` to Kyc from RulesEngine.csproj
- Build and verify

### Phase 6-7: Testing & Documentation (3-4 hours)

- Run full test suite
- Create architecture documentation
- Add boundary verification tests

---

## Critical Decision Point

### Should GetQuestionFieldsHandler be in Kyc module instead?

**Evidence it should be in Kyc**:
- ✅ Queries `Kyc.Question` repository
- ✅ Uses `QuestionType`, `AnswerType` enums (Kyc domain)
- ✅ Calls `LookupService` (Kyc service)
- ✅ Returns question field configuration for Kyc questionnaires

**Evidence it should be in RulesEngine**:
- ❓ Used by RulesEngine features?
- ❓ Supports rule condition building?

**Recommendation**: 
> Review with architecture/product team:
> - Does RulesEngine use GetQuestionFields?
> - Or is this a Kyc feature that was misplaced?
> - Decision affects refactoring approach

---

## Success Criteria

- [ ] Zero `using S1.Module.Kyc.*` in RulesEngine
- [ ] RulesEngine.csproj has NO Kyc project reference
- [ ] GetQuestionFieldsHandler works with `IQuestion` interface
- [ ] `dotnet build` succeeds with no errors/warnings
- [ ] `dotnet test` passes all tests
- [ ] Boundary verification tests created and passing
- [ ] Architecture documentation updated
- [ ] Code review approved

---

## Risk Assessment

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|-----------|
| Unit test failures | HIGH | MEDIUM | Run tests incrementally; mock interfaces early |
| GetQuestionFieldsHandler placement | MEDIUM | HIGH | Clarify with team before refactoring |
| Missing dependencies | MEDIUM | MEDIUM | Build frequently; resolve immediately |
| Data/behavior changes | LOW | HIGH | Review handler logic carefully |

---

## Documents Created

✅ **Module-Boundary-Fix-Plan.md** (11KB)
- Comprehensive 7-phase implementation guide
- Detailed task breakdown with time estimates
- Rollback plan

✅ **Boundary-Violations-Inventory.md** (11KB)
- Detailed analysis of 4 violations
- Severity assessment
- Remediation roadmap
- Questions for stakeholders

✅ **Module-Boundary-Workplan-Summary.md** (THIS DOCUMENT) (7KB)
- Executive summary
- Violations at a glance
- Critical decision points
- Risk assessment

---

## Database Tasks

Created 7 tracked todos with dependencies:
```
1. identify-kyc-imports (PENDING)
   ├─→ 2. create-shared-contracts (PENDING)
   ├─→ 3. refactor-ruleeengine-imports (PENDING)
   │   ├─→ 4. update-dto-layer (PENDING)
   │   └─→ 5. update-project-references (PENDING)
   │       └─→ 6. run-build-and-tests (PENDING)
   │           └─→ 7. document-boundary-policy (PENDING)
```

---

## Next Steps

### Immediate (This Session)
1. ✅ Complete Phase 1 analysis
2. ⏭️ Get stakeholder feedback on findings
3. ⏭️ Clarify GetQuestionFieldsHandler location

### Short-term (Next Session)
1. Begin Phase 2: Create shared contracts
2. Start Phase 3: Refactor GetQuestionFieldsHandler
3. Run incremental builds to verify progress

### Medium-term (After Core Fix)
1. Update Kyc module
2. Remove Kyc project reference
3. Full test suite + documentation

---

## Related Work Items

- **WI-27735**: Spike investigation (current work)
- **WI-27695**: Risk - Use more specific and easy to understand descriptions (blocked by this work)

---

## Contact & Questions

For questions on:
- **Architecture decisions**: Review Decision Point section
- **Implementation details**: See Module-Boundary-Fix-Plan.md
- **Violation analysis**: See Boundary-Violations-Inventory.md

---

**Report Generated**: 2026-08-11 15:30 UTC+2  
**Analysis Status**: PHASE 1 COMPLETE ✅  
**Ready for**: Phase 2 implementation OR stakeholder review
