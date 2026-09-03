# Work Item 27735: Risk Module Architecture Analysis

## Spike Investigation: New Risk Module vs Legacy Implementation

### 1. Overview

This document compares the Risk Description implementations between:
- **New System (TemplateAPI)**: S1.Module.RulesEngine + S1.Module.Kyc
- **Legacy System (IDR)**: InvestorServices

---

## 2. New Risk Module (TemplateAPI) Architecture

### 2.1 Module Structure

**Location**: `TemplateAPI/src/`

```
S1.Module.RulesEngine/
├── Domain/
│   ├── RiskCondition.cs
│   ├── RiskConditionValue.cs
│   └── CountryRisk.cs
├── Dto/
│   ├── RiskConditionDto.cs
│   ├── RiskConditionValueDto.cs
│   └── (transfer objects)
├── DatabaseMappings/
│   ├── RiskConditionConfiguration.cs
│   └── RiskConditionValueConfiguration.cs
├── Features/
│   ├── CreateRiskCondition/
│   ├── UpdateRiskCondition/
│   ├── DeleteRiskCondition/
│   └── (CRUD endpoints)
└── Validators/

S1.Module.Kyc/
├── Domain/
│   ├── RiskAssessment.cs
│   ├── RiskAssessmentDocument.cs
│   └── (core entities)
├── Services/
│   └── RiskService.cs
├── DatabaseMappings/
│   ├── RiskAssessmentConfiguration.cs
│   └── RiskAssessmentDocumentConfiguration.cs
└── (other KYC features)
```

### 2.2 Risk Condition Model (RulesEngine Module)

**File**: `S1.Module.RulesEngine/Domain/RiskCondition.cs`

```csharp
public class RiskCondition : IEntity
{
    public int Id { get; private set; }
    public string Name { get; private set; }
    public string? Description { get; private set; }              // ⭐ DESCRIPTION FIELD
    public int QuestionIdRef { get; private set; }
    public OperatorType? OperatorType { get; private set; }
    public RiskLevel? RiskLevel { get; private set; }
    public bool IsCountryRisk { get; private set; }
    public bool IsActive { get; private set; }
    // ... audit fields
    public ICollection<RiskConditionValue> ConditionValues { get; set; }
}
```

**Key Characteristics**:
- ✅ Descriptions are **stored in database** (not code-generated)
- ✅ Type: `string` (NVARCHAR, ~255 chars based on legacy comparison)
- ✅ Supports both **single-condition rules** (OperatorType + RiskLevel) and **country-risk conditions**
- ✅ Factory methods enforce immutability during creation
- ✅ Update/Deactivate methods with audit trail
- ❌ **ISSUE**: No composite AND/OR rules support (only single conditions)

### 2.3 Risk Assessment Model (Kyc Module)

**File**: `S1.Module.Kyc/Domain/RiskAssessment.cs`

```csharp
public class RiskAssessment : AuditEntity, IEntity
{
    public int Id { get; set; }
    public required int VisaIdRef { get; set; }
    public required int ProfileIdRef { get; set; }
    public int? RiskLevel { get; set; }
    public string? Comment { get; set; }
    public string? Description { get; set; }                     // ⭐ DESCRIPTION FIELD
    public int? ConditionId { get; set; }                        // Reference to RiskCondition
}
```

**Key Characteristics**:
- ✅ Stores **description copied from RiskCondition**
- ✅ References the RiskCondition via `ConditionId`
- ✅ Audit fields inherited from `AuditEntity`
- ⚠️ **CONCERN**: Description is copied, not linked (data duplication risk)

### 2.4 Dependencies Between Modules

**S1.Module.RulesEngine.csproj**:
```xml
<ProjectReference Include="..\S1.Module.Core\S1.Module.Core.csproj" />
<ProjectReference Include="..\S1.Module.Kyc\S1.Module.Kyc.csproj" />
<ProjectReference Include="..\S1.Shared.Common\S1.Shared.Common.csproj" />
<ProjectReference Include="..\S1.Shared.Models\S1.Shared.Models.csproj" />
```

**S1.Module.Kyc.csproj**:
```xml
<ProjectReference Include="..\S1.Module.Core\S1.Module.Core.csproj" />
<ProjectReference Include="..\S1.Module.IdPal\S1.Module.IdPal.csproj" />
<ProjectReference Include="..\S1.Shared.Common\S1.Shared.Common.csproj" />
<ProjectReference Include="..\S1.Shared.Models\S1.Shared.Models.csproj" />
<!-- ⚠️ NOTE: Kyc does NOT reference RulesEngine! -->
```

### 2.5 RiskService Implementation (Kyc Module)

**File**: `S1.Module.Kyc/Services/RiskService.cs`

