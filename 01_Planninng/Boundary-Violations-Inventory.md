# Boundary Violations Inventory - Phase 1 Analysis

**Date**: 2026-08-11  
**Scope**: S1.Module.RulesEngine module violations  
**Total Violations Found**: 4 files with 6 direct Kyc imports

---

## Summary

RulesEngine has **tight coupling** to Kyc module via:
- 4 C# files with `using S1.Module.Kyc.*` statements
- 1 .csproj project reference
- Direct domain model usage
- Direct service dependencies

---

## Violation Details

### Violation 1: AppConfiguration.cs

**File**: `TemplateAPI/src/S1.Module.RulesEngine/Configuration/AppConfiguration.cs`

**Violation Type**: Service Dependency

```csharp
using S1.Module.Kyc.Services;  // ← VIOLATION

public static IServiceCollection AddRulesEngineModule(...)
{
    services.AddScoped(typeof(LookupService));  // From Kyc module
    return services;
}
```

**Analysis**:
- ❌ Directly depends on `LookupService` from Kyc
- ❌ Registers Kyc service in RulesEngine's DI container
- ❌ Creates implicit coupling at startup

**Impact**: Medium - Dependency injection level

**Fix Strategy**:
- [ ] Create `ILookupService` interface in `S1.Shared.Models`
- [ ] Move `LookupService` registration to composition root
- [ ] Use abstraction instead of concrete type

---

### Violation 2: CountryRisk.cs

**File**: `TemplateAPI/src/S1.Module.RulesEngine/Domain/CountryRisk.cs`

**Violation Type**: Domain Model Direct Usage

```csharp
using Idr.Utils;
using S1.Shared.Common.Models.Enums;
using S1.Module.Kyc.Domain;  // ← VIOLATION

namespace S1.Module.RulesEngine.Domain;

public class CountryRisk : AuditEntity, IEntity
{
    public string CountryCode { get; private set; } = string.Empty;
    public RiskLevel RiskLevel { get; private set; }
}
```

**Analysis**:
- ❌ Imports `S1.Module.Kyc.Domain` namespace
- ⚠️ BUT: Only uses common enum `RiskLevel` (from `S1.Shared.Common`)
- ❌ Unnecessary import - `AuditEntity` is also likely shared
- ❌ Inheritance from Kyc's base class

**Questions**:
- Where does `AuditEntity` come from? (Kyc module?)
- Is `RiskLevel` from Shared.Common or Kyc?

**Impact**: High - Domain model inheritance

**Fix Strategy**:
- [ ] Move `AuditEntity` to shared if not already there
- [ ] Remove Kyc.Domain import
- [ ] Create `ICountryRisk` interface in Shared
- [ ] Verify `RiskLevel` is in `S1.Shared.Common`

---

### Violation 3: QuestionFieldDto.cs

**File**: `TemplateAPI/src/S1.Module.RulesEngine/Dto/QuestionFieldDto.cs`

**Violation Type**: DTO Cross-Module Usage

```csharp
using S1.Module.Kyc.Dto;  // ← VIOLATION
using S1.Shared.Common.Models.Enums;

namespace S1.Module.RulesEngine.Dto;

public class QuestionFieldDto
{
    public required int Id { get; set; }
    public required string Name { get; set; }
    public required HashSet<OperatorType> Operators { get; set; } = [];
    public required HashSet<OptionDto> ValueOptions { get; set; } = [];  // ← From Kyc
}
```

**Analysis**:
- ❌ Imports `S1.Module.Kyc.Dto` namespace
- ❌ Uses `OptionDto` from Kyc module
- ⚠️ DTO-to-DTO dependency (slightly better than domain, but still coupling)
- ❌ Prevents RulesEngine from being used without Kyc

**Impact**: Medium-High - DTO usage

**Fix Strategy**:
- [ ] Move `OptionDto` to `S1.Shared.Models` (rename to neutral name)
- [ ] Create `IOption` interface in Shared
- [ ] Update QuestionFieldDto to use shared types

---

### Violation 4: GetQuestionFieldsHandler.cs

**File**: `TemplateAPI/src/S1.Module.RulesEngine/Features/GetQuestionFields/GetQuestionFieldsHandler.cs`

