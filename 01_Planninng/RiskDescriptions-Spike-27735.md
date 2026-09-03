# Spike 27735 — Old and New Risk Descriptions

**Ticket:** [User Story 27735](https://dev.azure.com/idregister/IDR/_workitems/edit/27735) — *Spike: What are all the old and new Risk Descriptions?*
**Feeds into:** User Story 27695 — *Risk: Use more specific and easy to understand Risk Descriptions*

**Acceptance criteria this document addresses:**
- [x] List of all the different risk description combos from Rebuild
- [x] List of all the different risk description combos from Legacy
- [ ] Begin User Story 27695 (not started — this is the input to that story)

---

## TL;DR

Neither system *generates* risk description sentences from a template. In both Rebuild and Legacy, the sentence (e.g. *"if PEP is equal to true, then risk is HIGH"*) is **free text typed once by a human** (admin or seed-data author) and stored verbatim in a database column. Downstream code only ever copies that string — it never builds it. This matters for 27695: rewriting the descriptions means rewriting stored data (or introducing an actual generator), not changing a code template.

| | **Rebuild** (`S1.Module.RulesEngine` + `S1.Module.Kyc`, `C:\Code\TemplateAPI`) | **Legacy** (`InvestorServices`, `C:\Code\IDR`) |
|---|---|---|
| Description storage | `Risk.Condition.Description` — `NVARCHAR(255)` | `dbo.DueDiligenceTriggerRiskRating.Description` — `NVARCHAR(1000)` (+ `DueDiligenceCustomRiskRating.Description` for ad-hoc entries) |
| Authored by | API (`CreateRiskCondition`/`UpdateRiskCondition`) — currently only seeded as **test data**, no prod rows in repo | Angular.js Admin CMS (`due-diligence-triggers-component.js`) — live DB only, not in source control |
| Condition engine | `CalculateRiskHandler` — `OperatorType`: `Equal`, `NotEqual`, `EqualsAny`, `GreaterThan(OrEqual)`, `LessThan(OrEqual)`, + `IsCountryRisk` | `DueDiligenceTaskEvaluator` — `OperatorType` table: `TEXT_IS_EQUAL/NOT_EQUAL`, `TEXT_CONTAINS/DOESNT_CONTAIN`, `*_BEGINS/ENDS_WITH`, `LOOKUP_IS_EQUAL/NOT_EQUAL/IS_IN/NOT_IN`, `DATE_IS_EQUAL/BEFORE/AFTER`, `BOOL_IS_TRUE/FALSE`, `*_IS_EMPTY/NOT_EMPTY` |
| Risk levels | `RiskLevel`: **Low / Medium / High** (implied — see Rebuild section) | `RiskRatingLevel`: **Low / Standard / High** (note: "Standard", not "Medium" — terminology mismatch to reconcile in 27695) |
| Source of truth | Internal to this repo — in-process call from `S1.Module.Kyc` to `S1.Module.RulesEngine`, no external API involved | Internal to `InvestorServices` — no external "Sonata One" API; "Sonata One" is just the product's own display name in the UI copy |
| Validation that description matches condition logic | None — free text, unvalidated | None — free text, unvalidated |

---

## Database tables reference

Split into "where the rule + description text is *defined*" vs "where the *result* gets recorded per profile/entity."

### Rebuild — schema `Risk` / `Kyc`

| Table | Purpose | Key columns |
|---|---|---|
| `Risk.Condition` | **Definition** — the rule itself and its free-text description | `Description NVARCHAR(255)`, `OperatorType`, `RiskLevel` |
| `Risk.ConditionValue` | The comparison value(s) a condition checks against | referenced by `ConditionId` |
| `Risk.CountryRisk` | Country → risk-level lookup (used by the `IsCountryRisk` condition type) | ~249 ISO country rows, `RiskLevel` |
| `Kyc.RiskAssessment` | **Result** — the applied/matched risk factor for a specific profile, description copied verbatim from `Risk.Condition.Description` at evaluation time | `Description`, `ConditionId`, `RiskLevel`, `Comment` |

EF configs: `S1.Module.RulesEngine\DatabaseMappings\RiskConditionConfiguration.cs`, `CountryRiskConfiguration.cs`. Schema DDL: `database\TemplateApi.DatabaseConfiguration\Risk\Tables\Condition.sql`, `CountryRisk.sql`.

### Legacy — schema `dbo`

| Table | Purpose | Key columns |
|---|---|---|
| `dbo.DueDiligenceTrigger` | **Definition** — the named rule (e.g. "PEP") | trigger name |
| `dbo.DueDiligenceTriggerConditionGroup` / `dbo.DueDiligenceTriggerCondition` | The AND/OR condition logic for a trigger | `FK_DueDiligenceFieldID`, `FK_OperatorTypeID`, `Value` |
| `dbo.DueDiligenceField` | The field a condition checks (PEP, sanctioned jurisdiction, etc.) | `Code` (e.g. `IS_PEP`) |
| `dbo.DueDiligenceTriggerRiskRating` | **Definition** — the free-text description + risk level tied to a trigger | `Description NVARCHAR(1000)`, `FK_RiskRatingLevelID` |
| `dbo.RiskRatingLevel` | Low/Standard/High level names | seeded 1/2/3 |
| `dbo.DueDiligenceRiskRating` | **Result** — the applied risk rating for a specific entity, referencing the matched `DueDiligenceTriggerRiskRating` | `FK_DueDiligenceTriggerRiskRatingID` |
| `dbo.DueDiligenceCustomRiskRating` | **Result** — manual, rule-independent entries added directly via the "Add New Item" UI (no trigger/condition at all) | `Description NVARCHAR(1000)`, level |

Schema DDL: `InvestorServices.DatabaseConfiguration\dbo\Tables\DueDiligenceTriggerRiskRating.sql` etc.

**Note for 27695:** in both systems, editing the description text means editing rows in the *definition* table (`Risk.Condition` / `dbo.DueDiligenceTriggerRiskRating`) — not the *result* tables, which just hold copies. Legacy's live trigger/description rows aren't in source control, so seeing the actual current values means querying `dbo.DueDiligenceTriggerRiskRating` (joined to `DueDiligenceTrigger` and `RiskRatingLevel`) on the live/staging DB directly.

---

## Rebuild (new system) — `C:\Code\TemplateAPI`

### Pipeline
```
Risk.Condition.Description (DB, free text)
  → S1.Module.RulesEngine: CalculateRiskHandler evaluates conditions,
    copies matched condition's Description into AppliedRiskConditionDto
  → S1.Module.Kyc: RiskService.CalculateRiskLevelAsync (internal service call)
  → SubmitQuestionnaireHandler.UpdateRiskAssessmentsAsync copies
    AppliedRiskConditionDto.Description → RiskAssessment.Description (DB)
  → GetRiskAssessmentsHandler reads RiskAssessment rows for display
```
No `string.Format`/interpolation exists anywhere in this path. Confirmed by unit tests using arbitrary placeholder text (`"Desc"`, `"Risk by country"`) that the handler never inspects the description's content or shape.

Key files:
- `S1.Module.RulesEngine\Domain\RiskCondition.cs` — `Description` is a plain `string?`
- `S1.Module.RulesEngine\Features\CalculateRisk\CalculateRiskHandler.cs` — the evaluator
- `S1.Module.RulesEngine\Features\CreateRiskCondition\CreateRiskConditionHandler.cs` — admin API writes description straight from request body
- `S1.Module.Kyc\Domain\RiskAssessment.cs`, `S1.Module.Kyc\Services\RiskService.cs`
- `S1.Module.Kyc\Features\Questionnaires\SubmitQuestionnaire\SubmitQuestionnaireHandler.cs`

### Combo list (from seed data — `database\TemplateApi.DatabaseConfiguration\Risk\TestData\Condition.sql`)

⚠️ **These are TEST-only seed rows**, excluded from production by `PostDeploymentScript.sql` (`IF DB_NAME() <> 'prod-user-api'`). No production risk-condition catalog currently exists in this repo — production rows only exist if an admin has created them via the API.

| # | Condition Name | Description (verbatim) | Operator | Level |
|---|---|---|---|---|
| 1 | PEP - EQUAL - HIGH | "if PEP is equal to true, then risk is HIGH" | Equal | High |
| 9 | Affiliated Activities In Sanctioned Jurisdiction - EQUAL - HIGH | "if Affiliated Aactivities In Sanctioned Jurisdiction is equal to true, then risk is HIGH" *(sic — typo "Aactivities" in seed data)* | Equal | High |
| 10 | Issues Bearer Shares - EQUAL - HIGH | "if Issues Bearer Shares is equal to true, then risk is HIGH" | Equal | High |
| 11 | Sensitive Activities - EQUALANY - HIGH | "if Sensitive Activities is equal to Any, then risk is HIGH" | EqualsAny | High |
| 2–8 | Country-risk conditions (Country Of Birth, Country Of Citizenship, Nationality, Country Of Incorperation *(sic)*, Country Of Tax Residency, Country Of Funds Raised, Country Of Funds Distributed) | "Set [field] from CountryRisk table" style phrasing — level resolved separately via `Risk.CountryRisk` lookup (~249 ISO country rows) | IsCountryRisk | Varies by country (1/2/3) |

**Totals:** 11 seeded `RiskCondition` rows → 4 "if X is equal to Y, then risk is Z" sentences, 1 "equal to Any" variant, 6 country-risk sentences. 8 distinct mechanical operator/condition *types* exist in code, but the number of distinct *sentences* is unbounded — `Description` is free text, so this list reflects only what's currently seeded, not a hard limit.

Underlying questions referenced: `Question.sql` id 82 ("Are you or an affiliate undertaking any activities in a sanctioned jurisdiction?"), id 84 ("Do you undertake any of the following sensitive activities?").

---

## Legacy (old system) — `C:\Code\IDR` (InvestorServices)

### Pipeline
```
DueDiligenceTriggerRiskRating.Description (DB, free text, authored via Admin CMS)
  → DueDiligenceTriggerService.ApplyTriggers() raises RunDueDiligenceEvent
  → DueDiligenceTaskEvaluator.IsRequired() evaluates condition groups (AND) / groups (OR)
    via DueDiligenceProfileDelegates (field extraction) + FieldConditionDelegateLookup (operators)
  → On match: DueDiligenceRiskRating row created, referencing the trigger's
    pre-authored Description + RiskRatingLevel
  → EntityRiskRatingService.cs: riskRatingResult.Description = trigger.Description (straight copy)
  → profile-risk-rating.html renders the "Indicative Risk Factors" panel
    (this IS the "Sonata One has highlighted the following risk factors" panel
    quoted in the ticket screenshot — title "Indicative Risk Factors", columns
    Description / Level / Comments / Reviewed Level / Explanatory Documents)
```
Also feeds SSRS reports (`Risk Factor Summary.rdl`, `RegCo Clients.rdl`, `MLCO.rdl`) via straight `SELECT Description AS IndicativeRiskFactor` — no formatting there either.

Key files:
- `dbo\Tables\DueDiligenceTriggerRiskRating.sql` — `Description NVARCHAR(1000)`
- `InvestorServices.Web\App\Admin\DueDiligenceTriggers\due-diligence-triggers-component.js` / `.html` — admin CMS authoring screen
- `InvestorServices.Api\Controllers\V1\Entities\Services\DueDiligenceTriggerService.cs` — `GetRiskRatings`/`SaveRiskRatings`
- `InvestorServices.General\Processors\Tasking\Tasks\DueDiligence\DueDiligenceTaskEvaluator.cs` — the evaluator
- `InvestorServices.General\Processors\Tasking\Tasks\DueDiligence\DueDiligenceProfileDelegates.cs` — field code dictionary
- `InvestorServices.Web\App\Entity\Profile\Components\profile-risk-rating.html` — the actual "Indicative Risk Factors" UI panel

### Combo list

⚠️ **The actual live trigger names and their exact `Description` sentences are NOT in source control** — they exist only in the live/staging database and are entirely admin-authored via the CMS. The table below is the field-code inventory recoverable from source (`DueDiligenceProfileDelegates.cs`), not the literal sentences. **To get the exact legacy sentences for 27695, query the live/staging DB `dbo.DueDiligenceTriggerRiskRating` table directly.**

**Field codes with matching examples in the ticket:**

| Field Code | What it checks | Ticket example match |
|---|---|---|
| `IS_PEP` | Politically exposed person | "if PEP is equal to true, then risk is HIGH" ✅ |
| `IS_SANCTIONED` | Sanctioned jurisdiction activities | "if Affiliated Activities in Sanctioned Jurisdiction is equal to true, then risk is HIGH" ✅ |
| `SENSITIVE_ACTIVITY` | Sensitive activities (lookup list below) | "if Sensitive Activities is equal to Any, then risk is HIGH" ✅ |
| `IS_BEARER` | Unregistered bearer shares | matches Rebuild's "Issues Bearer Shares" condition |

**Full field-code inventory** (33 fields, `DueDiligenceProfileDelegates.cs`): `NAME`, `TITLE`, `GENDER`, `FORMER_NAME`, `NATIONALITY`, `TAX_COUNTRY`, `BIRTH_DATE`, `BIRTH_PLACE`, `EXCHANGE`, `REGULATOR`, `REG_NO`, `INCORPORATION_COUNTRY`, `INCORPORATION_DATE`, `SCHEME_NAME`, `ACTIVITIES_OVERVIEW`, `ECONOMIC_JURISDICTION`, `IS_PEP`, `IS_STATE_OWNED`, `UBO_WEALTH_SOURCE`(+`_TEXT`), `ENTITY_TYPE`, `CITY`/`POSTCODE`/`STATE`/`COUNTRY`, `SENSITIVE_ACTIVITY`, `IS_BEARER`, `IS_RENOUNCED_US`, `PRIMARY_FUNDS_RAISED`/`DISBURSED`, `IS_SANCTIONED`, `IS_WORLD_CHECK_PEP`/`_SANCTIONS`/`_LAW_ENFORCEMENT`, `LINKED_TO_FUND_JURISDICTION_FOR_FUND`/`INVESTOR`, `LINKED_TO_SERVICE_FOR_FUND`/`INVESTOR`, `CHAPTER_3_STATUS`, `IS_NON_PROFIT_ORGANIZATION`, `IS_REGISTERRD_CHARITY` *(sic)*, `IS_SUBSCRIBED_TO_FATCA`, `ENTITY_PARTNER`.

**"Sensitive Activities" lookup values** (`dbo\Data\Lookup.sql`, `LookupTypeID = 14`) — a concrete, source-controlled list, unlike the trigger descriptions themselves:
- Trading requiring advance payment with no consumer protection
- Military goods/equipment/technology/personnel
- Pharmaceutical goods/devices manufacturing or marketing
- Scientific research
- Dual-use goods for sanctioned activities
- Vulture funds
- Mining, drilling, or quarrying for natural resources
- Cash-intensive businesses (restaurants, convenience stores, petrol stations, vending, beauty salons)
- Gambling/betting/casino businesses
- Dealing in virtual currencies
- Other sensitive activities / No sensitive activities

**Related "Core DD" questions** (`dbo\Data\DueDiligenceCoreDDQuestions.sql`, 73 rows) that map to a separate risk-escalation mechanism — notable matches: Q46/Q53 (sanctioned jurisdiction), Q47/Q54 (PEP), Q82 (sensitive activities), Q28 (state-owned corp executive), Q26 (bearer shares), Q31/Q32 (non-profit/charity).

**Additional structures found, not directly description-related but relevant to 27695 scope:**
- A second, largely dead/commented-out admin-task rule engine (`AdminRuleTriggerService`) exists — architecturally similar but unrelated to risk ratings; flagged so it isn't confused with the risk engine.
- End users can also add a fully manual, rule-independent `DueDiligenceCustomRiskRating` ("Add New Item" on the Risk Rating panel) — free text with no condition/trigger behind it at all.

---

## SQL queries to pull the live data

Schema confirmed against the DDL in each repo (`Risk.Condition`, `Kyc.Question`, `Risk.ConditionValue`, `Kyc.RiskAssessment` for Rebuild; `dbo.DueDiligenceTrigger`, `DueDiligenceTriggerRiskRating`, `RiskRatingLevel`, `DueDiligenceCustomRiskRating`, `DueDiligenceRiskRating` for Legacy). `OperatorType`/`RiskLevel` are stored as raw `INT` in Rebuild with no lookup table — decoded below from `S1.Shared.Common.Models.Enums.OperatorType`/`RiskLevel`.

### Rebuild — run against the `TemplateApi` (Rebuild) database

**1. All defined risk conditions (the "rule catalog") — this is the main combo list:**
```sql
SELECT
    c.Id,
    c.Name,
    c.Description,
    q.Text                                  AS QuestionText,
    CASE c.OperatorType
        WHEN 1 THEN 'Equal'
        WHEN 2 THEN 'Not Equal'
        WHEN 3 THEN 'Greater Than'
        WHEN 4 THEN 'Greater Than Or Equal'
        WHEN 5 THEN 'Less Than'
        WHEN 6 THEN 'Less Than Or Equal'
        WHEN 7 THEN 'Equals Any'
    END                                      AS Operator,
    c.IsCountryRisk,
    CASE c.RiskLevel
        WHEN 1 THEN 'Low'
        WHEN 2 THEN 'Medium'
        WHEN 3 THEN 'High'
    END                                      AS RiskLevel,
    c.IsActive
FROM Risk.Condition c
JOIN Kyc.Question q ON q.Id = c.QuestionIdRef
ORDER BY c.IsActive DESC, c.Name;
```

**2. Comparison values behind each condition (what "equal to X" actually checks):**
```sql
SELECT
    c.Id            AS ConditionId,
    c.Name          AS ConditionName,
    cv.Value,
    cv.AnswerTypeRef
FROM Risk.ConditionValue cv
JOIN Risk.Condition c ON c.Id = cv.ConditionIdRef
ORDER BY c.Name, cv.Value;
```

**3. Country-risk lookup (backs the `IsCountryRisk = 1` conditions):**
```sql
SELECT
    CountryCode,
    CASE RiskLevel WHEN 1 THEN 'Low' WHEN 2 THEN 'Medium' WHEN 3 THEN 'High' END AS RiskLevel,
    IsActive
FROM Risk.CountryRisk
ORDER BY RiskLevel DESC, CountryCode;
```

**4. Descriptions actually applied to real profiles (the ground truth of what users have seen — most valuable for 27695):**
```sql
SELECT
    ra.Description,
    CASE ra.RiskLevel WHEN 1 THEN 'Low' WHEN 2 THEN 'Medium' WHEN 3 THEN 'High' ELSE 'Unset' END AS RiskLevel,
    COUNT(*)              AS TimesApplied,
    COUNT(DISTINCT ra.ProfileIdRef) AS DistinctProfiles
FROM Kyc.RiskAssessment ra
WHERE ra.IsActive = 1
GROUP BY ra.Description, ra.RiskLevel
ORDER BY TimesApplied DESC;
```

### Legacy — run against the `InvestorServices` (Legacy) database

**1. All defined trigger risk ratings (the "rule catalog") — this is the main combo list:**
```sql
SELECT
    t.DueDiligenceTriggerID,
    t.DueDiligenceTriggerName,
    rr.Description,
    lvl.Name        AS RiskLevel,
    rr.IsActive      AS RatingIsActive,
    t.IsActive       AS TriggerIsActive
FROM dbo.DueDiligenceTriggerRiskRating rr
JOIN dbo.DueDiligenceTrigger t   ON t.DueDiligenceTriggerID = rr.FK_DueDiligenceTriggerID
JOIN dbo.RiskRatingLevel lvl     ON lvl.RiskRatingLevelID   = rr.FK_RiskRatingLevelID
ORDER BY t.IsActive DESC, t.DueDiligenceTriggerName, lvl.RiskRatingLevelID;
```

**2. Ad-hoc custom risk ratings (manually added, no trigger/condition behind them):**
```sql
SELECT
    cr.DueDiligenceCustomRiskRatingID,
    cr.Description,
    lvl.Name        AS RiskLevel,
    cr.DateCreated,
    cr.IsActive
FROM dbo.DueDiligenceCustomRiskRating cr
JOIN dbo.RiskRatingLevel lvl ON lvl.RiskRatingLevelID = cr.FK_RiskRatingLevelID
ORDER BY cr.DateCreated DESC;
```

**3. Descriptions actually applied to real entities (ground truth — covers both trigger-based and custom ratings, most valuable for 27695):**
```sql
SELECT
    COALESCE(rr.Description, cr.Description)          AS Description,
    COALESCE(trigLvl.Name, custLvl.Name, finalLvl.Name) AS RiskLevel,
    t.DueDiligenceTriggerName,
    COUNT(*)                          AS TimesApplied,
    COUNT(DISTINCT d.FK_EntityID)     AS DistinctEntities
FROM dbo.DueDiligenceRiskRating d
LEFT JOIN dbo.DueDiligenceTriggerRiskRating rr ON rr.DueDiligenceTriggerRiskRatingID = d.FK_DueDiligenceTriggerRiskRatingID
LEFT JOIN dbo.DueDiligenceTrigger t            ON t.DueDiligenceTriggerID = rr.FK_DueDiligenceTriggerID
LEFT JOIN dbo.RiskRatingLevel trigLvl          ON trigLvl.RiskRatingLevelID = rr.FK_RiskRatingLevelID
LEFT JOIN dbo.DueDiligenceCustomRiskRating cr  ON cr.DueDiligenceCustomRiskRatingID = d.FK_DueDiligenceCustomRiskRatingID
LEFT JOIN dbo.RiskRatingLevel custLvl          ON custLvl.RiskRatingLevelID = cr.FK_RiskRatingLevelID
LEFT JOIN dbo.RiskRatingLevel finalLvl         ON finalLvl.RiskRatingLevelID = d.FK_FinalRiskRatingLevelID
WHERE d.IsActive = 1
GROUP BY COALESCE(rr.Description, cr.Description), COALESCE(trigLvl.Name, custLvl.Name, finalLvl.Name), t.DueDiligenceTriggerName
ORDER BY TimesApplied DESC;
```

> Query 3 uses `FK_FinalRiskRatingLevelID` as a fallback level in case the linked trigger/custom rating's level was later changed and diverged from what was actually applied at the time — worth eyeballing whether `trigLvl`/`custLvl` and `finalLvl` ever disagree, since that itself would be a data-quality finding for 27695.

---

## Rebuild's shortcomings vs. Legacy

`S1.Module.RulesEngine` has exactly 6 features total: `CalculateRisk`, `CreateRiskCondition`, `UpdateRiskCondition`, `DeleteRiskCondition`, `GetRiskConditions`, `GetQuestionFields`. Nothing else — that narrow surface area is the source of most gaps below.

1. **No composite rules — only single-condition triggers.** `Risk.Condition` is flat: one row = one question + one operator + one risk level (`Domain/RiskCondition.cs`). There's no equivalent of Legacy's `DueDiligenceTriggerConditionGroup`/`DueDiligenceTriggerCondition` structure, which lets one trigger fire on multiple conditions combined with AND/OR (e.g. "sanctioned jurisdiction AND high-risk entity type"). Rebuild cannot express a compound rule today.

2. **Extension point is questionnaire-only — no screening or relationship signals.** `Risk.Condition.QuestionIdRef` is the only FK a condition can evaluate against (schema-enforced). Legacy's field inventory includes `IS_WORLD_CHECK_PEP`, `IS_WORLD_CHECK_SANCTIONS`, `IS_WORLD_CHECK_LAW_ENFORCEMENT` (real-time screening match results) and `LINKED_TO_FUND_JURISDICTION_FOR_FUND/INVESTOR` (relationship/geography data) as condition inputs. Rebuild's rules engine has no path to react to IdPal/screening outcomes or connection-graph data, despite the KYC module having an IdPal integration it could theoretically draw on.

3. **Country risk is a hardcoded special case, not a general capability.** `Risk.Condition` has an `IsCountryRisk BIT` column with a `CHECK` constraint forcing `RiskLevel`/`OperatorType` to `NULL` when set (`Condition.sql:18-21`) — country-risk had to be bolted on as schema-level special-casing. Legacy handles the same need generically: `TAX_COUNTRY`/`NATIONALITY`/etc. are ordinary fields evaluated with an ordinary `LOOKUP_IS_IN` operator. Rebuild's operator set (`Equal`, `NotEqual`, `EqualsAny`, `GreaterThan(OrEqual)`, `LessThan(OrEqual)`) has no "look up value in reference table" operator, so it couldn't express country-risk generically.

4. **No bulk re-evaluation when rules change.** `CalculateRiskLevelAsync` is only ever called from `SubmitQuestionnaireHandler`, `SubmitMergedQuestionnaireSectionHandler`, and the two `GetQuestionnaire(Section)` handlers — risk is (re)computed at questionnaire submit/view time only. There's no batch job or admin-triggered endpoint to re-run risk against existing profiles when a `RiskCondition` is added or edited. Legacy's "Apply Triggers" button is coarse and slow, but it exists; Rebuild has no equivalent, so changing a rule silently only affects future submissions.

5. **No manual/ad-hoc risk entry.** `Features/RiskAssessments` only contains `UpdateRiskAssessmentComment` and `UploadRiskAssessmentDocument` — nothing to add a rule-independent risk item the way Legacy's `DueDiligenceCustomRiskRating` ("Add New Item") lets an analyst flag something the rule engine didn't catch.

6. **No per-trigger evidence requirements or reportable-jurisdiction linkage.** Legacy ties evidence requirements (`DueDiligenceTriggerEvidenceRequirement`, confirmed live via the Withholding 1042 migration) and reportable jurisdictions directly to a trigger, so a firing risk factor can mandate a specific document upload. Rebuild's `RiskCondition` → `RiskAssessment` has no such linkage; evidence upload is manual and untied to which condition fired.

7. **Description field is *shorter*, not longer — works against 27695's own goal.** `Risk.Condition.Description` is `NVARCHAR(255)` (also enforced in `CreateRiskConditionRequestValidator`: `MaximumLength(255)`); Legacy's `DueDiligenceTriggerRiskRating.Description` is `NVARCHAR(1000)`. Story 27695 wants "more specific and easy to understand" descriptions, but Rebuild's schema has less room for that than the system it's replacing — worth resizing before that story starts.

8. **No "test this rule" tooling.** Legacy's admin CMS has a "Test Triggers" action to dry-run a rule against a specific entity before it goes live. Rebuild's `CreateRiskCondition`/`UpdateRiskCondition` have no preview/dry-run equivalent — a new condition's real behavior is only visible once a real questionnaire submission hits it.

9. **No admin authoring UI confirmed in this repo.** Legacy's Angular CMS (`due-diligence-triggers-component.js`) is a full no-code rule editor. Rebuild only exposes a plain CRUD API. The frontend repos weren't in scope for this search, so this is flagged as unconfirmed rather than a verified gap — worth checking separately.

**Shared weakness (not Rebuild-specific):** both systems store descriptions as free text with zero enforced link to the actual condition logic — that design carried over rather than being fixed in the rebuild.

---

## API: creating Risk Conditions in Rebuild

Yes — `POST /rules/risk/conditions` in `S1.Module.RulesEngine.Features.CreateRiskCondition`.

**Route:** `POST rules/risk/conditions`
**Auth:** `[Authorize]` (any authenticated user — no role restriction visible at the endpoint)
**Feature-gated:** `[FeatureGate(FeatureOptions.RulesEngine)]` — can be switched off per environment via feature flag.

**Request body** (`CreateRiskConditionRequest`):
```csharp
{
  "name": "string, required, max 50 chars",
  "description": "string, optional, max 255 chars",
  "questionIdRef": "int, required, must be > 0 (FK to Kyc.Question)",
  "isCountryRisk": "bool, default false",
  "operatorType": "OperatorType enum, required unless isCountryRisk=true, must be null if isCountryRisk=true",
  "riskLevel": "RiskLevel enum, required unless isCountryRisk=true, must be null if isCountryRisk=true",
  "conditionValues": "RiskConditionValueDto[], required (empty allowed only when isCountryRisk=true)"
}
```

**Validation** (`CreateRiskConditionRequestValidator`, FluentValidation):
- `Name` non-empty, ≤50 chars
- `Description` ≤255 chars
- `QuestionIdRef` > 0
- If `IsCountryRisk = false`: `OperatorType` and `RiskLevel` must both be set to defined enum values, and at least one `ConditionValue` is required
- If `IsCountryRisk = true`: `OperatorType` and `RiskLevel` must both be `null` (enforced both here and by the DB `CHECK` constraint)

**Handler logic** (`CreateRiskConditionHandler`):
- Requires an authenticated user
- Rejects duplicates: fails if a condition with the same `Name` already exists, or if one already exists for the same `QuestionIdRef` + `OperatorType` combo
- Branches on `IsCountryRisk`: `true` → `RiskCondition.CreateCountryRisk(...)` (no operator/level/values); `false` → `RiskCondition.Create(...)` plus each `RiskConditionValue`
- Persists via `IRepository<RiskCondition>.UpsertAsync`, returns the created condition including its generated `Id` and echoed-back `ConditionValues`

**Companions in the same feature area:** `UpdateRiskCondition` (`PUT`, same shape), `DeleteRiskCondition`, `GetRiskConditions` (list), `GetQuestionFields` (returns questions + valid operators/lookup options for whatever UI builds the request), and `CalculateRisk` (the evaluator that consumes what's created here).

The API exists and is reasonably well validated, but it only supports the single-condition shape described in the shortcomings section above — one question, one operator, one set of values, one risk level per condition. There's no way through this API to compose multiple conditions into one trigger the way Legacy's condition-group model allows.

---

## Notes for User Story 27695 (rewrite the descriptions)

1. **Both systems store descriptions as unvalidated free text with no enforced link to the underlying condition logic.** Drift between "what the rule checks" and "what the sentence says" is structurally possible in both — worth deciding whether 27695 should introduce an actual generated-description mechanism (build the sentence from the condition's field/operator/value at read time) rather than continuing to hand-author text.
2. **Terminology mismatch:** Legacy uses Low/Standard/High; ticket examples and Rebuild's seed data use Low/Medium/High-style wording. Reconcile before rewriting.
3. **No production risk-condition data exists in the Rebuild repo yet** — the 11-row combo list above is test/seed data only. The real "Rebuild" combo inventory should also be pulled from whatever environment currently has admin-authored `RiskCondition` rows, if any exist beyond test.
4. **The true legacy combo list lives only in the live DB**, not source control — this spike's legacy findings identify *where* to query (`dbo.DueDiligenceTriggerRiskRating`, join `DueDiligenceTrigger` + `RiskRatingLevel`), but pulling the actual current sentences requires DB access, not just code search.
5. Known seed-data typos worth deciding whether to carry forward or fix: "Affiliated **Aactivities**" (Rebuild), "**Incorperation**" (Rebuild), "**REGISTERRD** Charity" (Legacy field code).

---

*Compiled from two codebase searches: `S1.Module.RulesEngine`/`S1.Module.Kyc` (Rebuild, `C:\Code\TemplateAPI`) and `InvestorServices` (Legacy, `C:\Code\IDR`). No production/staging database was queried — all findings are from source code and committed seed data only.*