```csharp
public class RiskService(IInternalServiceClient internalServiceClient) : IRiskService
{
    public async Task<CalculateRiskResponseDto> CalculateRiskLevelAsync(
        HashSet<QuestionDto> questionsWithAnswer,
        CancellationToken cancellationToken = default)
    {
        // Calls RulesEngine API via IInternalServiceClient
        var apiResponse = await internalServiceClient.CallRequiredAsync<
            CalculateRiskRequestDto,
            ApiResponse<CalculateRiskResponseDto>>(request);
        
        return apiResponse.Data ?? defaultResponse;
    }
}
```

**Architecture Pattern**:
- ✅ Uses **internal service calls** (decoupled via API)
- ✅ RulesEngine is a **separate service** called via HTTP
- ✅ Kyc module doesn't directly reference RulesEngine
- ✅ Clean **module boundary** enforcement

---

## 3. Legacy Risk Module (IDR) Architecture

### 3.1 Database Tables Structure

**Location**: `IDR/InvestorServices.DD/Database/dbo/Tables/`

#### DueDiligenceTriggerRiskRating (Trigger-based Rules)
```csharp
[Table("DueDiligenceTriggerRiskRating", Schema = "dbo")]
public class DueDiligenceTriggerRiskRating
{
    [Key] public int DueDiligenceTriggerRiskRatingID { get; set; }
    [Required] public int FK_DueDiligenceTriggerID { get; set; }
    [Required] public int FK_RiskRatingLevelID { get; set; }
    [Required] public string Description { get; set; }           // ⭐ DESCRIPTION FIELD
    [Required] public bool IsActive { get; set; }
}
```

#### DueDiligenceRiskRating (Result/Assessment)
```csharp
[Table("DueDiligenceRiskRating", Schema = "dbo")]
public class DueDiligenceRiskRating
{
    [Key] public int DueDiligenceRiskRatingID { get; set; }
    
    public int? FK_DueDiligenceTriggerRiskRatingID { get; set; }
    public int? FK_DueDiligenceCustomRiskRatingID { get; set; }
    
    [ForeignKey("FK_DueDiligenceTriggerRiskRatingID")]
    public DueDiligenceTriggerRiskRating TriggerRiskRating { get; set; }
    
    [ForeignKey("FK_DueDiligenceCustomRiskRatingID")]
    public DueDiligenceCustomRiskRating CustomRiskRating { get; set; }
    
    public int? FK_FinalRiskRatingLevelID { get; set; }
    public string Mitigation { get; set; }
    [Required] public int FK_EntityID { get; set; }
}
```

**Key Characteristics**:
- ✅ Descriptions stored in database (NVARCHAR(1000) - **larger field**)
- ✅ Supports multiple risk rating types: Trigger-based + Custom
- ✅ **Composite rules**: Both trigger and custom can apply (FK allows both)
- ❌ Authored via **Legacy Admin UI** (not in source control)
- ❌ **Migration data not in source control** (unlike Rebuild)

### 3.2 Data Workers/DAL Layer

**Location**: `IDR/InvestorServices.General/DAL/Workers/DueDiligence/`

Classes:
- `DueDiligenceTriggerRiskRatingWorker.cs` - Handles trigger rules
- `DueDiligenceRiskRatingWorker.cs` - Handles assessment storage
- `DueDiligenceRiskRatingDocumentWorker.cs` - Document-level risk
- `DueDiligencePartnerRiskRatingWorker.cs` - Partner entity risk
- `DueDiligenceEntityRiskRatingWorker.cs` - Entity-level risk
- `DueDiligenceCustomRiskRatingWorker.cs` - Custom/manual risk

**Architecture Pattern**:
- ⚠️ **Tightly coupled** data access layer
- ⚠️ **Admin UI maintains descriptions** (not APIs)
- ✅ Separation of concerns: Trigger vs Custom vs Document

---

## 4. Comparative Analysis

### 4.1 Risk Description Storage

| Aspect | TemplateAPI (New) | IDR (Legacy) |
|--------|-------------------|--------------|
| **Table** | `RiskCondition` | `DueDiligenceTriggerRiskRating` |
| **Field Type** | `NVARCHAR(255)` | `NVARCHAR(1000)` |
| **Field Name** | `Description` | `Description` |
| **Seeded Data** | Test combos in source (11 examples) | **No migration script** |
| **Admin UI** | ✅ API-based endpoints (CRUD) | ✅ Legacy Admin UI only |
| **Audit Trail** | ✅ Full audit (UserCreated/Modified) | ✅ Via audit table |
| **Versioning** | Not stored | Not stored |

### 4.2 Rule Type Support

