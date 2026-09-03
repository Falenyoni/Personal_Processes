# IDR Tax Compliance Domain — Complete Extract

> Extracted from `C:\Code\IDR` (InvestorServices.DD, InvestorServices.General,
> InvestorServices.Api, InvestorServices.DatabaseConfiguration, Components/ISC.BulkTIEProcessing,
> InvestorServices.SSRS). All evidence is sourced from code, SQL schema, and report definitions.

---

## Table of Contents

1. [Domain Overview](#1-domain-overview)
2. [Regulatory Frameworks Covered](#2-regulatory-frameworks-covered)
3. [TIE — Tax Information Exchange Umbrella](#3-tie--tax-information-exchange-umbrella)
4. [FATCA (US)](#4-fatca-us)
5. [CRS (Common Reporting Standard)](#5-crs-common-reporting-standard)
6. [HMRC / UK FATCA](#6-hmrc--uk-fatca)
7. [Withholding — Form 1042 / 1042-S](#7-withholding--form-1042--1042-s)
8. [W-Forms (W-Series)](#8-w-forms-w-series)
9. [Classification Workflow](#9-classification-workflow)
10. [Registration Module](#10-registration-module)
11. [Investigation (Financial Data)](#11-investigation-financial-data)
12. [Reportable Jurisdictions](#12-reportable-jurisdictions)
13. [Tax Residences](#13-tax-residences)
14. [Treaty Rates](#14-treaty-rates)
15. [TIN Strategy by Country](#15-tin-strategy-by-country)
16. [Bulk XML Generation & IGOR Submission](#16-bulk-xml-generation--igor-submission)
17. [Report Lifecycle (Generate → Submit → Resubmit)](#17-report-lifecycle-generate--submit--resubmit)
18. [Scheduled Tasks & Cache Jobs](#18-scheduled-tasks--cache-jobs)
19. [SSRS Reports](#19-ssrs-reports)
20. [Controller Surface](#20-controller-surface)
21. [Key Business Rules](#21-key-business-rules)
22. [Database Schema Map](#22-database-schema-map)
23. [Architecture Notes for Rebuild](#23-architecture-notes-for-rebuild)

---

## 1. Domain Overview

Tax compliance in IDR is one of its largest and most complex domains. It spans:

- **FATCA** — US-origin, IRS-driven, mandatory for all financial institutions globally
- **CRS** — OECD standard, adopted by 100+ countries, per-jurisdiction reporting
- **HMRC AEOI** — UK-specific CRS variant with bespoke portal (`HmrcAeoiPortalID`)
- **Form 1042 / 1042-S** — US withholding on payments to foreign persons
- **W-Forms** — IRS certification forms (W-9, W-8BEN, W-8BEN-E, W-8ECI, W-8EXP, W-8IMY) executed via DocuSign
- **Tax Residences** — per-entity self-reported and system-verified residency data
- **Treaty Rates** — per-country reduced withholding rates for income types

The umbrella concept unifying all these is **TIE (Tax Information Exchange)**.

The domain constitutes approximately **85+ controllers/services** in the monolith and has its own:
- dedicated Angular UI pages
- SSRS reporting suite (25+ reports)
- standalone bulk processing component (`ISC.BulkTIEProcessing`)
- separate test project (`ISC.TaxInformationExchange.Test`)
- IGOR portal integration for IRS submission

---

## 2. Regulatory Frameworks Covered

| Enum Value | Description | Authority | XML Format |
|---|---|---|---|
| `TIEReportType.US` | FATCA | IRS (US) | OECD CRS XML v1/v2 |
| `TIEReportType.CRS` | Common Reporting Standard | OECD / local tax authority | CRS OECD XML (jurisdiction-specific) |
| `TIEReportType.HMRC` | UK AEOI (UK FATCA equivalent) | HMRC | Bespoke HMRC XML |
| `TIEReportType.USFatca` | US FATCA (alternate channel) | IRS | FATCA XML |
| `TIEReportType.Form1042` | Annual Withholding Return | IRS | ASCII fixed-width |
| `TIEReportType.Form1042s` | Statement to Foreign Persons | IRS | ASCII fixed-width |

**Chapter indicators** (IRS classification):
- `ChapterIndicators.Chapter3` = income classification (dividends, interest, royalties)
- `ChapterIndicators.Chapter4` = FATCA status (FFI, NPFFI, US person, etc.)

---

## 3. TIE — Tax Information Exchange Umbrella

### Core DB Tables

```
TIERegistration       — One per entity: identifiers (GIIN, HMRC ID, Lux CCSS, Cayman FI, Singapore IDs, Swiss Tax ID)
TIEPeriod             — Reporting year periods (Name, Code, StartDate, EndDate, IsAvailable)
TIEReport             — Generated XML/JSON report per entity per period per type
TIEReportType         — Reference: US / CRS / HMRC / USFatca / Form1042 / Form1042s
TIEReportStatus       — Status lookup (Draft, Generated, Validated, Submitted, Resubmitted etc.)
TIEReportSchema       — Schema version config per report type
TIEReportMessageType  — Message type (NewData, Correction, Void, etc.)
TIEReportSplit        — Large reports split into multiple files (France/Lux/Spain splitters)
TIEReportIGORSubmission   — Track IGOR portal submission per report
TIEReportSubmissionHistory — Audit trail of submissions
TIEInvestigation      — Financial data: AccountBalance, Dividend, Interest, Proceeds, OtherPayment
TIEEntityInvestigation — Link entity to investigation record
TIEEntityInvestigation1042  — 1042 specific data with DocuSign approval workflow
TIEEntityInvestigation1042s — 1042-S specific
TIEInvestigationDraft — Draft investigation data before confirmation
TIEClassification     — Per-entity US + CRS classification with 3-level approval
TIEClassificationQuestion / QuestionSet / Answer — Classification questionnaire
TIEClassificationDocument — Supporting docs for classification
TIEContact            — FATCA/CRS contact person per entity
TIEContactCategory    — Category of contact (primary, secondary, etc.)
TIEDocument           — Documents attached to TIE workflows
TIEEntitySubmissionCompliance — Cayman compliance submission tracking
TIEWithholdingReport  — 1042/1042-S withholding report (ASCII format)
TIEWithholdingReportSubmissionHistory
```

### Key Relationships

```
Entity (1) ──── (1) TIERegistration
Entity (1) ──── (N) TIEClassification  [per classification round]
Entity (1) ──── (N) TIEReport           [per period × per type]
Entity (1) ──── (N) TIEInvestigation    [per period × per relationship]
Entity (1) ──── (N) DueDiligenceReportableJurisdiction
Relationship (N) ── (1) TIEInvestigation [financial data per investor-fund relationship]
```

---

## 4. FATCA (US)

### What IDR Does

1. **Registration**: Capture GIIN, assign Responsible Officer, record IRS FATCA login credentials, track `IsRegistered`, `IsRegisteredWithIgor`, `BeginReportingFromYear`
2. **Classification**: Determine US Chapter 3 + Chapter 4 status for each entity via questionnaire → IDR approval → optional KPMG approval
3. **Investigation**: Collect financial data (account balance, dividends, interest, proceeds, other) per investor-fund relationship per period
4. **Report generation**: Produce IRS OECD CRS XML with `IrsGenerator`, jurisdiction-specific variants for Lux (`FatcaLuxGenerator`), France (`FranceIrsGenerator`), Finland (`FinlandIrsGenerator`)
5. **Submission**: Via IGOR portal (`IsRegisteredWithIgor`), tracked in `TIEReportIGORSubmission`

### Key Source Files

```
InvestorServices.General/FatcaAndCrs/Generation/IRS/IrsGenerator.cs
InvestorServices.General/FatcaAndCrs/Generation/IRS/LUX/FatcaLuxGenerator.cs
InvestorServices.General/FatcaAndCrs/Generation/IRS/FranceIrsGenerator.cs
InvestorServices.General/FatcaAndCrs/Generation/IRS/FinlandIrsGenerator.cs
InvestorServices.General/FatcaAndCrs/Generation/UsTinGenerator.cs
InvestorServices.General/FatcaAndCrs/Generation/IRS/SubstantialOwnerFilter.cs
InvestorServices.General/FatcaAndCrs/Generation/IRS/IrsTinGenerator.cs
Components/ISC.BulkTIEProcessing/Domain/Igor/IgorFatcaBulkProcessor.cs
```

### FATCA Dashboard Features

From Angular + API services:
- FATCA/CRS combined dashboard showing registration, investigation, reporting status per entity
- **Chart service** (`FatcaCrsDashboardChartController`) — aggregated charts by status
- **Financial data view** per entity per period
- **W-Form dashboard** — separate view for W-Form status and expiry tracking
- **Withholding dashboard** — W-Form beneficiary withholding view

### Classification — Chapter 3 Special Cases

```csharp
// TieClassificationChapter3StatusEnum
SingleMemberLLC = 1    // "Sole proprietor or single-member LLC (US)"
DisregardedEntity = 2  // "Disregarded entity"
```

These affect how the entity appears in IRS FATCA XML (the entity is treated as its owner for classification purposes).

### FATCA Scheduled Tasks

```sql
-- system/Tables/FatcaScheduledTask.sql: scheduled background tasks for:
-- CacheEntityFatcaCrsData
-- CacheFatcaCrsReportableAccounts (monthly)
-- FatcaCrsMonthlyReportUpdate
-- InactiveActiveFatcaCrsMonthlyReportUpdate
```

---

## 5. CRS (Common Reporting Standard)

### Architecture

CRS uses a **jurisdiction-specific generator factory** pattern. Each jurisdiction has its own generator class that inherits from the base `CrsGenerator`.

```
CrsGenerator (base)
├── CrsCaymanGenerator          → Cayman Islands (KY)
├── CrsFranceGenerator          → France (FR) [+ CrsFranceAccountReportGeneratorFactory]
├── CrsLuxGenerator             → Luxembourg (LU) [+ CrsLuxembourgPassiveNfeAccountReportGenerator]
│   └── LuxCrsBodySplitter      → splits large Lux CRS reports by body
├── CrsSpainGenerator           → Spain (ES) [+ SpainSplitGenerator]
├── CrsFinlandGenerator         → Finland (FI)
└── Default                     → All other jurisdictions
```

**Additional report generator types:**
- `DefaultCrsActiveAccountReportGenerator` — standard account holder
- `DefaultCrsPassiveNfeAccountReportGenerator` — passive non-financial entity
- `CrsCaymanPassiveNfeAccountReportGenerator` — Cayman-specific passive NFE
- `CrsSingaporePassiveNfeAccountReportGenerator` — Singapore-specific passive NFE
- `CrsActiveAccountSplitReportGenerator` — split report for active accounts

### CRS Report Structure (OECD XML)

Top-level `CrsOecd` object:
```
CrsOecd
├── Version          (standard "1.0" or Switzerland uses "2.0")
├── InfoDaten        (Austria-specific: FastnrFonTn, FastnrFi, Vers="01.00")
├── MessageSpec      (MessageRefId, SendingCompanyIN=GIIN, MessageType, ReceivingCountry, TransmittingCountry)
└── CrsBody[]        (per-account-holder block)
    ├── AccountHolder
    ├── AccountReport (financial data)
    └── ControllingPerson[] (beneficial owners/controllers)
```

### CRS Account Report Field Mapping

From `TIEInvestigation`:
| DB Column | CRS XML Element |
|---|---|
| `AccountBalance` | Account Balance |
| `Dividend` | Dividends |
| `Interest` | Interest |
| `ProceedsAndRedemption` | Other Income (Proceeds / Redemption) |
| `OtherPayment` | Other Income |
| `AccountNumber` | Account Number |
| `ReportableJurisdictions` | Determines which CRS bodies are generated |

### CRS Non-Default TIN Countries

Countries where IDR does **not** use default TIN logic (handled by `CrsTinGenerator` with `nonDefaultTinCountries`):
- `KY` — Cayman Islands
- `VG` — British Virgin Islands
- `BS` — Bahamas
- `GG` — Guernsey

For these countries TIN generation uses fallback or `NOTIN` patterns.

### Cayman-Specific CRS (Extra Compliance)

Cayman Islands has additional compliance requirements:
- `CaymanCrsComplianceConfig` table tracks Cayman-specific config
- `Cayman_GetCRSComplianceReportBulk` stored procedure
- `FatcaCrsBulkCaymanCrsCompliance.xlsx` — bulk registration template
- `cayman-crs-compliance.html` — dedicated UI page
- AML/KYC obligation tracking (`IsAMLCFTObligationAsPerCaymanIsland`, `FK_CountryId_AMLCFTJurisdictionLawCountry`)
- Nature of Business capture (`FK_LookupID_NatureOfBusiness`, `NatureOfBusinessExplaination`)
- Audited financial statement flag (`DoesFIAuditedFinancialStatement`)

### Luxembourg CRS Specifics

- Uses `CrsLuxGenerator` + `LuxCrsBodySplitter` (splits large messages)
- `CrsBodyCountryCodeProvider` maps body country codes for Lux format
- TIN strategy: always uses `#NTA001#` (Luxembourg TIN not applicable code)
- Depositor fields in `TIERegistration`: `Depositor`, `DepositorFirstname`, `DepositorLastName`, `DepositorEmail`, `DepositorPhoneNumber`, `Matricule` (Luxembourg CCSS registration number)
- Scheduled task: `CreateLuxCrsNotificationScheduleTask` (2024-03-05 data script)

### Singapore CRS Specifics

Singapore IRAS code logic in `SingaporeStrategy`:
```
IRAS101 = TIN Not Issued (for non-TIN countries: VG, KY, BS, BM, MO, AE)
IRAS103 = TIN Issued (Singapore issues its own TIN)
IRAS104 = TIN Other (country issues TIN but AH/CP doesn't have one)
```
Singapore TaxIDType is included as prefix in the `In` element.

Bulk import: `FatcaCrsRegistrationSingaporeImport` form, dedicated `IFatcaCrsRegistrationBulkImportSingaporeService`.

### Jersey CRS Specifics

Jersey TIN strategy: always outputs `NOTIN`.

### Austria CRS Specifics

Austria uses `InfoDaten` block (special OECD XML wrapping):
```csharp
// From CrsGenerator.GetInfoDaten()
if (ReportingToCountryCode == "AT")
{
    // FastnrFonTn = GetTaxIdentifierNoByCountry("AT", ...)
    // FastnrFi = same value
    // Vers = "01.00"
}
```

### Switzerland CRS Specifics

- Schema version forced to `"2.0"` (vs default "1.0")
- `SwitzerlandTaxID` field in `TIERegistration`
- Bulk import: `FatcaCrsRegistrationSwitzerlandImport`

### France CRS Specifics

- `CrsFranceGenerator` + `CrsFranceAccountReportGeneratorFactory` + `FranceSplitGenerator`
- Regulators tracked: `AMF` (Autorite des marches financiers) and `ACPR` (Banque De France)
- Large reports split by `FranceSplitGenerator`

### Spain CRS Specifics

- `CrsSpainGenerator` + `SpainSplitGenerator`
- **AEAT portal** requires **8 KB file limit per XML file**
- `SpainSplitGenerator` intersects account holder countries with `DueDiligenceReportableJurisdiction` to produce per-country AEAT files with **sequential presentation codes**
- Spain's own ISO code (`ES`) is explicitly excluded from receiving country codes in the split

### Complete Report Schema Codes (`TIEReportSchema`)

| Schema Code | Jurisdiction | Generator |
|---|---|---|
| `IRS` | United States (FATCA) | `IrsGenerator` |
| `LUX` | Luxembourg (FATCA) | `FatcaLuxGenerator` |
| `FATCAFINLAND` | Finland (FATCA) | `FinlandIrsGenerator` |
| `FATCAFRANCE` | France (FATCA) | `FranceIrsGenerator` + `FranceSplitGenerator` |
| `CRS` | Standard (all other) | `CrsGenerator` |
| `CRSLUX` | Luxembourg (CRS) | `CrsLuxGenerator` |
| `CRSCAYMAN` | Cayman Islands (CRS) | `CrsCaymanGenerator` |
| `CRSFINLAND` | Finland (CRS) | `CrsFinlandGenerator` |
| `CRSFRANCE` | France (CRS) | `CrsFranceGenerator` |
| `CRSSPAIN` | Spain (CRS) | `CrsSpainGenerator` (AEAT, 8 KB/file limit) |
| `HMRC` | United Kingdom | `HmrcGenerator` |

### Bulk Generation Status (`FatcaCrsBulkXmlGenerationStatusEnum`)

| Value | Description |
|---|---|
| `Initiated` | Request created, not yet started |
| `InProgress` | Processing underway |
| `Complete` | Successfully generated |
| `Error = 99` | Generation failed |

### Beneficial Owner / Controlling Person Filtering

- `BeneficialOwnerFilter` — filters controlling persons; UK falls back to Director/Controller if no investors present
- `SubstantialOwnerChecker` — determines FATCA substantial owner per entity/relationship
- `AccountFilter` — filters accounts for inclusion in CRS/FATCA reports
- `ControllingPersonFilter` — filters controlling persons for inclusion

### Implementation

HMRC uses its own generator `HmrcGenerator` (separate from IRS or OECD CRS path) and validation `HMRCValidator`.

```
InvestorServices.General/FatcaAndCrs/Generation/HMRC/HmrcGenerator.cs
InvestorServices.General/Components/TaxInformationExchange/DD/Reporting/HMRC/HolderTaxInfo.cs
InvestorServices.General/Components/TaxInformationExchange/Validation/Reporting/HMRCValidator/HolderTaxInfoValidator.cs
```

Test coverage:
```
ISC.TaxInformationExchange.Test/HMRC/HolderTaxInfoTests.cs
ISC.TaxInformationExchange.Test/HMRC/UkFatcaSubmissionFiReportTests.cs
```

### Registration

`HmrcAeoiPortalID` stored in `TIERegistration`. IDR manages HMRC AEOI portal registration separately from IRS FATCA (`GIIN`).

### HMRC Report (SSRS)

`RPT_HMRCFatcaReportProcedure.sql` — custom stored procedure for HMRC reporting output.

---

## 7. Withholding — Form 1042 / 1042-S

### Purpose

- **Form 1042** — Annual Withholding Tax Return for US source income paid to foreign persons (fund → IRS)
- **Form 1042-S** — Individual statement per foreign payee (fund → investor)

Both use ASCII fixed-width format stored in `TIEWithholdingReport.ReportASCII`.

### DB Tables

```sql
TIEWithholdingReport
  ├── FK_TIEPeriodID
  ├── FK_TIEReportTypeID   (Form1042 or Form1042s)
  ├── FK_EntityID
  ├── FK_TIEReportStatusID
  ├── FK_TIEReportMessageTypeID
  ├── ReportASCII           -- the actual report content
  ├── IsValidated / ValidationErrors
  ├── SubmissionReference / DateSubmitted
  ├── ResubmissionReference / DateResubmitted
  └── ResubmissionComments

TIEEntityInvestigation1042
  ├── FK_TIEInvestigation1042Id
  ├── FK_TIEPeriodId
  ├── FK_EntityID_Requestor
  ├── FK_DocumentId           -- attached document
  ├── EnvelopeId / EnvelopeStatus   -- DocuSign envelope
  ├── IsInitialApproved / DateInitialApproved / FK_UserID_InitialApprovedBy
  └── IsSecondaryApproved / DateSecondaryApproved / FK_UserID_SecondaryApprovedBy
```

### 1042-S Dashboard Export Types

```csharp
// Dashboard1042sExportType enum
Interest  = 1   // "Interest Income"
Dividends       // "Dividend Income"
Other           // "All Income"
```

### Treaty Rate Integration

Withholding rates come from the `Treaty` table (see §14). Each payment type can be reduced from the default 30% US withholding rate based on the investor's country treaty.

### Approval Workflow for 1042

`TIEEntityInvestigation1042` has a **two-stage DocuSign approval**:
1. **Initial Approval** — first approver signs via DocuSign
2. **Secondary Approval** — second approver signs via DocuSign

Envelope status tracked via `EnvelopeId` + `EnvelopeStatus`.

### W-Form Dashboard

`FatcaCrsWithholdingDashboardView.sql` joins W-Form data with withholding applicable entities. The W-Form Dashboard (`FatcaCrsWithholdingDashboardController`) supports:
- Filter by period, entity, W-Form type
- Filter by beneficiary (W-series beneficiaries view)
- Grouped status views
- Expiration filter

---

## 8. W-Forms (W-Series)

### Types

```csharp
// WFormType enum
W9     = 1  // US persons — certify US tax status and TIN
W8BEN  = 2  // Non-US individuals — claim treaty benefits
W8BENE = 3  // Non-US entities — most common for fund investors
W8ECI  = 4  // Entities with ECI (Effectively Connected Income)
W8EXP  = 5  // Foreign governments / international organisations / tax-exempt orgs
W8IMY  = 6  // Intermediaries / flow-through entities (partnerships, trusts)
```

### DB Structure

```sql
WFormType
  ├── FK_LookupId         → Lookup (maps to WFormType enum value)
  ├── FK_DocuSignTemplateId → DocuSign template to use for this form
  ├── Tag                 → string identifier
  └── IsActive

WFormQuestion
  ├── FK_WFormTypeId      → which W-Form this question belongs to
  ├── FK_LookupTypeId     → optional lookup for dropdown answers
  ├── FieldType           → "text", "checkbox", "date", etc.
  ├── QuestionText
  ├── QuestionTag         → maps to DocuSign field tag
  └── IsActive

WFormAnswer
  — Stores answers per entity per W-Form instance

WFormChapter3StatusMapping
  — Maps Chapter 3 status lookup values to Tags (used in IRS XML)

WFormChapter4StatusMapping
  — Maps Chapter 4 status lookup values to Tags (used in IRS XML)
```

### W-Form DocuSign Integration

Each `WFormType` row points to a `DocuSignTemplate`. When a W-Form is requested:
1. IDR creates a DocuSign envelope using the template
2. Questions are mapped to DocuSign field tags via `WFormQuestion.QuestionTag`
3. Investor/entity signs electronically
4. Signed document returned, status tracked in `DocuSignDocumentStatus`
5. `TF065TaxFormUploadedGenerated` task fires on upload
6. `TF067TaxFormExpiry` task tracks expiry (W-8 forms expire every 3 years)
7. `TF069TaxFormDeletionTask` handles form deletion
8. `TF070TaskTaxForm` handles general W-Form workflow tasks

### W-Form Identifier Constants

```csharp
// WFormIdentifiers enum
WSeriesEvidenceRequirementId = 59   // evidence requirement ID for W-series forms
LookupId = 1082                      // lookup ID for W-Form types
```

### W-Form Dashboard

`FatcaCrsWformDashboardController` (Admin) exposes:
- Paginated W-Form dashboard
- Grouped status data
- Grouped W-Form type data
- Expiration filter data
- W-Series beneficiaries dashboard

### W-Form Tasks (Processor)

```
TF065  — Tax Form Uploaded/Generated
TF067  — Tax Form Expiry (triggered on expiry date approach)
TF069  — Tax Form Deletion
TF070  — General Tax Form workflow task
```

---

## 9. Classification Workflow

### Purpose

Every entity that is subject to FATCA/CRS must be classified under:
- **US Classification** (Chapter 3: income type; Chapter 4: FATCA entity category)
- **CRS Classification** (OECD entity type: Financial Institution, Active NFE, Passive NFE, etc.)

### Three-Level Approval

```
TIEClassification
├── IsClassifiedAlready     — initial questionnaire completed by user/analyst
├── IsIDRApproved           — IDR analyst has reviewed and approved
│   └── FK_UserID_IDRApprovedBy / DateIDRApproved
└── IsKpmgApproved          — KPMG (external reviewer) has approved (optional)
    ├── IsKpmgApprovalRequested / DateKpmgApprovalRequested
    └── FK_UserID_KpmgApprovedBy / DateKpmgApproved
```

### Classification References

- `FK_ClassificationID_US` → `Classification` lookup (IRS Chapter 3/4 type)
- `FK_ClassificationID_CRS` → `Classification` lookup (OECD CRS type)
- `FK_Chapter3StatusID` → special Chapter 3 status (SingleMemberLLC, DisregardedEntity)
- `FK_TaxClassificationID` → additional tax classification
- `FK_TIEClassificationQuestionSetID` → which questionnaire version was used

### Classification Group Type

```csharp
// ClassificationGroupTypeEnum
ActiveNonFinancialEntity = 1  // "Active Non-Financial Entity"
// (Passive NFE and Financial Institution are lookup values, not enum)
```

### Classification History

`GetEntityFatcaAndCrsClassificationHistoryFunction` and `GetEntityFatcaAndCrsHistoryFunction` provide full audit history of all classification changes.

### Classification Reports (SSRS)

- `FATCA And CRS Classification.rdl` — full classification report per entity
- `FATCA and CRS Classification Summary.rdl` — summary across entities
- `FATCA and CRS Classification Summary Cover.rdl` — cover page
- `Classification Registration Summary.rdl` — registration + classification combined
- `RPT_GetFATCACRSClassificationProcedure.sql` — stored procedure for report
- `RPT_GetFATCACRSControlDataProcedure.sql` — control data report

---

## 10. Registration Module

### `TIERegistration` — Complete Field List

| Field | Purpose |
|---|---|
| `FK_EntityID` | Entity being registered |
| `FK_UserID_ResponsibleOfficer` | IRS Responsible Officer designation |
| `IsIpesResponsibleOfficer` | Whether IPES (IDR) is the Responsible Officer |
| `IsRegistered` | FATCA registration complete |
| `IsRequested` | Registration requested (not yet confirmed) |
| `IsIpesRegistered` | IPES registered on behalf of entity |
| `GIIN` | IRS Global Intermediary Identification Number |
| `IsWithholdingReportable` | Whether entity has 1042/1042-S obligations |
| `IRSFATCALogin` / `IRSFATCAPassword` | IRS FATCA portal credentials |
| `FK_CountryID_ReportingTo` | Primary reporting jurisdiction |
| `HmrcAeoiPortalID` | HMRC AEOI portal identifier |
| `LuxembourgCCSSNumber` | Luxembourg CCSS registration |
| `CaymanFiNumber` | Cayman FI number for CRS |
| `IsRegisteredWithIgor` | IGOR portal registration status |
| `BeginReportingFromYear` | First reportable year |
| `IsManuallyExcluded` | Admin can exclude entity from TIE |
| `SwitzerlandTaxID` | Switzerland tax identifier |
| `IsRegistrationComplete` | All fields validated complete |
| `FatcaContactName/Email/Phone` | FATCA point of contact |
| `IsIdrTheIrsPointOfContact` | Whether IDR acts as IRS contact |
| `Depositor` / `DepositorFirstname` / ... | Luxembourg depositor details |
| `Matricule` | Luxembourg CCSS matricule |
| `FK_LookupID_SingaporeTaxIDType` | Singapore tax ID type (IRAS classification) |
| `FK_LookupID_SingaporeSubmitterTaxIDType` | Singapore submitter type |
| `SingaporeSubmitterName` / `SingaporeSubmitterTaxId` | Singapore submitter identity |
| `FK_LookupID_LateDataID` | Late data submission reason |
| `FK_CurrencyID` | Reporting currency |
| `FK_LookupID_NatureOfBusiness` | Cayman nature-of-business lookup |
| `NatureOfBusinessExplaination` | Free-text explanation |
| `DoesFIAuditedFinancialStatement` | Cayman: audited financial statements |
| `WhoCarriesOutAMLKYCObligation` | Cayman AML/KYC delegation |
| `AMLKYCObligationEntityName` | Name of AML/KYC delegate |
| `FK_CountryId_AMLKYCObligationEntityLocation` | Location of AML/KYC delegate |
| `IsAMLCFTObligationAsPerCaymanIsland` | Cayman AMLCFT flag |
| `FK_CountryId_AMLCFTJurisdictionLawCountry` | Cayman jurisdiction law |
| `POCFirstName/LastName/Email/Phone` | General point of contact |

### Bulk Registration Imports (by Jurisdiction)

IDR supports jurisdiction-specific bulk Excel imports for registration data:
```
FatcaCrsRegistrationImport             — standard (all)
FatcaCrsRegistrationCaymanImport       — Cayman-specific fields
FatcaCrsRegistrationGuernseyImport     — Guernsey-specific
FatcaCrsRegistrationLuxembourgImport   — Luxembourg-specific (CCSS, Depositor, Matricule)
FatcaCrsRegistrationSingaporeImport    — Singapore IRAS fields
FatcaCrsRegistrationSwitzerlandImport  — Switzerland Tax ID
FatcaCrsRegistrationUKImport           — HMRC AEOI Portal ID
```

Corresponding services and controllers for each bulk import type.

---

## 11. Investigation (Financial Data)

### What Is an Investigation?

The "investigation" is the annual data collection of financial figures per investor-fund relationship per reporting period. This is the source data for all TIE reports.

### `TIEInvestigation` Fields

| Field | CRS/FATCA Mapping |
|---|---|
| `FK_RelationshipID` | Investor ↔ Fund relationship |
| `FK_TIEPeriodID` | Reporting period (year) |
| `AccountBalance` | Account Balance as of 31-Dec |
| `Dividend` | Dividend income during period |
| `Interest` | Interest income during period |
| `ProceedsAndRedemption` | Gross proceeds (shares sold/redeemed) |
| `OtherPayment` | Other income / payments |
| `FK_CurrencyID` | Currency of figures |
| `AccountNumber` | Account/account identifier |
| `ReportableJurisdictions` | JSON/list of CRS jurisdictions for this account |

### Investigation Import

Multiple import paths exist:
```
FatcaCrsInvestigationImport          — standard line import
FatcaCrsInvestigationImportMaster    — master entity import
FatcaCrsInvestigationBulkImportService — bulk via Excel
FatcaCrsInvestigationExcelImportService — Excel-specific
```

Excel templates:
- `FatcaCrsDashboardInvestigationTemplate.xlsx`
- `FatcaCrsDashboardInvestigationBulkTemplate.xlsx`
- `FATCA_and_CRS_Investigation_Export.xlsx`

### Previous Year Data Copy

`PreviousYearFatcaCrsFinancialDataCopyService` and `PreviousYearFatcaCrsCacheDataCopyService` handle year-on-year data rollover — so the prior year's account balances can be carried forward as a starting point.

### Investigation Comparison

`EntityAndInvestorDataComparer`, `InvestigationChangeType`, `InvestigationComparisonResult`, `InvestigationDataChange` — detect and record year-on-year changes in financial data for the investigation changes report.

### 1042 Investigation

`TIEEntityInvestigation1042` extends investigation for withholding-specific data:
- Linked to a DocuSign approval flow (two-stage)
- Linked to a document (1042 form attachment)
- Period-scoped

---

## 12. Reportable Jurisdictions

### Purpose

Each entity that is a fund must declare which jurisdictions it reports to for CRS. Investors in that fund are then reported to their country of residence.

### `DueDiligenceReportableJurisdiction` Workflow

```
User self-selects jurisdictions (IsUserSelected = 1)
         ↓
Admin can also force-add (IsAdminSelected = 1)
         ↓
IsAgreed — user and admin agree on final list
         ↓
IsApproved — IDR analyst approves the list
         ↓
IsConfirmed — final confirmation
```

### Trigger Jurisdictions

`DueDiligenceTriggerReportableJurisdiction` — jurisdictions that auto-trigger reporting based on entity characteristics (e.g. entity type, residency).

### Functions

- `HasCrsReportableJurisdictionFunction` — does entity have any CRS reportable jurisdictions?
- `IsCrsPassiveAccountHolderReportable` — is this passive account holder reportable?

### Country Excluded Jurisdictions

`CountryExcludedJurisdiction` table (referenced in prior session exploration) lists countries explicitly excluded from CRS reporting for certain fund structures.

### SSRS Reports

- `Reportable Jurisdictions.rdl`
- `Reportable Jurisdictions Tax Residency and Approval.rdl`
- `Account Holder Reporting Status.rdl`

---

## 13. Tax Residences

### `DueDiligenceProfileTaxResidence`

Per-entity/profile tax residency record capturing:
- Country of tax residence
- TIN (Tax Identification Number) for that country
- Reason if TIN not provided

### DB Functions

- `GetAllProfileTaxResidencesByEntityFunction` — all residences for an entity
- `GetAllProfileTaxResidencesByProfile` — by profile
- `GetDueDiligenceProfileTaxResidenceByProfileIds` — bulk by profile IDs
- `HasCddTaxResidenceFunction` — does profile have any CDD tax residence?
- `HasUsqTaxStatusFunction` — does profile have USQ tax status?
- `F_IsCddTaxResidence` — reports schema function

### Views

- `EntityTaxResidenceView` — all tax residences per entity
- `EntityTaxResidencePivotView` — pivoted (one row per entity, columns per country)
- `EntityReportableTaxResidenceView` — only CRS-reportable residences
- `EntityReportableTaxResidencePivotView` — pivoted reportable residences

### Tax Residence in KYC Questionnaire

Tax residences are captured as part of the KYC questionnaire flow:
- `DueDiligence/TaxResidence` Angular component in the investor-facing questionnaire
- `OnboardingImportCddTaxResidence` for bulk onboarding import
- `RPT_UniversalQuestionnaire_03_TaxStatus_TaxResidences.sql` — USQ section 3 report
- `RPT_UniversalQuestionnaire_03_TaxStatus_GeneralTaxProvisions.sql`

### Missing Tax Code Report

`EntitiesWithNoOrInvalidTaxFormView` — view identifying entities that are missing or have invalid W-Forms (used for remediation workflows).

`MissingTaxCodeLookupItem` — DTO for missing TIN code lookup.

### SSRS Reports

- `Tax Forms Report.rdl`
- `Tax Portal Registration.rdl`
- `HMRC Non-resident UTR.rdl` (UK non-resident Unique Taxpayer Reference)

---

## 14. Treaty Rates

### `Treaty` Table — Complete Field List

Per country, reduced withholding rates:

| Field | Description |
|---|---|
| `FK_CountryId` | Treaty country |
| `Interest` | Standard interest rate |
| `PortfolioInterest` | Portfolio interest rate (often 0% under US treaty) |
| `BankInterest` | Bank deposit interest rate |
| `ContingentNonPortfolioInterest` | Contingent interest (non-portfolio) |
| `Dividends` | General dividend rate |
| `QualifyingDirectDividendRate` | Qualifying direct dividend (inter-company) |
| `PensionAndAnnuities` | Pension/annuity payment rate |
| `OtherRoyalties` | General royalty rate |
| `Patents` | Patent royalty rate |
| `RoyaltiesFilmAndTV` | Film/TV royalty rate |
| `RoyaltiesCopyrights` | Copyright royalty rate |

### Usage

Treaty rates are applied when:
1. An investor provides a valid W-Form (W-8BEN / W-8BEN-E) claiming treaty benefits
2. The investor's country of residence has a treaty with the US
3. The specific income type falls under the treaty article

Default US withholding rate (no treaty): **30%**. Most income types can be reduced to 0–15% under treaty.

---

## 15. TIN Strategy by Country

### Source: `TinStrategyFactory` + Strategy Classes

The `TinStrategyFactory` selects the appropriate TIN strategy for CRS report generation:

| Country | Strategy | TIN Value Logic |
|---|---|---|
| `SG` — Singapore | `SingaporeStrategy` | IRAS101 (not issued), IRAS103 (issued), IRAS104 (other) — depends on account holder's country |
| `LU` — Luxembourg | `LuxembourgStrategy` | Always `#NTA001#` |
| `JE` — Jersey | `JerseyStrategy` | Always `NOTIN` |
| All others | `DefaultStrategy` | Standard TIN from entity data |

### Non-Default TIN Countries (CRS Base Generator)

These countries are flagged in `CrsGenerator.nonDefaultTinCountries` as not requiring standard TIN inclusion:
- `KY` — Cayman Islands
- `VG` — British Virgin Islands
- `BS` — Bahamas
- `GG` — Guernsey

### IRS TIN Generation

`UsTinGenerator` and `IrsTinGenerator` handle US-side TIN formatting for FATCA XML.

`OrganisationTinResult` wraps the TIN result for organisation entities.

---

## 16. Bulk XML Generation & IGOR Submission

### `FatcaCrsBulkXmlGenerationConfiguration` Enum

| Value | Description | What It Does |
|---|---|---|
| `NilReturn = 1` | Nil Return | Generate XML with zero values (no reportable accounts) |
| `SubstantiveReturn = 2` | Substantive Return | Full report with actual financial data |
| `SubmitToIgor = 3` | Submit To Igor | Generate + upload to IGOR portal |
| `BulkDownload = 4` | Bulk Download | Generate and package as ZIP for download |
| `RollOver = 5` | Roll Over | Copy prior year data to new period |

### `FatcaCrsBulkXmlGenerationRequest` Table

```sql
FatcaCrsBulkXmlGenerationRequestID   -- PK
FatcaCrsBulkXmlGenerationRequestGUID -- GUID for async tracking
BulkXmlGenerationConfiguration       -- Which config (1-5)
PeriodId                              -- Reporting period
EntityIds                             -- JSON array of entity IDs to process
FK_RequestedBy                        -- User who triggered
GenerationRequestStatus               -- Queued / Processing / Complete / Failed
DateCreated
GenerationRequestStartTime
LastUpdatedDateTime
ZipPackageAzureBlobID                 -- Azure Blob storage GUID for the ZIP file
ErrorMessage
BulkDownloadRequestIds                -- Reference to individual download requests
```

### Processing Components (`ISC.BulkTIEProcessing`)

```
BulkXmlGenerationRequestProcessor   -- main orchestrator
├── BulkNilReturnProcessor          -- handles NilReturn config
├── BulkSubReturnProcessor          -- handles SubstantiveReturn config
├── IgorBulkProcessor               -- base Igor processor
│   ├── IgorFatcaBulkProcessor      -- FATCA → IGOR
│   └── IgorCrsBulkProcessor        -- CRS → IGOR
├── BulkDownloadProcessor           -- ZIPs XMLs for download
│   └── BulkXmlDownloadOutputPathGenerator
├── RollOverProcessor               -- copies prior period data
└── ManualRequestProcessor          -- for manual/ad-hoc runs
```

### Output Structure

```
XmlInLeadProfileFolderProcessor     -- organises XMLs into per-lead-profile folder
XmlChildProfileFolderProcessor      -- organises XMLs for child profiles
```

### IGOR Submission

IGOR is the IRS/external TIE submission portal. IDR integrates via:
- `TIEReportIGORSubmission` table tracks submission per report per IGOR run
- `IsRegisteredWithIgor` flag on `TIERegistration` enables/disables
- `IGOR_IPES_ORGANISATION_ID = "234"` hardcoded in `CrsGenerator.GetMessageSpec()`
- Submission reference returned by IGOR stored in `TIEReport.SubmissionReference`

### Report Splits

Some jurisdictions generate reports too large for a single XML file:
- France: `FranceSplitGenerator`
- Spain: `SpainSplitGenerator`
- Luxembourg: `LuxCrsBodySplitter`
- `CrsActiveAccountSplitReportGenerator` — split active account reports
- `TIEReportSplit` table — stores individual split parts
- `TieReportSplitFormats` DTO wraps split output

---

## 17. Report Lifecycle (Generate → Submit → Resubmit)

### `TIEReport` Lifecycle Fields

```
ReportXML / ReportJSON         → generated content (also stored as Documents)
FK_TIEReportStatusID           → status tracking
IsValidated / ValidationErrors → XML schema validation result

-- Test submission
SubmissionTestReference
DateTestSubmitted
SubmissionTestComments

-- Live submission
SubmissionReference
DateSubmitted
HasBeenSubmitted

-- Resubmission (corrections)
ResubmissionReference
DateResubmitted
ResubmissionComments

IsExternal                     → externally generated report (not by IDR)
```

### Status Flow

```
Draft → Generated → Validated → TestSubmitted → Submitted → (Resubmitted)
                                                           ↘ Corrected
```

### Report Schema Versioning

`TIEReportSchema` table allows multiple schema versions per report type. Data update script `20240409_Insert_TIEReportSchemaCRS.sql` shows how new schema versions are inserted.

### Report Validator

`TieReportValidator.cs` — validates generated XML against the appropriate schema before submission.

### `FatcaCrsReportDocument` Table

```sql
FatcaCrsReportDocument
├── FK_TIEReportID
├── FK_DocumentID
└── DocumentType   (XML, JSON, etc.)
```
Separate audit table (`audit_FatcaCrsReportDocument`) maintains full history.

---

## 18. Scheduled Tasks & Cache Jobs

### Cache Tables

```
FatcaCrsAccountReportableCache              — is account CRS/FATCA reportable? (cached result)
FatcaCrsAccountSummaryCache                 — summary figures per entity per period
FatcaCrsOwnerControllerCache                — owner/controller cache for complex entity structures
FatcaCrsLeadEntityOwnerAndControllerPercentageCache — ownership % for lead entities
```

### Cache Stored Procedures

```sql
CacheEntityFatcaCrsDataProcedure              — full entity cache refresh
CacheFatcaCrsAccountReportableProcedure       — refresh reportability cache
CacheFatcaCrsAccountSummaryProcedure          — refresh summary figures
CacheFatcaCrsOwnerAndControllerProcedure      — refresh owner/controller cache
CacheFatcaCrsLeadEntityOwnerAndControllerPercentageProcedure — refresh % cache
```

### Scheduled Task Classes

```
FatcaCrsReportableAccountsCacheTaskExecutor — runs cache refresh on schedule
ScheduledTaskProcessor                      — general task runner
```

Data scripts show monthly cache refresh tasks (`FatcaCrsMonthlyReportUpdateTask`).

### Task Functions

```sql
HasOutstandingFatcaCrsContactTaskFunction     — outstanding contact tasks?
HasOutstandingFatcaRegistrationTaskFunction   — outstanding registration tasks?
HasOutstandingUsFatcaClassificationTaskFunction — US FATCA classification pending?
HasOutstandingCrsClassificationTaskFunction   — CRS classification pending?
HasMissingFatcaCrsContactDetailsFunction      — contact info missing?
HasReportableUsFatcaClassificationFunction    — US FATCA reportable?
HasReportableCrsClassificationFunction        — CRS reportable?
```

---

## 19. SSRS Reports

### Tax-Specific Reports

| Report | Type | Purpose |
|---|---|---|
| `FATCA And CRS Classification.rdl` | SSRS | Per-entity FATCA+CRS classification detail |
| `FATCA and CRS Classification Summary.rdl` | SSRS | Across-portfolio classification status |
| `FATCA and CRS Classification Summary Cover.rdl` | SSRS | Cover page for classification pack |
| `Fatca Submission Timeline.rdl` | SSRS | Timeline of all FATCA submissions |
| `FatcaDashboard.rdl` | Custom | FATCA dashboard overview |
| `Fatca Crs Status.rdl` | Custom | FATCA/CRS status per entity |
| `Fatca Crs Contact.rdl` | Custom | FATCA contact details |
| `Fatca Lst Year Data.rdl` | Custom | Last year data comparison |
| `FATCA Data.rdl` | Custom | Raw FATCA data extract |
| `FATCA W-Form.rdl` | Custom | W-Form status report |
| `FATCA Submission - Profile.rdl` | Custom | Submission details per profile |
| `Tax Forms Report.rdl` | Custom | All tax forms per entity |
| `Tax Portal Registration.rdl` | Custom | TIE registration status |
| `Reportable Jurisdictions.rdl` | SSRS | CRS reportable jurisdiction list |
| `Reportable Jurisdictions Tax Residency and Approval.rdl` | Custom | Combined report |
| `Submission Summary.rdl` | SSRS | Overall submission status |
| `Submission Summary 1042.rdl` | SSRS | 1042 submission summary |
| `Submission Summary 1042S.rdl` | SSRS | 1042-S submission summary |
| `TIE Investigation Summary.rdl` | SSRS | Investigation data by entity |
| `TIE Investigation Appendix.rdl` | SSRS | Investigation appendix |
| `TIE Investigation Appendix 3.rdl` | SSRS | Appendix 3 (Cayman-specific) |
| `TIE Investigation Summary Cover.rdl` | SSRS | Cover for investigation pack |
| `Investigation Summary Year on Year Comparison Report.rdl` | SSRS | YoY delta report |
| `Investigation Changes.rdl` | Custom | What changed in financial data this year |
| `Withholding Statement.rdl` | Custom | 1042-S / withholding statement |
| `HMRC Non-resident UTR.rdl` | Custom | UK UTR for non-residents |
| `Account Holder Reporting Status.rdl` | SSRS | CRS account holder status |
| `KPMG Cayman - Indicative Risk Factors.rdl` | Custom | Cayman risk for KPMG review |
| `Profile Classification and GIIN.rdl` | Custom | Entity classification + GIIN export |

### Custom Stored Procedures for Reports

```sql
RPT_FatcaCrsStatusProcedure
RPT_FatcaCrsStatusAggregateProcedure
RPT_GetFatcaDashboardDataProcedure
RPT_GetFatcaLoginDataProcedure
RPT_GetFATCACRSClassificationProcedure
RPT_GetFATCACRSControlDataProcedure
RPT_HMRCFatcaReportProcedure
RPT_FATCA_and_CRS_Classification_Questions
RPT_FATCA_and_CRS_Classification_Results
RPT_FATCA_and_CRS_Classification_Summary
RPT_TaxForms
RPT_GetTaxPortalRegistration
RPT_UniversalQuestionnaire_03_TaxStatus_GeneralTaxProvisions
RPT_UniversalQuestionnaire_03_TaxStatus_TaxResidences
GetFatcaSubmissionTimelineProcedure
GetFatcaCrsDashboardFinancialDataProcedure
```

---

## 20. Controller Surface

### Entity-Level Controllers

```
FatcaCrsBulkXmlGenerationController   GET/POST bulk XML generation requests
FatcaCrsClassificationController      GET/PUT/POST classification + history
FatcaCrsContactController             GET/PUT FATCA contact details
FatcaCrsDashboardChartController      GET chart data for dashboard
FatcaCrsDashboardFinancialDataController  GET financial data per entity per period
FatcaCrsDocumentController            GET/DELETE attached documents
FatcaCrsInvestigationController       GET/POST/PUT/DELETE investigation data
FatcaCrsMetricsController             GET submission references / metrics
FatcaCrsRegistrationController        GET/PUT registration data
FatcaCrsReportingController           GET/POST/PUT reporting (generate/submit/resubmit)
```

### Admin Controllers

```
FatcaCrsWformDashboardController        W-Form dashboard views + filter
FatcaCrsWithholdingDashboardController  Withholding dashboard views + filter
```

### Tax Status Controllers

```
TaxStatusController   (Usq module)  GET/PUT universal questionnaire tax status
```

---

## 21. Key Business Rules

### FATCA Reportability

1. Entity must have `TIERegistration.IsRegistered = true`
2. Entity must not be `IsManuallyExcluded = true`
3. Entity must have `BeginReportingFromYear ≤ current reporting year`
4. `HasReportableUsFatcaClassificationFunction` must return true
5. Account must be `IsUSApplicable = true` in `GetTieDataSetAccountFunction`

### CRS Reportability

1. Entity must have CRS reportable jurisdictions (`HasCrsReportableJurisdictionFunction`)
2. Each account holder's country must be in the entity's `DueDiligenceReportableJurisdiction` (agreed + approved)
3. Account must be `IsCRSApplicable = true`
4. `IsCrsPassiveAccountHolderReportable` — passive NFE account holders require controlling persons (beneficial owners) to be reportable

### W-Form Rules

1. W-8 forms expire every 3 years (tasks TF067 / TF069 manage this)
2. W-9 required for all US persons (individual or entity)
3. W-8BEN-E required for non-US entities claiming treaty benefits
4. W-8IMY required for intermediaries (partnerships, trusts, nominees)
5. Missing/invalid W-Forms tracked in `EntitiesWithNoOrInvalidTaxFormView`

### Classification Rules

1. All entities must have both US (Chapter 4) and CRS classification before reporting
2. IDR approval is mandatory (`IsIDRApproved` must be true before report generation)
3. KPMG approval is optional per fund configuration
4. Chapter 3 status (`SingleMemberLLC` / `DisregardedEntity`) affects how entity's TIN is presented

### Withholding Rules

1. Default withholding rate: **30%** (US source income to foreign persons)
2. Reduced rate if valid W-Form + Treaty row exists for investor's country
3. `IsWithholdingReportable` on `TIERegistration` must be true for 1042 generation
4. 1042 requires two-stage DocuSign approval before submission

### TIN Rules by Country (CRS)

| Scenario | Rule |
|---|---|
| Reporting to Luxembourg | Always `#NTA001#` |
| Reporting to Jersey | Always `NOTIN` |
| Reporting to Singapore | IRAS codes (101/103/104) based on account holder's country |
| Account holder in KY/VG/BS/GG | Use non-default TIN handling (these countries don't issue TINs) |
| All others | Standard TIN from entity data |

### Report Submission Rules

1. Report must pass XML schema validation (`IsValidated = true`) before submission
2. Test submission (`SubmissionTestReference`) should precede live submission
3. Resubmission requires new reference and comments
4. IGOR portal submission only available if `IsRegisteredWithIgor = true`

---

## 22. Database Schema Map

```
dbo schema (primary):
├── TIERegistration
├── TIEPeriod
├── TIEReport
├── TIEReportType (ref)
├── TIEReportStatus (ref)
├── TIEReportSchema
├── TIEReportMessageType (ref)
├── TIEReportSplit
├── TIEReportIGORSubmission
├── TIEReportSubmissionHistory
├── TIEInvestigation
├── TIEEntityInvestigation
├── TIEInvestigationDraft
├── TIEEntityInvestigation1042
├── TIEEntityInvestigation1042s
├── TIEInvestigation1042
├── TIEClassification
├── TIEClassificationQuestion
├── TIEClassificationQuestionSet
├── TIEClassificationAnswer
├── TIEClassificationDocument
├── TIEContact
├── TIEContactCategory
├── TIEDocument
├── TIEWithholdingReport
├── TIEWithholdingReportSubmissionHistory
├── TIEEntitySubmissionCompliance
├── FatcaCrsBulkXmlGenerationRequest
├── FatcaCrsReportDocument
├── FatcaCrsAccountReportableCache
├── FatcaCrsAccountSummaryCache
├── FatcaCrsOwnerControllerCache
├── FatcaCrsLeadEntityOwnerAndControllerPercentageCache
├── CaymanCrsComplianceConfig
├── DueDiligenceReportableJurisdiction
├── DueDiligenceTriggerReportableJurisdiction
├── DueDiligenceProfileTaxResidence
├── Treaty
├── WFormType
├── WFormQuestion
├── WFormAnswer
├── WFormChapter3StatusMapping
├── WFormChapter4StatusMapping
└── EntitiesWithNoOrInvalidTaxFormView (view)

audit schema:
├── FatcaCrsBulkXmlGenerationRequest (audit)
├── FatcaCrsReportDocument (audit)
└── DueDiligenceProfileTaxResidence (audit)

system schema:
└── FatcaScheduledTask

reports schema:
└── (25+ stored procedures for SSRS)
```

---

## 23. Architecture Notes for Rebuild

### What Tax Compliance Needs as a Bounded Context

If extracting Tax Compliance to a dedicated microservice or module, it needs:

**Inbound data (owned by other contexts):**
- Entity + EntityType (from KYC/Entity context)
- Relationship (Investor ↔ Fund, from Subscription/Onboarding context)
- Country / Lookup reference data (from Reference Data context)
- Currency (from Reference Data)
- DocuSign integration (from Document/Evidence context)
- User permissions (from Auth context)

**Owned by Tax Compliance:**
- All TIE* tables
- FatcaCrs* tables
- Treaty table
- WForm* tables
- DueDiligenceReportableJurisdiction
- DueDiligenceProfileTaxResidence
- CaymanCrsComplianceConfig

**Outbound events Tax Compliance publishes:**
- `EntityClassificationApproved` (used by reporting + due diligence)
- `WFormExpired` (triggers task creation)
- `WFormCompleted` (marks evidence requirement satisfied)
- `TIEReportSubmitted` (audit / compliance tracking)

### Recommended Module Split

```
S1.Module.TaxInformationExchange
├── Registration/           (GIIN, HMRC, Lux, Cayman, Singapore, Swiss IDs)
├── Classification/         (US Chapter 3/4, CRS type, KPMG approval workflow)
├── Investigation/          (financial data per period per relationship)
├── ReportableJurisdictions/ (per-entity CRS jurisdiction elections)
├── TaxResidence/           (per-profile residency + TIN)
├── WForms/                 (W-9/W-8 DocuSign workflow)
├── Reporting/              (generate + validate + submit TIE XML)
├── Withholding/            (1042/1042-S generate + approve + submit)
└── BulkProcessing/         (batch generation, IGOR submission, rollover)
```

### Generation Library

`ISC.TaxInformationExchange` (the generation library used by `InvestorServices.General`) should be extracted as a **nuget package** or standalone library — it already has clear boundaries and its own test project.

### Complexity Hotspots (Rebuild Risk)

| Area | Risk | Reason |
|---|---|---|
| CRS jurisdiction-specific generators | HIGH | 8+ generators with unique XML schema requirements |
| IGOR portal integration | HIGH | External dependency, documented only in code |
| W-Form DocuSign flow | MEDIUM | Complex tag mapping, 3-year expiry tracking |
| Treaty rate application | MEDIUM | Applied at report time; logic in generation code |
| Cayman CRS compliance | HIGH | Additional AML/KYC layer beyond standard CRS |
| 1042 two-stage DocuSign approval | MEDIUM | Custom approval workflow |
| Year-on-year cache + rollover | MEDIUM | Data consistency across periods |
| Classification approval chain | MEDIUM | Three actors (user, IDR, KPMG), each optional |

---

*Document generated from IDR codebase analysis — `C:\Code\IDR`*
*Last updated: 2025*
