# KYC Access Permissions & New KYC Scope

> Question: Can internal analysts view any profiles in KYC? Is "New KYC" building only an investor journey?

---

## 1. KYC Profile Access Permissions (IDR Monolith)

### Authorization Chain: `AuthorizeOrReadOnlyKYCOrManagedKyc()`

When an **analyst** accesses any KYC profile/endpoint, IDR checks permissions in this order:

```csharp
public PermissionType AuthorizeOrReadOnlyKYCOrManagedKyc(int userId, Type permissionType, int permissionSubjectId)
{
    // Level 1: Full permission on this specific entity
    if (IsAuthorized(userId, permissionType, permissionSubjectId))
        return PermissionType.Full;  // ✅ CAN VIEW & EDIT

    // Level 2: Read-only KYC permission (generic across all entities assigned to analyst)
    if (Workers.User.HasReadOnlyKYCPermissionFunction(userId, permissionSubjectId))
        return PermissionType.ReadOnly;  // ✅ CAN VIEW ONLY (NO EDIT)

    // Level 3: Managed KYC permission (for MKYC analysts managing on behalf of entities)
    if (Workers.User.HasManagedKYCPermissionFunction(userId, permissionSubjectId))
        return PermissionType.ReadOnly;  // ✅ CAN VIEW ONLY (NO EDIT)

    // If none match: deny access
    ThrowError(userId, $"{permissionType.Name} or Read Only KYC", permissionSubjectId, "Access Denied");
    return PermissionType.None;  // ❌ CANNOT VIEW
}
```

### Answer: Can Analysts View Any Profiles?

**NO** — Not "any" profiles. An analyst can view only profiles where they have:

1. **Full Permission** (direct assignment) — can view AND edit
2. **Read-Only KYC Permission** (admin-granted generic access) — can view ONLY
3. **Managed KYC Permission** (MKYC role) — can view ONLY (but for entities they manage on behalf of)

**Profile visibility is controlled by explicit permission assignment, not role membership.**

### Permission Subject ID

The permission is always scoped to a specific `entityId` (the profile/entity being accessed). An analyst doesn't get a blanket "view all KYC" permission; instead:

```
Analyst User → Permission on Entity X → Can access Entity X KYC
Analyst User → Permission on Entity Y → Can access Entity Y KYC
Analyst User → NO Permission on Entity Z → CANNOT access Entity Z KYC
```

---

## 2. KYC Types in IDR (3 Service Types)

### IKYC — Investor Self-Service Journey

```
Service.Registration = 20
Description: "IKYC"
Standard: InvestmentKycStandard (value 4)
State Machine: 4 states
Ownership: Investor fills out KYC themselves
Who can access?
├── Investor (self-service via investor portal)
├── Analyst (read-only if granted "Read-Only KYC" permission)
└── Fund administrator
```

**Path:** `/entities/{entityId}/dueDiligence` → `GetCddProfile()`

### MKYC — Managed KYC (Analyst-Driven)

```
Service.MANAGEDKYC = 18
Description: "MANAGEDKYC"
State Machine: 6 states (includes WITH_COUNTERPARTY, QUESTION_PENDING_WITH_IDR)
Ownership: Analyst fills out KYC on behalf of investor/counterparty
Who can access?
├── Analyst managing this entity (via MKYC permission)
├── Fund administrator
└── Investor (view-only access to their profile)
```

**Special Fields:**
- `CounterPartyRequestsRelatedProfile` — linked to counterparty profile
- `ManagedKycCertificate` — certificate of completion
- `MkycBulkChaserEmail` — chaser emails to complete KYC

**Dashboard:** `ManagedKycDashboard` with separate state + action tracking

### CDD/KYC — Standard Due Diligence

```
Service.CDD = 8
Description: "CDD"
Ownership: Fund's compliance officer or analyst
When used: Additional due diligence beyond IKYC (for high-risk investors)
Who can access?
├── Analyst with CDD permission
├── Fund administrator
└── Compliance officer
```