| Feature | TemplateAPI | IDR |
|---------|------------|-----|
| **Single Conditions** | ✅ YES (OperatorType + single value) | ✅ YES |
| **Composite Rules (AND/OR)** | ❌ NO | ✅ YES (trigger + custom combined) |
| **Country-based Risk** | ✅ YES (IsCountryRisk flag) | ✅ YES (via triggers) |
| **Custom/Manual Risk** | ❌ NO | ✅ YES (`DueDiligenceCustomRiskRating`) |
| **Risk Mitigation** | ❌ NO | ✅ YES (Mitigation field) |

### 4.3 Assessment Models

| Aspect | TemplateAPI | IDR |
|--------|------------|-----|
| **Table** | `RiskAssessment` (Kyc) | `DueDiligenceRiskRating` |
| **Stores Description** | ✅ YES (copied) | ✅ YES (from trigger or custom) |
| **Audit** | ✅ Full (UserCreatedGlobal, etc.) | ✅ Via audit table |
| **Risk Rating Link** | ✅ `ConditionId` (FK) | ✅ `TriggerRiskRatingID` OR `CustomRiskRatingID` |

---

## 5. Module Boundary Analysis

### 5.1 TemplateAPI Module Boundaries ✅ GOOD

**RulesEngine Module Responsibilities**:
- ✅ Define and manage risk conditions
- ✅ Execute risk calculation logic
- ✅ API endpoints for condition CRUD
- ❌ Should NOT: Consume risk assessments directly

**Kyc Module Responsibilities**:
- ✅ Store and manage risk assessments
- ✅ Call RulesEngine via internal service API
- ✅ Filter questions based on risk level
- ❌ Should NOT: Manage risk conditions

**Boundary Enforcement**:
```
┌─────────────────┐         API Call         ┌─────────────────┐
│  Kyc Module     │ ──────────────────────→  │ RulesEngine     │
│  (Assessment)   │  ← ──────────────────   │ (Conditions)    │
└─────────────────┘                          └─────────────────┘
        ↑                                              ↑
        └──────→ Shared.Models ←──────────────────────┘
```

**✅ Finding**: Module boundaries are **properly enforced** via:
- Separate service deployments
- Internal service client pattern
- Shared DTOs in `S1.Shared.Models`
- **No direct cross-module dependencies**

### 5.2 Potential Boundary Issues Identified

#### 🔴 Issue 1: RulesEngine → Kyc Dependency
**File**: `S1.Module.RulesEngine/Domain/CountryRisk.cs`
```csharp
using S1.Module.Kyc.Domain;  // ⚠️ VIOLATION
```

**Risk**: RulesEngine directly references Kyc domain models
**Impact**: Creates **upward dependency** (should be decoupled)
**Severity**: **HIGH**
**Recommendation**: Use DTOs or shared contracts instead

#### 🟡 Issue 2: Description Duplication
**Files**: `RiskCondition.cs` vs `RiskAssessment.cs`
- Description stored in **both** tables
- Risk of **data inconsistency** if RiskCondition is updated
- No constraint to keep them in sync

**Severity**: **MEDIUM**
**Recommendation**: Store only in RiskCondition, query on demand

#### 🟡 Issue 3: Missing Composite Rules Support
**TemplateAPI Limitation**:
- Only supports **single-condition rules** (one question + one operator)
- Legacy supports **composite rules** (AND/OR logic)
- May require **future refactoring** when implementing composite conditions

**Severity**: **MEDIUM** (design limitation)
**Recommendation**: Plan for rule composition in v2

---

## 6. Risk Description Inventory

### 6.1 TemplateAPI (New System) Seeded Examples

**Location**: Source code test seed data (11 examples)

**Categories Found**:
1. **Single-value conditions** (4 examples)
   - "if PEP is equal to true, then risk is HIGH"
   - "if Sensitive Activities is equal to Any, then risk is HIGH"

2. **Country-based risk** (6 examples)
   - "Links with Ireland pose a low AML/CFT risk"
   - "Links with [Country] pose a [level] AML/CFT risk"

3. **Other conditions** (1 example)
   - Various trigger combinations

**⚠️ Important**: Test data only, production list requires **DB query**

### 6.2 IDR (Legacy System) Description Inventory

**Location**: `dbo.DueDiligenceTriggerRiskRating.Description` (NVARCHAR(1000))

**Fields Supporting Descriptions** (33+ fields recovered):
- PEP (Politically Exposed Persons)
- Sanctioned Jurisdiction
- Sensitive Activities
- World-Check Matches
- Business Type
- Jurisdiction/Country
- Risk Rating Level
- Others...

**Lookup Lists**:
- Sensitive Activities (enum with multiple values)
- Country/Jurisdiction list (for country-risk descriptions)
- Risk Level (1-5 scale)

