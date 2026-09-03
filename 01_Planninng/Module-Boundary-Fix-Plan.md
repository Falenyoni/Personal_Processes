# Module Boundary Fix Plan

**Objective**: Fix the module boundary violation where `S1.Module.RulesEngine` directly imports `S1.Module.Kyc.Domain`

**Target**: Enforce clean architectural boundaries using shared contracts pattern

---

## Problem Statement

### Current Violation
```
S1.Module.RulesEngine
    ↓ (VIOLATION)
uses S1.Module.Kyc.Domain
    ↓
RulesEngine directly depends on Kyc's internal domain models
```

**Affected Files**:
- `TemplateAPI/src/S1.Module.RulesEngine/Domain/CountryRisk.cs` 
  - Contains: `using S1.Module.Kyc.Domain;`

**Impact**:
- ❌ Circular dependency risk
- ❌ Tight coupling between modules
- ❌ RulesEngine cannot be deployed/tested independently
- ❌ Changes in Kyc domain break RulesEngine

---

## Target Architecture

### Desired State
```
┌─────────────────────────────────────────────┐
│  S1.Shared.Models (Shared Contracts)        │
│  ├── RulesEngine/Risk/*.cs                  │
│  └── Kyc/*.cs                               │
└─────────────┬─────────────────────────────┘
              ↑
    ┌─────────┴──────────┐
    ↓                    ↓
S1.Module.RulesEngine  S1.Module.Kyc
  (Conditions)          (Assessments)
    ↓                    ↓
    └─────────┬──────────┘
              ↓
        API/Internal Service Client
        (Clean Boundary)
```

### Communication Pattern
```
RulesEngine → Shared.Models (imports OK)
Kyc → Shared.Models (imports OK)
RulesEngine → Kyc (API ONLY via IInternalServiceClient)
RulesEngine ✗ Kyc.Domain (NEVER direct import)
```

---

## Implementation Plan

### Phase 1: Analysis & Inventory (2-3 hours)

#### 1.1 Identify All Boundary Violations
**Task**: Find all `using S1.Module.Kyc` in RulesEngine

```bash
# Search for all Kyc imports in RulesEngine
grep -r "using S1.Module.Kyc" TemplateAPI/src/S1.Module.RulesEngine/
grep -r "S1.Module.Kyc" TemplateAPI/src/S1.Module.RulesEngine/*.csproj
```

**Expected Files**:
- [ ] `S1.Module.RulesEngine/Domain/CountryRisk.cs` (confirmed)
- [ ] (Others to identify)

#### 1.2 Map Domain Model Dependencies
**Task**: For each violation, identify:
- What domain models from Kyc are used?
- Why are they needed in RulesEngine?
- Can they be extracted to Shared.Models?

**Example**:
```csharp
// File: CountryRisk.cs
using S1.Module.Kyc.Domain;  // ← VIOLATION

public class CountryRisk : IEntity
{
    // Does this use Kyc domain models?
    // Needs investigation
}
```

**Output**: Create `Boundary-Violations-Inventory.md` listing all violations

---

### Phase 2: Create Shared Contracts (3-4 hours)

#### 2.1 Create Shared RulesEngine Contracts Folder

**Location**: `TemplateAPI/src/S1.Shared.Models/Contracts/RulesEngine/`

```
S1.Shared.Models/Contracts/RulesEngine/
├── Risk/
│   ├── IRiskEntity.cs
│   ├── ICountryRiskEntity.cs
│   └── (other shared interfaces)
├── Enums/
│   ├── OperatorType.cs (if not already shared)
│   └── RiskLevel.cs (if not already shared)
└── Models/
    ├── CountryRiskModel.cs
    └── (other shared DTOs)
```

**Principle**: Move ONLY what's needed for inter-module communication

#### 2.2 Extract Shared Interfaces

**Example - From Kyc.Domain to Shared.Models**:

**Current** (in Kyc module):
```csharp
// S1.Module.Kyc/Domain/RiskAssessment.cs
namespace S1.Module.Kyc.Domain;

public class RiskAssessment : AuditEntity, IEntity
{
    public int Id { get; set; }
    public string? Description { get; set; }
    // ...
}
```

**Shared** (in Shared.Models):
```csharp
// S1.Shared.Models/Contracts/RulesEngine/Risk/IRiskAssessment.cs
namespace S1.Shared.Models.Contracts.RulesEngine.Risk;

public interface IRiskAssessment : IEntity
{
    int Id { get; }
    string? Description { get; }
}
```