**Violation Type**: Domain Model + Service + Enum Usage

```csharp
using FluentResults;
using Idr.Utils.Nova.Data;
using MediatR;
using Microsoft.EntityFrameworkCore;
using S1.Module.Kyc.Domain;              // ← VIOLATION 1
using S1.Module.Kyc.Domain.Enums;        // ← VIOLATION 2
using S1.Module.Kyc.Services;            // ← VIOLATION 3
using S1.Module.RulesEngine.Dto;
using S1.Shared.Common.Models.Enums;

public class GetQuestionFieldsHandler(
    IReadOnlyRepository<Question> questionRepository,  // ← From Kyc.Domain
    LookupService lookupService                        // ← From Kyc.Services
    ) : IRequestHandler<GetQuestionFieldsRequest, Result<GetQuestionFieldsResponse>>
{
    public async Task<Result<GetQuestionFieldsResponse>> Handle(...)
    {
        var questions = await questionRepository
            .Get(q => q.IsActive)
            .ToListAsync(cancellationToken);
        
        // Uses Question from Kyc.Domain directly
        var questionFieldDto = new QuestionFieldDto
        {
            Id = question.Id,
            Name = question.Text,
            // ...
        };
        
        if (question.LookupType != null)
        {
            questionFieldDto.ValueOptions = lookupService.GetLookupOptions(...);
        }
        
        var operators = GetOperators(question);  // Question from Kyc
    }

    private static HashSet<OperatorType> GetOperators(Question question)
    {
        return question.QuestionType switch  // ← QuestionType from Kyc
        {
            QuestionType.InputEmail or
            QuestionType.InputTextCurrency or
            // ... uses Kyc.Domain.Enums
        };
    }
}
```

