# Questionnaires, Sections, and Standards — Counts and Real Usage

**Question asked:** "How many questionnaires are there and sections?" — followed by "Standards whats the links and can you give me queries to run."

**Scope:** a factual inventory of `S1.Module.Kyc`'s questionnaire data — how many questionnaires and sections exist, how `QuestionnaireStandard` links to `Questionnaire`/`ProfileType`/`Profile`, and (once queries were run against a live database) how that seeded structure is actually being used in production. The seed-data facts are grounded in `src/S1.Module.Kyc/docs/flows/questionnaires.md` and the table DDL under `database/TemplateApi.DatabaseConfiguration/Kyc/Tables/`; the usage facts are grounded in query results run by the user against a live environment.

---

## 1. The link chain

A profile is never joined directly to a questionnaire — `Visa` is the join, and a profile can hold several:

```
Kyc.QuestionnaireStandard (lookup, 6 rows: 0=None, 1=Global, 2=US, 3=Investment KYC, 4=Hellman and Friedman, 5=CBRE)
        ↑ Questionnaire.QuestionnaireStandardRef
Kyc.Questionnaire (27 rows) ── ProfileTypeRef → Kyc.ProfileType (13 rows)
        ↑ Visa.QuestionnaireIdRef
Kyc.Visa ── ProfileIdRef → Kyc.Profile        (a profile can hold several visas → several questionnaires, "merged")
```

Two things worth knowing about this chain before querying it:

