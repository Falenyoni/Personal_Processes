# Work Item 27735: Spike - Risk Descriptions Analysis

## Overview

| Field | Value |
|-------|-------|
| **ID** | 27735 |
| **Type** | User Story |
| **State** | In Progress |
| **Assigned To** | Bongani Nyoni |
| **Created By** | Luke Butchart |
| **Story Points** | 3 |
| **Priority** | 2 |
| **Area** | IDR\Muiz |
| **Sprint** | Sprint 16 - 2026 |
| **Tags** | Backend; UAT Feedback |

## Title

**Spike: What are all the old and new Risk Descriptions?**

## Description

The current Risk Assessment descriptions are using "technical language" that is confusing to a user.

**Examples:**
- "if PEP is equal to true, then risk is HIGH"
- "if Sensitive Activities is equal to Any, then risk is HIGH"

In legacy, the text is more descriptive to help the user understand:
- "Links with Ireland pose a low AML/CFT risk"

**Goal:** Understand how the Rebuild descriptions are made and how the Legacy ones are, in order to correct this issue.

## Acceptance Criteria

- [ ] We have a list of all the different risk description combos from Rebuild
- [ ] We have a list of all the different risk description combos from Legacy
- [ ] We are able to begin User Story 27695: Risk - Use more specific and easy to understand Risk descriptions

## Key Findings (Spike Results)

### Rebuild System (S1.Module.RulesEngine + S1.Module.Kyc)

**Findings:**
- Descriptions live in `Risk.Condition.Description` (NVARCHAR(255))
- Copied through to `Kyc.RiskAssessment.Description`
- 11 seeded example combos found:
  - 4× "if X is equal to Y, then risk is HIGH"
  - 1× "equal to Any"
  - 6× country-risk combos
- These are test-seed data only, excluded from prod deploy
- Real production combo list requires DB query, not source search
- CreateRiskCondition API exists (POST rules/risk/conditions) but only supports single-condition rules
- No AND/OR composite rules like Legacy has

### Legacy System (InvestorServices)

**Findings:**
- Descriptions live in `dbo.DueDiligenceTriggerRiskRating.Description` (NVARCHAR(1000))
- Authored via Admin Legacy UI: https://ci-app.sonataone.com/Admin#/DueDiligenceTriggers
- No equivalent seed/migration script exists
- Live combo list is not in source control
- Recovered field inventory (33 fields):
  - PEP
  - Sanctioned Jurisdiction
  - Sensitive Activities
  - World-Check matches
  - Others
- Sensitive Activities lookup list recovered
- Literal sentences require DB query

### Key Insights

1. **Neither system generates description sentences** — both store them as free text authored once (by an admin or seed data) and copied verbatim through every downstream layer
2. **"Making descriptions clearer" means editing stored data**, not changing a code template/SQL script
3. **There needs to be an FE ticket in Rebuild** to define the new Rules (Create, Edit, Delete)

## Dates

| Event | Date |
|-------|------|
| Created | 2026-07-29T08:51:25.887Z |
| Last Changed | 2026-08-11T11:26:57.813Z |
| State Changed | 2026-08-11T08:44:26.063Z |
| Activated | 2026-08-05T09:15:06.883Z |

## Comments

Total comments: 3

## Related Links

- [User Story 27695](https://idregister.visualstudio.com/IDR/_workitems/edit/27695) - Risk: Use more specific and easy to understand Risk descriptions
- [Azure DevOps Work Item](https://dev.azure.com/idregister/6f14ad85-6621-465d-a4f0-abb1a384e5ff/_workitems/edit/27735)