**Kyc Module** (updated):
```csharp
// S1.Module.Kyc/Domain/RiskAssessment.cs
namespace S1.Module.Kyc.Domain;

public class RiskAssessment : AuditEntity, IRiskAssessment
{
    // Implements shared interface
}
```

#### 2.3 Create Shared Enums

If not already in Shared.Models:
- `OperatorType` → `S1.Shared.Models/Contracts/RulesEngine/Enums/OperatorType.cs`
- `RiskLevel` → `S1.Shared.Models/Contracts/RulesEngine/Enums/RiskLevel.cs`

---

### Phase 3: Refactor RulesEngine Module (4-5 hours)

#### 3.1 Update CountryRisk.cs

**Before**:
```csharp
using S1.Module.Kyc.Domain;  // ← REMOVE THIS

namespace S1.Module.RulesEngine.Domain;

public class CountryRisk : IEntity
{
    // References Kyc models
}
```

**After**:
```csharp
using S1.Shared.Models.Contracts.RulesEngine.Risk;  // ← ADD THIS

namespace S1.Module.RulesEngine.Domain;

public class CountryRisk : IEntity
{
    // References shared interfaces instead
}
```

#### 3.2 Update All RulesEngine Files

**For each file importing Kyc**:
- [ ] Replace `using S1.Module.Kyc.Domain` with `using S1.Shared.Models.Contracts.*`
- [ ] Update all type references to use shared interfaces
- [ ] Remove any Kyc-specific logic
- [ ] Add unit test to verify no Kyc references

#### 3.3 Update Project References

**File**: `TemplateAPI/src/S1.Module.RulesEngine/S1.Module.RulesEngine.csproj`

**Before**:
```xml
<ProjectReference Include="..\S1.Module.Kyc\S1.Module.Kyc.csproj" />
```

**After**:
```xml
<!-- Remove the direct Kyc reference -->
<!-- Only keep Shared references -->
<ProjectReference Include="..\S1.Shared.Models\S1.Shared.Models.csproj" />
<ProjectReference Include="..\S1.Shared.Common\S1.Shared.Common.csproj" />
```

---

### Phase 4: Update Kyc Module (2-3 hours)

#### 4.1 Add Shared Contracts Reference

**File**: `TemplateAPI/src/S1.Module.Kyc/S1.Module.Kyc.csproj`

```xml
<ProjectReference Include="..\S1.Shared.Models\S1.Shared.Models.csproj" />
```

#### 4.2 Implement Shared Interfaces

Update Kyc domain models to implement shared interfaces:

```csharp
// S1.Module.Kyc/Domain/RiskAssessment.cs
public class RiskAssessment : AuditEntity, IRiskAssessment
{
    // Now implements the shared interface
}
```

---

### Phase 5: Update DTO Layer (2-3 hours)

#### 5.1 Verify Shared DTOs

**Files to check**:
- `S1.Shared.Models/Contracts/RulesEngine/Risk/RiskAssessmentDto.cs`
- `S1.Module.RulesEngine/Dto/RiskConditionDto.cs`
- `S1.Module.Kyc/Dto/RiskAssessmentDto.cs` (if exists)

**Goal**: Ensure DTOs are the **only** inter-module communication vehicle

#### 5.2 Create/Update Mapping Profiles

Ensure clean DTO mapping (e.g., AutoMapper):
```csharp
// Example mapping
CreateMap<RiskCondition, RiskConditionDto>();
CreateMap<RiskAssessment, RiskAssessmentDto>();
```

---

### Phase 6: Validation & Testing (3-4 hours)

#### 6.1 Build Verification

```bash
# Clean build
cd TemplateAPI
dotnet clean
dotnet build --no-restore

# Check for build errors
# Expected: 0 errors
```

#### 6.2 Dependency Analysis

```bash
# Verify no circular dependencies
# Tool: NDepend or dotnet-tools

# Check RulesEngine dependencies
dotnet add reference ... --validate-only
```

#### 6.3 Run Module Tests

```bash
# RulesEngine tests
dotnet test test/S1.Module.RulesEngine.Tests/ -v minimal

# Kyc tests
dotnet test test/S1.Module.Kyc.Tests/ -v minimal

# Integration tests
dotnet test test/TemplateApi.Integration.Tests/ -v minimal
```

#### 6.4 Create Boundary Verification Tests