- **`Domain/Enums/QuestionnaireStandard.cs` only declares 4 values** (`Global`, `US`, `InvestmentKYC`, `HellmanFriedman`) — the DB lookup has 6 (`None=0` and `CBRE=5` are missing from the C# enum). Reading a CBRE questionnaire's `QuestionnaireStandardRef` through C# yields an out-of-range enum value, and serializes as the raw number `5` rather than a name, because `JsonStringEnumConverter` can't map it.
- **Only `QuestionnaireStandard.Global` is ever assigned automatically.** `VisaService.AssignVisaForProfileAsync` hard-codes `Global` on profile creation — it's the only standard wired to fire without something else explicitly assigning a visa. §3 below shows how rarely that "something else" actually runs.

---

## 2. Seed-data counts (theoretical — what's defined, not what's used)

**27 questionnaires**, seeded entirely as SQL (`Kyc/Data/Questionnaire.sql`), no admin UI:

| Ids | Set |
|---|---|
| 1 | Comprehensive — every question, reference/testing only |
| 2–14 | Global Standard, one per `ProfileType` (13) |
| 15–27 | CBRE Standard, one per `ProfileType` (13) |

Sections aren't a table — they're an array inside `Kyc.Questionnaire.JsonConfig`, so there's no `Kyc.Section` row to count directly; each questionnaire's section count only exists inside that JSON blob.

---

## 3. Queries run, and real results

```sql
-- 1. Every questionnaire, with resolved standard + profile type names
SELECT q.Id, q.Name, pt.Name AS ProfileType, qs.Name AS Standard, q.IsActive
FROM Kyc.Questionnaire q
JOIN Kyc.ProfileType pt ON pt.Id = q.ProfileTypeRef
JOIN Kyc.QuestionnaireStandard qs ON qs.Id = q.QuestionnaireStandardRef
ORDER BY q.QuestionnaireStandardRef, q.ProfileTypeRef;

-- 2. Count of questionnaires per standard
SELECT qs.Id, qs.Name, COUNT(q.Id) AS QuestionnaireCount
FROM Kyc.QuestionnaireStandard qs
LEFT JOIN Kyc.Questionnaire q ON q.QuestionnaireStandardRef = qs.Id
GROUP BY qs.Id, qs.Name
ORDER BY qs.Id;

-- 3. Section count per questionnaire (JsonConfig is a JSON array of sections)
SELECT q.Id, q.Name,
       (SELECT COUNT(*) FROM OPENJSON(q.JsonConfig)) AS SectionCount
FROM Kyc.Questionnaire q
ORDER BY q.Id;

-- 4. Total sections across every seeded questionnaire
SELECT SUM(SectionCount) AS TotalSections
FROM (
    SELECT (SELECT COUNT(*) FROM OPENJSON(q.JsonConfig)) AS SectionCount
    FROM Kyc.Questionnaire q
) x;

-- 5. Total distinct questions (flat table, referenced by id from every JsonConfig)
SELECT COUNT(*) AS TotalQuestions FROM Kyc.Question;

-- 6. Real-world usage: how many profiles actually hold a visa for each questionnaire
SELECT v.QuestionnaireIdRef, q.Name, COUNT(*) AS ProfilesWithThisVisa
FROM Kyc.Visa v
JOIN Kyc.Questionnaire q ON q.Id = v.QuestionnaireIdRef
WHERE v.IsActive = 1
GROUP BY v.QuestionnaireIdRef, q.Name
ORDER BY ProfilesWithThisVisa DESC;

-- 7. Any profile assigned more than one active visa (candidates for "merged" behaviour)
SELECT v.ProfileIdRef, COUNT(*) AS ActiveVisaCount
FROM Kyc.Visa v
WHERE v.IsActive = 1
GROUP BY v.ProfileIdRef
HAVING COUNT(*) > 1
ORDER BY ActiveVisaCount DESC;
```

### Results

**Query 1/3 — all 27 questionnaires, with section counts:**

| Standard | Count | Section counts seen |
|---|---|---|
| Comprehensive Questionnaire (standard `None`) | 1 | 7 |
| Global Standard (13, one per `ProfileType`) | 13 | mostly 6; Joint Account = 5; Public Body = 4 |
| CBRE Standard (13, one per `ProfileType`) | 13 | mostly 6; Public Body = 4 |

**Total sections: 158. Total distinct questions: 135.**

**Query 2 — questionnaires per standard:**

| Standard | QuestionnaireCount |
|---|---|
| None | 1 |
| Global | 13 |
| US | **0** |
| Investment KYC | **0** |
| Hellman and Friedman | **0** |
| CBRE | 13 |

**Query 6 — real active-visa usage (25,660 active visas total):**

| Questionnaire | Profiles |
|---|---|
| Global Standard Individual | 11,949 |
| Global Standard Private Company | 5,652 |
| Global Standard Partnership | 3,423 |
| Global Standard Regulated Entity | 1,384 |
| Global Standard Trust | 1,073 |
| Global Standard LLC | 917 |
| Global Standard Pension | 353 |
| Global Standard Foundation or NPO | 297 |
| Global Standard Listed Entity | 292 |
| Global Standard Public Body | 155 |
| Global Standard University | 85 |
| Global Standard Joint Account | 49 |
| Global Standard Sovereign Wealth Fund | 30 |
| **CBRE Standard Partnership** | **1** |

**Query 7 — profiles with multiple active visas:** at least 7 profiles hold 3 simultaneously (`266301`, `283150`, `283147`, `266368`, `266312`, `266285`, `283142`), with more holding 2 beyond what the result set showed on screen.

---

## 4. What the real data shows that the seed data alone didn't

- **Three of the six standards are entirely unused, not just rare.** `US`, `Investment KYC`, and `Hellman and Friedman` have zero questionnaires seeded — not zero visas, zero *questionnaires to assign in the first place*. Any code path that assumes one of these three resolves to something real would hit the same "zero matches → `NotFoundError`" case `VisaService.AssignVisaForProfileAsync` already documents for a missing `(ProfileType, Standard)` pair.
- **CBRE is fully built but almost never used.** All 13 CBRE questionnaires exist and are active, but only **one profile in the entire system** — `CBRE Standard Partnership` — actually holds a visa against any of them, out of 25,660 total active visas. Worth confirming whether that single row is a genuine production onboarding or a test/leftover profile: if the latter, the CBRE standard may be effectively dead code with a live-looking database footprint; if the former, the non-Global assignment path exists and works, it's just never wired into any common flow.
- **Global Standard usage is heavily skewed.** Individual + Private Company alone account for ~69% of all active visas (17,601 of 25,660); Sovereign Wealth Fund (30) and the single CBRE row sit at the long tail.
- **Multiple visas per profile is real, but it's a bug, not merge behaviour being used as intended.** Follow-up in `12-Profile-Visa-And-Profile-Creation-Flow.md` §3a traced the 7 multi-visa profiles found here: on every one, all 2–3 visa rows point at the *identical* questionnaire, share the same `UserCreatedGlobal`, and were inserted within microseconds of each other — a race condition in `VisaService.AssignVisaForProfileAsync`'s check-then-insert pattern (no unique constraint backs it in the DB), not a profile legitimately holding two different standards. `QuestionnaireMergingService` explicitly dedupes visas by `QuestionnaireIdRef` before merging, so these profiles see a normal, unmerged questionnaire — the duplicate rows are wasted data, not a visible merge bug. The bigger finding: no code path in the module can currently give a profile two visas with genuinely *different* `QuestionnaireIdRef`s, so the merge-multiple-distinct-questionnaires logic itself appears to be dormant/unreachable in production today — see doc 12 §3a for the full trace.

---

## 5. Open follow-up

Confirm what `266301`, `283150`, `283147`, `266368`, `266312`, `266285`, and `283142` (the 3-visa profiles) actually are, and what the single CBRE Partnership profile is — both are small enough result sets to inspect directly and would settle whether CBRE and multi-visa merging are genuine production paths worth hardening, or edge cases from test/setup data.
