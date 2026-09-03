# Zachman Matrix — Apex/SonataOne KYC Platform

**Terms used throughout:** [`Glossary.md`](./Glossary.md).

**Purpose:** per `../README.md`'s explanation of Zachman, this isn't a process artifact — it's a completeness check. The value is entirely in what it shows as *missing*, not in the cells that are easy to fill. Populated only with what's actually confirmed by direct investigation; every empty or partial cell is left visibly empty rather than filled with a guess.

Columns are the 6 interrogatives; rows are the 6 perspectives, most abstract (top) to most concrete (bottom).

---

| | **What (Data)** | **How (Function)** | **Where (Network)** | **Who (People)** | **When (Time)** | **Why (Motivation)** |
|---|---|---|---|---|---|---|
| **Scope (Contextual)**<br>*Planner's view* | KYC subject data exists across ≥2 unlinked services (IDR, S1.Module.Kyc, ManagedServices) | Onboarding, ongoing due diligence, fund admin, transfer agency, tax compliance | IDR (legacy) + TemplateAPI modules + ManagedServices, sync'd via Sync/SyncExchange | Investor, Analyst, Fund Manager*, Compliance Officer | Not captured — no confirmed enterprise-level event/cycle calendar | Legacy replacement (strangler-fig, in progress, unplanned as a formal program) |
| **Business Model (Conceptual)**<br>*Owner's view* | Business capability map — `03-Phase-B-...md §2` | Investor/Analyst/Fund Manager journeys — `02-Journey-Scoping.md`, `03-Phase-B-...md §5-6` | Not yet mapped — which capability is delivered by which *business unit*, not system | Actor/Capability matrix — `03-Phase-B-...md §3` | Not captured | Business drivers (`02-Phase-A-...md §2`); gap analysis — `03-Phase-B-...md §7` |
| **System Model (Logical)**<br>*Designer's view* | Conceptual data model (Profile/Answer/Visa/Questionnaire) — `00-...md`; target schema-per-module map — `04-Phase-C-...md §4` | Questionnaire assembly + answer submission flow — `12-Profile-Visa-...md §4` | Application Communication diagram — `00-...md`; modular monolith vs. surrounding-services boundary — `04-Phase-C-...md §3` | KYC access control model — `01-KYC-Explained-And-Access-Control.md` | Sync trigger points (profile create, answer submit) — `12-...md §3` | Architecture Principles — `01-Preliminary-...md §3` |
| **Technology Model (Physical)**<br>*Builder's view* | `Kyc.Answer`/`Kyc.Question`/`Kyc.Visa` schema, confirmed via direct DDL reads | `SubmitAnswersHandler`/`AnswerReconciliationService` reconciliation logic | Technology Portfolio catalog with confidence levels - `05-Phase-D-...md §3`; still not confirmed: deployment topology for `TemplateAPI`/`IDR`/`ManagedServices` | Not captured - no confirmed IAM/role schema below the conceptual level | Not captured | Not applicable at this row (motivation is contextual/conceptual, not physical) |
| **Detailed Representations**<br>*Subcontractor's view* | Real production data samples — `11-Questionnaire-Standard-And-Section-Counts.md` | Actual C# implementation, file/line level, throughout this session's work | Not captured — no deployment topology (pods/instances/regions) confirmed | Not captured | Not captured — no confirmed cron/batch schedule inventory | Not applicable at this row |
| **Functioning Enterprise**<br>*the actual running thing* | Confirmed via live SQL queries against production — `11-...md §3` | Confirmed via live bug traces (DOB echo loop, tax-residence duplication) | Not directly observed — inferred from code/config only | Not directly observed | Not directly observed | Not directly observed |

*\* "Fund Manager" status as a real actor is itself an open question — see `02-Journey-Scoping.md §6`.*

---

## What this matrix says, read as a whole

- **The `Where` and `When` columns are the thinnest across every row.** Almost nothing is confirmed about physical deployment topology, geographic/regional distribution, or any enterprise-level time/event model. This is a genuine, actionable gap — not a formatting artifact of this table.
- **The `What` and `How` columns are the strongest**, because that's exactly where this session's investigation concentrated (data model, sync logic, bug traces). That's a real bias in the underlying source material, not a property of the platform — a future pass through Phase B/C should actively correct for it rather than assume the strength is representative.
- **The `Functioning Enterprise` row is the weakest of all** — almost everything here comes from *code and schema*, not from observing the live running system (metrics, actual traffic patterns, actual incident history). Phase B/C work should prioritize closing this specific row before assuming the System/Technology Model rows are sufficient on their own.

## Explicit backlog of gaps to close

1. `Where` column, all rows below Scope - Phase D (`05-Phase-D-...md`) looked for deployment/infrastructure topology directly and confirmed it's genuinely not documented anywhere, not just unchecked - containerization is only confirmed for 2 of roughly 10 services in scope (`Sync`, `SyncExchange`).
2. `When` column, every row — no event/cycle/schedule inventory exists.
3. `Who` column, Technology/Detailed rows — access control is understood conceptually (`01-KYC-Explained-And-Access-Control.md`) but not down to schema/implementation level.
4. `Functioning Enterprise` row — needs direct observation (metrics, logs, incident history), not more code reading.
5. **`Who`, Scope/Business Model rows — the Fund Manager attribution question (`03-Phase-B-...md §4`)** is the one gap on this list that isn't a matter of gathering more evidence from code; it needs someone with visibility into live-granted permissions to answer whether "Fund Manager" is a real external actor or an internal-staff-performed capability. Every other open item above can, in principle, be closed by more investigation — this one can't.