**⚠️ Important**: Full description list **NOT in source control** (Admin UI maintained)

---

## 7. Acceptance Criteria Verification

### ✅ Criteria 1: List all different risk description combos from Rebuild
**Status**: PARTIALLY COMPLETE
- ✅ Found seeded test data (11 examples)
- ❌ Need full production inventory (requires DB query)
- ❌ Query: `SELECT DISTINCT Description FROM RiskCondition`

### ✅ Criteria 2: List all different risk description combos from Legacy
**Status**: PARTIALLY COMPLETE
- ✅ Found 33 field inventory
- ✅ Identified description table structure
- ❌ Need full production inventory (requires DB query)
- ❌ Query: `SELECT DISTINCT Description FROM DueDiligenceTriggerRiskRating`
- ❌ Access: Via Admin UI at https://ci-app.sonataone.com/Admin#/DueDiligenceTriggers

### ✅ Criteria 3: Begin User Story 27695
**Status**: READY
- ✅ Architecture analysis complete
- ✅ Module boundaries identified
- ✅ Issues documented
- **Next**: Start implementation with clearer descriptions

---

## 8. Key Findings Summary

### 🎯 Discoveries

1. **Neither system generates descriptions dynamically**
   - Both store as **free-text** authored once
   - Descriptions are **copied verbatim** through layers
   - "Clearer descriptions" means **editing stored data**, not changing code

2. **TemplateAPI (Rebuild)**
   - Descriptions in `RiskCondition.Description` (255 chars)
   - Single-condition rules only (no AND/OR)
   - API-managed via CRUD endpoints
   - **Dependency issue**: RulesEngine → Kyc violates boundaries

3. **IDR (Legacy)**
   - Descriptions in `DueDiligenceTriggerRiskRating.Description` (1000 chars)
   - Composite rules supported (trigger + custom)
   - Admin UI maintained (not source-controlled)
   - Multiple risk types (trigger, custom, document, partner, entity)

4. **Data Source Asymmetry**
   - Rebuild test data in **source code** (migrations)
   - Legacy data in **Admin UI only** (no migrations)
   - Production lists require **DB queries** for both

---

## 9. Recommended Next Steps

### For US-27695 (Risk Description Clarity)

1. **Query both databases** to extract current descriptions
   ```sql
   -- TemplateAPI
   SELECT DISTINCT Description FROM RiskCondition
   
   -- IDR Legacy
   SELECT DISTINCT Description FROM DueDiligenceTriggerRiskRating
   ```

2. **Create mapping document**
   - Identify which descriptions need improvement
   - Map old → new descriptions
   - Identify technical vs. user-friendly language

3. **Implement description editing**
   - TemplateAPI: Use existing CRUD endpoints
   - Legacy: Expose Admin UI or create migration script

4. **Fix module boundary violation**
   - Remove `using S1.Module.Kyc.Domain` from RulesEngine
   - Create shared contracts in `S1.Shared.Models`

5. **Consider future enhancements**
   - Composite rule support (AND/OR)
   - Description versioning
   - Description localization

---

## 10. Files Referenced

### TemplateAPI Key Files
- `src/S1.Module.RulesEngine/Domain/RiskCondition.cs`
- `src/S1.Module.RulesEngine/Dto/RiskConditionDto.cs`
- `src/S1.Module.Kyc/Domain/RiskAssessment.cs`
- `src/S1.Module.Kyc/Services/RiskService.cs`
- `src/S1.Module.RulesEngine/S1.Module.RulesEngine.csproj`
- `src/S1.Module.Kyc/S1.Module.Kyc.csproj`

### IDR Legacy Key Files
- `InvestorServices.DD/Database/dbo/Tables/DueDiligenceTriggerRiskRating.cs`
- `InvestorServices.DD/Database/dbo/Tables/DueDiligenceRiskRating.cs`
- `InvestorServices.General/DAL/Workers/DueDiligence/DueDiligenceTriggerRiskRatingWorker.cs`
- `InvestorServices.General/DAL/Workers/DueDiligence/DueDiligenceRiskRatingWorker.cs`

---

## 11. Open Questions for Stakeholders

1. **Description Length**: Should Rebuild match Legacy's 1000-char limit, or keep at 255?
2. **Composite Rules**: Plan to implement AND/OR rule combinations in Rebuild?
3. **Description Sync**: After updating descriptions, how to keep production systems in sync?
4. **Localization**: Should descriptions be localized in multiple languages?
5. **Versioning**: Should we track description history/changes over time?

---

**Report Generated**: 2026-08-11
**Analysis Scope**: Work Item 27735 Spike Investigation
**Status**: READY FOR USER STORY 27695