**Analysis**:
- ❌ **MOST CRITICAL VIOLATION**
- ❌ Directly depends on Kyc domain entities: `Question`, `QuestionType`, `AnswerType`
- ❌ Receives injected `Question` repository (from Kyc's EF DbContext)
- ❌ Calls Kyc service methods
- ❌ Uses Kyc enums for business logic
- ❌ Tightly coupled to Kyc's data model

**Impact**: Critical - Core business logic coupling

**Issue**: RulesEngine is essentially a feature/handler within Kyc domain
- Can't be deployed separately
- Can't be tested without Kyc's DbContext
- Can't be reused in other contexts

**Fix Strategy**:
- [ ] Extract shared Question interface to `S1.Shared.Models`
- [ ] Create abstraction for LookupService → `ILookupProvider`
- [ ] Move operator resolution logic to Shared
- [ ] Use DTOs for inter-module communication instead of domain models
- [ ] Consider if this handler belongs in Kyc module instead

---

## Project Reference Analysis

**File**: `TemplateAPI/src/S1.Module.RulesEngine/S1.Module.RulesEngine.csproj`

### Current Dependencies
```xml
<ProjectReference Include="..\S1.Module.Core\S1.Module.Core.csproj" />
<ProjectReference Include="..\S1.Module.Kyc\S1.Module.Kyc.csproj" />    <!-- ❌ VIOLATION -->
<ProjectReference Include="..\S1.Shared.Common\S1.Shared.Common.csproj" />
<ProjectReference Include="..\S1.Shared.Models\S1.Shared.Models.csproj" />
```

### Impact
- ❌ RulesEngine cannot compile without Kyc module
- ❌ Circular dependency risk if Kyc ever depends on RulesEngine
- ❌ Violates dependency inversion principle

### Desired Dependencies
```xml
<ProjectReference Include="..\S1.Module.Core\S1.Module.Core.csproj" />
<ProjectReference Include="..\S1.Shared.Common\S1.Shared.Common.csproj" />
<ProjectReference Include="..\S1.Shared.Models\S1.Shared.Models.csproj" />
<!-- ❌ NO Kyc reference -->
```

---

## Severity Assessment

### 🔴 CRITICAL (Fix Immediately)

1. **GetQuestionFieldsHandler.cs** (Violation 4)
   - Direct domain entity usage
   - Multiple Kyc dependencies
   - Core business logic issue
   - Estimated effort: 2-3 hours refactoring

### 🟠 HIGH (Fix in Phase 2)

2. **CountryRisk.cs** (Violation 2)
   - Domain model inheritance from Kyc
   - Base class coupling
   - Estimated effort: 1 hour

3. **AppConfiguration.cs** (Violation 1)
   - Service dependency registration
   - DI container coupling
   - Estimated effort: 1 hour

### 🟡 MEDIUM (Fix in Phase 3)

4. **QuestionFieldDto.cs** (Violation 3)
   - DTO cross-module usage
   - Estimated effort: 1 hour

---

## Remediation Roadmap

### Step 1: Extract Shared Contracts (Priority: Critical)

**Create Files**:
```
S1.Shared.Models/Contracts/RulesEngine/
├── Questions/
│   ├── IQuestion.cs
│   ├── QuestionType.cs (enum)
│   ├── AnswerType.cs (enum)
│   └── LookupType.cs
├── Risk/
│   ├── ICountryRisk.cs
│   └── IRiskLevel.cs
└── Lookups/
    ├── IOption.cs
    └── ILookupProvider.cs
```

### Step 2: Refactor Handler (Priority: Critical)

1. Accept `IQuestion` interface instead of `Question` entity
2. Accept `ILookupProvider` instead of `LookupService`
3. Extract operator resolution to shared utility

### Step 3: Update DTOs (Priority: High)

1. Move `OptionDto` to Shared
2. Create shared DTO hierarchy

### Step 4: Remove Project Reference (Priority: High)

1. Delete Kyc from .csproj
2. Run build - verify compilation errors guide remaining work

---

## Risk Assessment

### Compilation Risk: **HIGH**
- Removing Kyc reference will break at least 4 files
- May expose other hidden dependencies

### Behavioral Risk: **MEDIUM**
- Handler logic must work with interfaces, not concrete types
- Operator resolution may have edge cases

### Testing Risk: **MEDIUM**
- Unit tests may depend on Kyc DbContext
- Need to mock `ILookupProvider`

---

## Dependencies for Implementation

```
PHASE 1: Analysis ✅ COMPLETE
  └─ Phase 1 Output: This document

PHASE 2: Create Shared Contracts
  └─ Blocked by: Analyzing what needs to be shared
  
PHASE 3: Refactor RulesEngine
  └─ Blocked by: Phase 2 completion
  
PHASE 4: Remove Project Reference
  └─ Blocked by: Phase 3 completion
```

---

## Recommended Implementation Order

1. **First**: GetQuestionFieldsHandler.cs (most critical)
   - Extract IQuestion, QuestionType, AnswerType to Shared
   - Create ILookupProvider interface
   - Refactor handler to use interfaces

2. **Second**: CountryRisk.cs
   - Extract base class handling
   - Create ICountryRisk interface

3. **Third**: QuestionFieldDto.cs
   - Move OptionDto to Shared
   - Update references

4. **Fourth**: AppConfiguration.cs
   - Update LookupService usage
   - May resolve itself after handler fix

5. **Fifth**: .csproj reference removal
   - Should be clean after above steps

---

## Questions for Clarification

1. **Is GetQuestionFieldsHandler in the right module?**
   - Currently in RulesEngine
   - Operates entirely on Kyc domain models
   - Suggestion: Move to Kyc module? Or make it shared handler?

2. **What is LookupService used for?**
   - Need to understand its contract
   - Should it be ILookupProvider or something else?

3. **Who consumes GetQuestionFields endpoint?**
   - RulesEngine UI? Kyc UI? Both?
   - May inform where this code should live

4. **Are there tests for these violations?**
   - Need to understand test coverage
   - Tests may need refactoring too

---

## Files to Examine Next

- [ ] `S1.Module.Kyc/Domain/Question.cs` - Define IQuestion
- [ ] `S1.Module.Kyc/Domain/Enums/QuestionType.cs` - Extract enum
- [ ] `S1.Module.Kyc/Services/LookupService.cs` - Create interface
- [ ] `S1.Module.Kyc/Dto/OptionDto.cs` - Move to Shared
- [ ] `S1.Shared.Models/Contracts/` - Verify existing structure

---

**Analysis Complete**: Ready for implementation planning

**Next Step**: Move to Phase 2 - Create Shared Contracts