---

## 3. The Three Controllers That Unify This

All three KYC types funnel through the **same DueDiligenceController** with the **same authorization method**:

```csharp
[Route("entities/{entityId:int}/dueDiligence")]
public class DueDiligenceController : ApiController
{
    [HttpGet]
    [Route("")]
    public IHttpActionResult GetCddProfile(int entityId)
    {
        var permissionType = AuthService.AuthorizeOrReadOnlyKYCOrManagedKyc(
            currentUser, 
            typeof(Permissions.Entity.Due_Diligence.Cdd_Profile_View),  // permission type
            entityId  // which profile/entity
        );

        // If permissionType == ReadOnly: return view-only data
        // If permissionType == Full: return editable data
        // If thrown exception: 403 Access Denied
    }
}
```

**Key insight:** Whether an analyst is accessing IKYC, MKYC, or CDD, the **permission check is identical** — they must have been explicitly granted access to that specific entity.

---

## 4. "New KYC" — TemplateAPI (SonataOne)

### What Is It?

**TemplateAPI** is a **new platform** (separate from IDR monolith) being built for KYC/compliance workflows. It's NOT just an "investor journey" — it's a complete rearchitecture of KYC.

### Architecture

```
TemplateAPI (SonataOne)
├── S1.Module.Kyc
│   ├── Domain/              KYC business rules
│   ├── Features/            KYC endpoints + handlers
│   ├── Services/            KYC business logic
│   ├── Security/            KYC-specific permissions
│   └── Sync/                sync with IDR (data mirror)
│
├── S1.Module.AnalystAction
│   ├── Domain/              Analyst-driven workflows
│   ├── Features/            Analyst task endpoints
│   └── (for MKYC/analyst-initiated KYC)
│
├── S1.Kyc.Templates         KYC questionnaire templates
└── S1.Module.RulesEngine    Rules that trigger KYC, classification checks
```

### Scope: What Each Module Actually Does

**New KYC is NOT investor-only.** It covers:

| Module | Actual Purpose | What It Does |
|---|---|---|
| **`S1.Module.Kyc`** | Investor Self-Service | Investor fills out KYC form (IKYC equivalent) |
| **`S1.Module.AnalystAction`** | Task/Action Management | Tracks analyst tasks & actions (Get/List/Track actions) — **NOT** MKYC itself, but manages analyst workflow tasks |
| **`S1.Module.RulesEngine`** | Rules-Based Triggering | Auto-trigger KYC/actions based on investment characteristics & compliance rules |
| **`S1.Kyc.Templates`** | Email Templates Only | **NOT questionnaire templates** — renders email templates for ProfileCreated, NewCertification notifications |

### Key Difference from IDR

| Aspect | IDR (Monolith) | TemplateAPI (New KYC) |
|---|---|---|
| **Analyst Permission Check** | `AuthorizeOrReadOnlyKYCOrManagedKyc()` hardcoded per endpoint | Permission model TBD (likely similar but service-based) |
| **Investor Access** | Separate investor portal | Integrated into TemplateAPI (same platform) |
| **Analyst Actions** | IKYC/MKYC/CDD mixed in one controller | Split: `Module.Kyc` (investor), `Module.AnalystAction` (analyst) |
| **Sync Strategy** | N/A (monolith) | **Data sync from TemplateAPI → IDR** via `Sync/` folder (dual-write pattern) |

---

## 5. Access Model: New KYC (TemplateAPI)

### Likely Similar Permission Model

Since TemplateAPI is using similar architecture (modular, contract-mediated), the permission model is likely:

```
Investor Journey (S1.Module.Kyc)
  ├── Investor can only view/edit their own profile
  │   └── Authorization: currentUserId == investorId
  └── Analyst can view if granted read-only or full permission
      └── Authorization: similar to IDR's AuthorizeOrReadOnlyKYCOrManagedKyc

Analyst Actions (S1.Module.AnalystAction)
  ├── Tracks tasks assigned to analysts (AssignedTo field)
  ├── Task types from ActionType (code/name)
  ├── Associated with Investor + Fund
  ├── Service Level Agreement tracking
  ├── Completion tracking (IsCompleted, CompletedBy)
  └── Access: Analyst can view actions assigned to them, Mgr can view team actions

Note: This is TASK MANAGEMENT, not MKYC execution.
      MKYC (managed KYC) logic is likely separate or within S1.Module.Kyc
```

### Email Templates (S1.Kyc.Templates)

**NOT questionnaire templates** — only email notification templates:
- `ProfileCreatedEmail.cshtml` — when a new profile is created
- `NewCertificationEmail.cshtml` — when certification is issued
- `Layout.cshtml` — email layout template

Used by `TemplateRenderer` to send notifications to investors/analysts.

### RulesEngine Integration

`S1.Module.RulesEngine` determines **when** KYC is triggered:

```csharp
// Pseudo-code from RulesEngine
if (investmentAmount > HighRiskThreshold)
{
    await _kycModuleService.TriggerKyc(investor, "HighRiskInvestor");
}

if (investorCountry in SanctionsList)
{
    await _kycModuleService.RequestAnalystKyc(investor, "SanctionsReview");
}
```

The **RulesEngine doesn't check permissions** — it triggers KYC creation. The **Kyc module** then enforces who can view/edit.

---

## 6. Summary: Answer to Your Questions

### Q1: Can internal analysts view any profiles in KYC?

**A:** No. Analysts can view only profiles where they have been **explicitly granted**:
- Full permission (edit + view), OR
- Read-only KYC permission (view-only), OR
- Managed KYC permission (MKYC role, view-only)

**Control:** Admin assigns per-analyst, per-entity.

**Code:** `AuthorizeOrReadOnlyKYCOrManagedKyc()` in IDR; similar model expected in TemplateAPI.

---

### Q2: Is "New KYC" (TemplateAPI) building only an investor journey?

**A:** Partially wrong in my original claim. New KYC actually covers:

1. **Investor Self-Service** (`S1.Module.Kyc`) — investor fills out KYC
2. **Analyst Task Management** (`S1.Module.AnalystAction`) — tracks analyst tasks & actions (NOT MKYC execution; MKYC is likely within Kyc module)
3. **Rules-Based Triggers** (`S1.Module.RulesEngine`) — auto-triggers KYC/tasks based on compliance rules
4. **Email Notifications** (`S1.Kyc.Templates`) — sends ProfileCreated & Certification emails

**Correction:** `AnalystAction` is NOT MKYC. It's a **task management system** tracking what analysts need to do (DueDate, AssignedTo, IsCompleted, ServiceLevelAgreement). MKYC (managed KYC) logic appears to be within `S1.Module.Kyc` module itself.

**Correction:** `Kyc.Templates` is NOT questionnaire templates. It's **email notification templates** (ProfileCreated, NewCertification). Questionnaire templates likely live in Kyc.Contracts or Kyc.Domain.

---

## 7. Migration Path: IDR → TemplateAPI

### Current State (IDR Monolith)

```
DueDiligenceController
├── IKYC (investor self-service) ✗ permission check (readonly + auth)
├── MKYC (analyst-driven)        ✗ permission check (readonly + auth)
└── CDD (additional DD)           ✗ permission check (readonly + auth)
```

### Target State (TemplateAPI + IDR Strangler)

```
IDR (legacy)                          TemplateAPI (new)
├── IKYC (read-only mode)       ←→   S1.Module.Kyc (investor journey)
├── MKYC (read-only mode)       ←→   S1.Module.AnalystAction (analyst journey)
└── CDD (read-only mode)        ←→   S1.Module.RulesEngine (triggers)

Data sync: TemplateAPI → IDR (for backward compatibility, reporting)
```

---

*Document created to clarify KYC access permissions and New KYC scope.*