Add tests to prevent regression:

```csharp
// Test: RulesEngine module should not reference Kyc directly
[Test]
public void RulesEngineModule_ShouldNotReferenceSModule_Kyc()
{
    var assembly = typeof(RiskCondition).Assembly;
    var kycReferences = assembly.GetReferencedAssemblies()
        .Where(a => a.Name.Contains("Kyc"))
        .ToList();
    
    Assert.That(kycReferences, Is.Empty, 
        "RulesEngine should not have direct Kyc references");
}
```

**Location**: `test/S1.Module.RulesEngine.Tests/Architecture/ModuleBoundaryTests.cs`

---

### Phase 7: Documentation (1-2 hours)

#### 7.1 Update Architecture Documentation

**File**: Create `docs/ARCHITECTURE.md` or `ARCHITECTURE_DECISIONS.md`

```markdown
# Module Boundary Policy

## Approved Communication Patterns

### ✅ ALLOWED
- Module A → Shared.Models (contracts, DTOs)
- Module A → Internal Service API
- Module A → Module B via shared contracts

### ❌ FORBIDDEN
- Module A → Module B domain models
- Module A → Module B internal classes
- Circular dependencies

## Example: RulesEngine ↔ Kyc
```

#### 7.2 Create Module Dependency Diagram

```
docs/module-dependencies.png
- Visual representation of allowed dependencies
- Highlight the fix applied
```

#### 7.3 Add Code Comments

Add comment to prevent future violations:

```csharp
// CountryRisk.cs
namespace S1.Module.RulesEngine.Domain;

/// <summary>
/// Represents country-based risk conditions.
/// 
/// NOTE: This module follows strict boundary policies.
/// Do not import S1.Module.Kyc.Domain directly.
/// Use S1.Shared.Models contracts for inter-module communication.
/// See: docs/ARCHITECTURE.md
/// </summary>
public class CountryRisk : IEntity
{
    // ...
}
```

---

## Detailed Tasks Breakdown

| Task | Owner | Effort | Dependencies |
|------|-------|--------|--------------|
| 1.1 - Search violations | @Bongani | 30min | - |
| 1.2 - Map dependencies | @Bongani | 1.5h | 1.1 |
| 2.1 - Create shared folders | @Bongani | 15min | 1.2 |
| 2.2 - Extract interfaces | @Bongani | 2h | 2.1 |
| 2.3 - Create shared enums | @Bongani | 45min | 2.1 |
| 3.1-3.3 - Refactor RulesEngine | @Bongani | 2h | 2.2, 2.3 |
| 4.1-4.2 - Update Kyc | @Bongani | 1h | 2.2, 2.3 |
| 5.1-5.2 - Update DTOs | @Bongani | 1h | 3.3, 4.2 |
| 6.1-6.4 - Testing | @Bongani | 2h | 5.2 |
| 7.1-7.3 - Documentation | @Bongani | 1h | 6.4 |
| **TOTAL** | | **~12 hours** | |

---

## Success Criteria

- [ ] ✅ Zero `using S1.Module.Kyc` statements in RulesEngine
- [ ] ✅ RulesEngine.csproj has no Kyc project reference
- [ ] ✅ All module tests pass (`dotnet test`)
- [ ] ✅ Integration tests pass
- [ ] ✅ No build warnings/errors
- [ ] ✅ Boundary verification tests created and passing
- [ ] ✅ Architecture documentation updated
- [ ] ✅ Code review approved

---

## Rollback Plan

If issues arise:

1. Git branch: Create `feature/fix-module-boundaries`
2. Revert strategy: `git revert` specific commits
3. Test suite: Run full test suite before reverting
4. Communication: Notify team of any blockers

---

## Next Steps After Fix

1. ✅ **Phase 1**: Fix module boundaries (THIS PLAN)
2. ⏭️ **Phase 2**: Fix description duplication (RiskCondition/RiskAssessment)
3. ⏭️ **Phase 3**: Plan composite rule support
4. ⏭️ **Phase 4**: Implement clearer descriptions (User Story 27695)

---

## References

- **Issue**: Work Item 27735 Spike Investigation
- **Analysis**: `WI-27735-RiskModule-Analysis.md`
- **Architecture Pattern**: Modular Monolith with Shared Contracts
- **Tools**: dotnet, git, (optional) NDepend for dependency analysis

---

**Created**: 2026-08-11
**Status**: READY FOR IMPLEMENTATION
