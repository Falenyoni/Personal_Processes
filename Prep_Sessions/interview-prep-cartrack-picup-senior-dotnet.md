# Senior .NET Developer Interview Prep — Cartrack & Picup

Weekend prep doc. Companion to `interview-notes-utano-notifications-and-debugging.md` (same
folder — that one's a personal-project story, still worth reading, but everything in *this* file
leads with real day-job evidence).

**Correction from the first draft of this file:** the primary evidence base is your actual work —
**TemplateAPI / SonataOne, specifically the TransferAgency module**, plus deep, current familiarity
with the **IDR → KYC migration** (`C:\Code\KYC-Notes.md`, your own handover/analysis notes while
transitioning onto that team). Utano and Indlela are personal side projects — genuinely useful to
mention (modern .NET 10, built solo, shows initiative), but clearly labeled as such, never
presented as production/day-job experience.

**Format note:** every story below is written the way you'd actually say it out loud — specific
decision, specific reason, specific outcome. Practice the "30-second version" of each section out
loud before Monday.

---

## 0. Company context

**Cartrack** — global fleet telematics / IoT SaaS. Vehicles carry a tracking unit (GNSS + sensors,
CAN bus data) streaming location/engine/speed data over cellular to a central data centre for
processing, served through a fleet-management platform. Core products: fleet management, stolen
vehicle recovery (time-critical), insurance telematics. 20+ years old, global scale — consistent
with a large existing .NET Framework estate and a real Framework → modern .NET move on the table.

**Picup** — Cape Town on-demand/crowd-sourced last-mile delivery platform (marketplace of drivers
matched to delivery jobs in real time), confirmed running on **Microsoft Azure**.

**What this means:** expect high-volume/high-frequency data questions from Cartrack (device pings,
out-of-order arrival, time-series storage, real-time alerting), real-time dispatch/marketplace
questions from Picup (race conditions, delivery state machines, webhook reliability). Both are
inherently **multi-tenant B2B platforms** — directly in your wheelhouse (Section D3).

---

## A. Architecture methodology

### A1. Monolith vs Modular Monolith vs Microservices

Don't reach for "microservices are the future" — that's the wrong answer for a senior interview.
The right answer is a decision framework, and you have real evidence on multiple points of it.

| | Monolith | **Modular Monolith** | Microservices |
|---|---|---|---|
| Deploy unit | 1 | 1 | N |
| Team scaling | Poor past ~1 team | Good — module boundaries = team boundaries | Best, at infra cost |
| Data consistency | Trivial (one DB) | Easy within a module, deliberate at boundaries | Hard — eventual consistency, sagas |
| When it's right | Small team/domain | Most products — the *default* | Independent scaling, independent deploy cadence, or a domain that's genuinely outgrown one process |

**TemplateAPI/SonataOne (your day job) is a modular monolith** — TransferAgency is one module
among several (Fund, KYC, Security, DocumentManagement, Admin, AnalystAction, RulesEngine,
DocuSign, IdPal) hosted as one deployable, each with its own layered structure (Domain /
Application / Infrastructure / Host / Models) and its own aggregates. Modules integrate through
shared interfaces and, where async decoupling is warranted, through events — not by one module
directly querying another's tables.

**The nuance to include if pushed on "why not microservices":** you have a *live counter-example*
in the same organization that argues for the modular-monolith default even more strongly — the KYC
migration (Section B, flagship story) is effectively running two full "services" (legacy IDR +
new TemplateAPI KYC module) that have to stay in sync over a message bus, and the hardest, riskiest
open problems in that whole migration are *exactly* the distributed-systems tax microservices
always cost: conflict resolution when both sides write near-simultaneously, messages that fail and
get silently stuck, no clear "source of truth" during the transition. That's not a theoretical
argument against microservices — it's the actual, current, lived cost of running two systems
instead of one, visible in a system you know in detail. Good ammunition for "have you seen the
downside of distributed systems in practice."

### A2. CQRS
TransferAgency and TemplateAPI use CQRS-lite via MediatR — `IRequestHandler<TCommand,
Result<TResponse>>` per use case (vertical slice: `FeatureEndpoint` / `FeatureHandler` /
`FeatureRequest` / `FeatureResponse`), controllers depend only on `ISender`. Same database serves
reads and writes — this is **not** full CQRS with a separate read store. Be precise about that
distinction if probed; conflating "I use MediatR" with "I do CQRS" is a tell.

**The one place a *separate read model* genuinely came up as a missing piece:** the KYC migration
(Section B) — see the reporting-parity gap below. That's your real example of knowing when CQRS's
full form (separate read projections) actually earns its cost, versus when CQRS-lite is enough.

### A3. Event-driven architecture vs event sourcing — nail the distinction

- **Event-driven**: state lives in normal tables (current values). Events are notifications that
  let other parts of the system react, sync or async. Not the source of truth.
- **Event sourcing**: the event log *is* the source of truth; current state is derived by replay.
  Full audit trail and temporal queries for free, but real cost in query complexity (need
  projections for anything you'd normally `SELECT`) and irreversible event-schema decisions.

**Your honest, well-evidenced answer:** you haven't built full event sourcing, and you can say
exactly why with a real example — the KYC domain (Section B) is precisely the kind of
compliance-heavy, audit-critical domain that's the textbook case *for* event sourcing, and your own
analysis of that system independently arrived at the same conclusion: `SyncEvent` already captures
every change with actor/timestamp/correlation ID, and leaning into that as the canonical audit
mechanism (rather than the current mix of separate ad hoc `*AuditEntry` tables per entity type)
would be a stronger foundation than what exists today. That's a much better answer than "no, never
used it" — it's "no, but I can point at the exact place in a real system where it would pay for
itself, and articulate why."

---

## B. Flagship story: the IDR → TemplateAPI KYC strangler-fig migration

This is your strongest material for almost every architecture/migration question they can ask.
It's real, current, and you know it in genuine depth — not from having designed it from scratch,
but from doing exactly the kind of deep structural analysis a senior engineer should be able to do
on an unfamiliar system: reading the code, tracing git history, mapping the sync mechanism, and
forming an independent, evidenced opinion on what's solid and what's risky. **Be honest about that
framing in the interview** — "I've been getting deeply familiar with this while transitioning onto
the KYC team" is a *strong* answer, not a hedge; claiming personal authorship of a migration you
didn't design would be the actual mistake.

### B1. The shape of it
```
IDR (legacy, .NET Framework 4.8)         TemplateAPI (new, .NET 10)
InvestorServices.Api/DD/Models   <--->   S1.Module.Kyc
      ^                                        ^
      | InvestorServices.Sync.Service          | KycSyncOutboxWorker
      | Saga: LegacyToRebuild /                | -> SyncDataOutboxMessage (same DB txn)
      |       RebuildToLegacy                  | -> Azure Service Bus
      +------------- Azure Service Bus --------+   (topic: kyc-global-events)
```
Both systems run **simultaneously** and stay in sync via events while the new module gradually
takes over. This is a **strangler-fig migration**, but the harder, more advanced version of it —
not "route new traffic to the new system," but "keep two full systems, old and new, consistent
with each other in both directions while the cutover happens incrementally per feature area."

### B2. The mechanism, precisely (this is the part worth being fluent in)
- **Transactional outbox pattern**: every domain entity implements `IHasSyncEvents`. A change
  (e.g. `Profile.CreateNewProfile`) raises a `SyncEvent` with a DTO snapshot, persisted as a
  `SyncDataOutboxMessage` row **in the same database transaction** as the actual change — so the
  event can never be silently lost even if the publish step later fails. This is the textbook
  correct way to combine "write to your own DB" with "publish an event," without a two-phase
  commit across a database and a message bus.
- A background worker (`KycSyncOutboxWorker`, polls every 30s) claims a batch via an **optimistic
  lock** (`LockId`/`LockedAt`, expiry ~5 min — so a crashed worker doesn't hold the batch forever),
  publishes each message to Azure Service Bus, marks it processed or records an error.
- On the IDR side, a **Saga** (`EntityChangedNotificationHandler`) consumes those events and routes
  them by direction — `LegacyToRebuild` (old→new) or `RebuildToLegacy` (new→old) — keeping both
  stores in lockstep during the transition window.
- **Permissions are a separate, synchronous bridge** — the new module still calls IDR live
  (Redis-cached) to resolve legacy permission entity IDs, explicitly marked as temporary scaffolding
  (a debug endpoint literally named `LegacyDiagnostics` with a `// TODO: remove` comment).

### B3. What your own structural review found — this is your "how do you evaluate an architecture" story
Framed as P0 (data-integrity risk)/P1 (tech debt)/P2 (strategic gap), grounded in actual file
references, not generic:

- **P0 — failed outbox messages get permanently stuck.** The batch-claim query only picks up rows
  where `Error == null`; once a message fails once (transient Service Bus blip, serialization
  error) it's excluded from every future batch forever — no retry, no backoff, no dead-letter path.
  In a dual-write sync, a silently dropped message means the two "sources of truth" quietly diverge
  and nobody notices. **This is the single most important thing to fix in any strangler-fig sync
  mechanism** — the sync's correctness *is* the migration's correctness.
- **P0 — no conflict resolution for bidirectional sync.** If the same record is edited in both
  systems near-simultaneously, nothing defines which write wins. Needs an explicit rule
  (version/timestamp check, last-writer-wins, or a lock) — otherwise silent data loss is possible in
  either direction.
- **P0 — a two-phase-commit-shaped gap**: one handler calls an external Security API *before* its
  own DB transaction commits. If the DB commit later fails, the external side effect is already
  applied with no compensating action. Same bug class anywhere an external call isn't wrapped by
  the unit of work.
- **P2 — the real strategic gap**: the new module's flexible EAV-style `Questionnaire`/`Answer`
  model is a genuine structural win over the legacy system's ~180 hardcoded nullable DD columns
  (composable per-fund questionnaire standards vs. "add a migration for every new question") — but
  that flexibility makes direct SQL reporting much harder, and legacy's flat-column reports have no
  equivalent yet. **The fix isn't to abandon the flexible model — it's CQRS done properly**: reuse
  the exact same event stream that already drives the sync, add a second consumer that projects
  into a denormalized reporting schema. Same event source, two read shapes — the write model stays
  flexible, reporting gets a purpose-built model instead of querying the write model directly.

### B4. What "the ideal version of this migration" would have looked like — your retrospective answer
This is your direct, evidenced answer to "how would you approach a similar migration":
1. **CQRS from day one, not retrofitted after the write model was already built.** Pair a flexible
   write model with a purpose-built read model fed by the same event stream from the start, instead
   of treating reporting as an afterthought once the schema shape was already locked in.
2. **Treat the sync layer as tier-1 infrastructure, not plumbing.** Retry/dead-letter/observability
   should ship *with* the outbox on day one, not get discovered as a gap later — because in a
   dual-source-of-truth migration, a silently dropped sync message is the single worst failure mode
   there is.
3. **Prefer phased write-ownership handoff over permanent bidirectional sync where you can.**
   Bidirectional sync is powerful but requires solving conflict resolution, which is genuinely hard.
   A model where legacy becomes read-only for a bounded context the moment that context cuts over
   sidesteps the whole bug class, at the cost of needing an explicit per-feature cutover sequence.
4. **Give temporary bridges (like the synchronous legacy-permission call) a committed decommission
   milestone**, not a floating TODO — "temporary" scaffolding with no deadline tends to become
   permanent load-bearing coupling by default.

### B5. 30-second version
*"There's a live strangler-fig migration in my org — a legacy .NET Framework KYC system being
replaced by a new module in our modern .NET modular monolith, with both systems running
simultaneously and syncing via a transactional outbox and a bidirectional saga over Azure Service
Bus. I've spent real time getting deep into both sides of it while moving onto that team, and did a
structural review: found the outbox silently drops messages permanently on first failure with no
retry or alerting — the single riskiest thing in the whole migration, since a dropped sync message
means the two systems quietly diverge — plus a missing conflict-resolution rule for simultaneous
edits, and a strategic gap where the new flexible questionnaire model makes reporting harder than
the old rigid schema, with no read-model projection built yet to solve it. My fix for that last one
is CQRS done properly — reuse the same event stream, add a reporting projection, instead of
querying the flexible write model directly."*

---

## C. Real stories from TransferAgency (your actual day-to-day work)

TransferAgency's real aggregate map (confirms and sharpens anything you say about it): `Commitments`,
`CapitalTransactions`, `CapitalAccountStatements`, `BankDetails`, `BankStatements`,
`InterestTransfers`, `Securities`, `Splits`, `Documents`, `FundEndReports` — plus shared
cross-cutting building blocks: `Shared/Approval` (maker-checker), `Shared/Archiving`,
`Shared/DomainValidation`. Feature areas layer on top: CapitalActivity, Reconciliation, Payment,
InvestorPortfolio, InvestorRegister, Notices, Projects, TransferOfInterest, Currencies.

### C1. Maker-checker / Approval pattern (real, confirmed in `Shared/Approval`)
`Approval` models the state (Pending/Approved/Rejected — one-way transitions only), `IApprovable`
is implemented by aggregates that need governance, with the core business rule that the approver
can never be the same user who made the request. This is standard fund-administration/fintech
governance — every capital transaction, bank detail change, or interest transfer that moves real
investor money needs a second set of eyes, enforced by the system rather than trusted to process.

**Why this is relevant to Cartrack/Picup:** any workflow with real liability attached (stolen
vehicle recovery dispatch authorization, a driver payout adjustment, a delivery dispute override)
plausibly needs the same governance shape. Worth raising proactively if they describe a workflow
like that.

### C2. Domain design tradeoff — rigid schema vs flexible model (direct parallel to B3)
The same tension shows up inside TransferAgency's own domain, not just in the KYC comparison:
`CapitalTransactions`/`Commitments` model well-defined, relatively stable fund-administration
concepts (a capital call, a distribution, a commitment amount) where a normal relational schema is
the right fit — contrast that with KYC's genuinely variable, standard-dependent questionnaire needs,
where the EAV model earns its complexity. **The lesson worth stating explicitly:** "flexible/generic
model" isn't universally better or worse than "rigid, well-typed schema" — it's a fit-for-purpose
decision per aggregate, and the KYC migration is a real example of getting that decision right on
the write side while under-investing in the read side to compensate for it.

### C3. Shared/Archiving and Shared/DomainValidation
Cross-cutting concerns pulled out as reusable domain building blocks rather than reimplemented per
aggregate — the same instinct as TemplateAPI's shared `AggregateRoot`/event-dispatch base classes.
Good evidence for "how do you avoid duplicating cross-cutting logic across many entities" — pull it
into a shared abstraction *once* it's needed by more than one aggregate, not preemptively.

---

## D. Multi-tenancy — two real models you can compare

You have two genuinely different real multi-tenancy implementations to contrast, which is a
stronger answer than describing just one:

### D1. Tenant-per-Azure-AD-tenant (IDR / SonataOneSecurity, your day job)
IDR's real setup: separate **Main Tenant** and **Customer Tenant** Azure AD app registrations, each
issuing custom claims (`s1AuthId`, `s1UserId`, `s1Source`, `s1Roles`, `s1Permissions`) consumed
across the whole system. Tenant isolation lives at the *identity provider* level — a customer's
users authenticate against their own Azure AD tenant, not a shared login with a discriminator
column.

**Tradeoff to state if asked:** this gives the strongest possible isolation (a compromised token
from one tenant literally cannot authenticate against another tenant's app registration) and fits
naturally when customers have their own enterprise Azure AD already — but it's heavier to
provision (new app registrations, new tenant configuration per customer) and doesn't fit
self-service signup well.

### D2. Shared-database with a discriminator column + global query filter (personal projects)
On the side, built this the lighter-weight way in two personal .NET 10 projects — every
tenant-scoped entity carries an `OrganizationId`/`PracticeId`, a global EF Core query filter
applies it to every query automatically (structurally impossible to forget to scope a query,
because scoping isn't opt-in per query), bypassed only in explicitly-commented cases (a background
job scanning across all tenants, an admin-only cross-tenant endpoint).

**Tradeoff:** cheap, simple, self-service-friendly, no cross-database joins ever — but weaker
isolation (a bug in the query filter is a real cross-tenant data leak risk) and no easy path to
"this one enterprise customer needs guaranteed physical isolation."

**How to answer "how would you design multi-tenancy":** lead with D2 as the sensible *default* for
most B2B SaaS (cheap, simple, fast to build), name D1 as the escalation you reach for when a
specific customer's compliance/enterprise-IT requirements demand real isolation at the identity
layer — and that you've actually worked with both, not just read about the tradeoff.

---

## E. .NET Framework → modern .NET migration — you have a real "before" and "after" in the same company

### E1. The concrete gap, evidenced
IDR (`InvestorServices.Api`) is real: `<TargetFrameworkVersion>v4.8</TargetFrameworkVersion>`,
classic non-SDK-style `.csproj` (`ToolsVersion="12.0"`), **Autofac** for DI (not built-in
`Microsoft.Extensions.DependencyInjection`). TemplateAPI/TransferAgency is `net10.0`, SDK-style
projects, built-in DI throughout. You don't need to imagine what this migration looks like — you
can describe the actual before/after inside your own organization.

### E2. Strategies — know all three, the KYC migration is your live example of the third
1. **Big-bang rewrite** — highest risk, easiest to whiteboard, hardest to deliver on schedule.
2. **Strangler Fig via routing** — new features built in modern .NET, traffic gradually
   redirected (reverse proxy / API gateway) as endpoints migrate. Lower-risk default for stateless,
   read-mostly surfaces.
3. **Strangler Fig with dual-write sync (what the KYC migration actually does)** — needed when the
   *data itself*, not just the traffic, has to stay valid and consistent in both the old and new
   store during a long transition, because the new module isn't feature-complete enough for an
   immediate full cutover. Harder to get right (see B3's P0 findings) but sometimes unavoidable when
   the domain is too large/critical to cut over all at once.

**Your answer to "how would you approach modernizing a large legacy .NET Framework estate":**
inventory the estate first (every project, every Framework-only dependency, every third-party
package's compatibility), pick strangler-fig-by-routing for anything stateless, reach for
dual-write-with-outbox-sync only where the domain genuinely can't cut over as one unit — and if you
do need the sync approach, ship retry/dead-letter/monitoring on the sync mechanism *from day one*,
because that's the exact thing the KYC migration under-invested in early and is now paying down as
technical debt.

### E3. Concrete gotcha checklist (name 4+ unprompted — shows real experience, not a blog post)

| Framework thing | Modern .NET replacement | Why it bites people |
|---|---|---|
| Autofac / other third-party DI container | Built-in `Microsoft.Extensions.DependencyInjection` | Not just a swap — Autofac supports things (property injection, more flexible lifetime scopes) the built-in container doesn't do the same way; registration code needs real translation, not find-replace |
| `Web.config` | `appsettings.json` + `IOptions<T>` | Static/global config becomes DI-injected and scoped — changes how deep code accesses config |
| `HttpContext.Current` | `IHttpContextAccessor` (injected) | The static-ambient pattern is gone; anything reading it off-thread needs a real refactor |
| Classic non-SDK `.csproj` | SDK-style `.csproj` | Different build system, implicit file globbing instead of listing every file, different package reference format |
| IIS in-process, `Global.asax` | Kestrel + minimal hosting, middleware pipeline | Framework's request-lifecycle events become explicit middleware — every one needs finding and translating |
| WCF | gRPC, or CoreWCF if a straight port is unavoidable | WCF server isn't supported in modern .NET at all — a genuinely hard item if present |
| EF6 | EF Core | Different config API, some LINQ that silently client-evaluated in EF6 throws or behaves differently in Core — needs query-by-query review |
| `ConfigurationManager.AppSettings` | `IConfiguration` | Same static→injected shape problem as Web.config |
| `SynchronizationContext`-dependent async code (`.Result`/`.Wait()`) | ASP.NET Core has no sync context by default | Old code that "worked" in Framework because of the sync context can deadlock differently, or just behave differently — test explicitly, don't assume |

---

## F. Azure ↔ On-Prem

Prepare for either direction. Moving *off* a cloud provider (repatriation) is a real, legitimate
trend for cost-at-scale reasons (Basecamp/37signals' public AWS-exit story is the well-known
example) — not a step backward.

### F1. Why a company does this
Cost at sustained/predictable scale (cloud is cheapest for spiky load, more expensive than owned
hardware for large steady baseline load — IoT device ingestion at Cartrack's scale is a plausible
candidate); data sovereignty/compliance (POPIA in South Africa, or contractual requirements from
enterprise/government customers); latency (device data ingestion benefits from being physically
close to where devices are); vendor lock-in avoidance.

### F2. How to design for portability
Core technique: never call a cloud SDK directly from business logic — an interface in the shared
layer, a swappable implementation in infrastructure, chosen per environment via DI. This is the
exact same discipline as B2's transactional outbox and D2's tenant-scoping — put the seam where you
need flexibility, one layer removed from the thing that changes.

| Azure service | Portable equivalent |
|---|---|
| Azure Service Bus | RabbitMQ, or **MassTransit** as an abstraction over either — the standard .NET answer, same app code against Service Bus or RabbitMQ, only transport config changes |
| Azure Blob Storage | MinIO (S3-compatible, self-hostable) |
| Azure Key Vault | HashiCorp Vault, or Kubernetes Secrets |
| Azure AD (D1's model) | On-prem AD/ADFS, or Keycloak (self-hostable OIDC) |
| Azure SQL / Cosmos DB | Self-hosted SQL Server/PostgreSQL — the harder migration, since PaaS-specific features (geo-distribution, elastic scaling) don't have a drop-in equivalent, only a "redesign around what you actually need" one |
| AKS | Containerize + Kubernetes generally — **the real portability layer**: if deployment is already Docker + k8s manifests/Helm rather than Azure-specific tooling, "move to on-prem" becomes "point kubectl at a different cluster," not a rewrite |

**Honest caveat:** portability isn't free — you trade managed-service convenience for it, and that
trade should be deliberate per service, not a blanket rule. "Enough abstraction that switching is a
project, not a rewrite" is the right target, not zero cloud-specific features ever.

---

## G. Domain-specific deep-dive prep

### G1. Cartrack-flavored: high-volume telemetry ingestion
Likely topics: ingesting millions of GPS/sensor pings/day without falling behind; devices that go
offline and replay a backlog (out-of-order arrival); idempotent processing (a device retry
shouldn't double-count); time-series storage; real-time alerting latency for stolen-vehicle
recovery.

**Bridge from what you actually know:** the transactional-outbox pattern (B2) is directly relevant
vocabulary — "never lose an event even if the downstream publish fails" is exactly the same
guarantee a device-ingestion pipeline needs for "never lose a GPS ping even if the processing
pipeline is temporarily down." Be honest that the *volume* is new to you, but the pattern
vocabulary (idempotency, out-of-order handling, at-least-once processing, dead-letter handling) is
not — and you have a real, current example (B3's P0 finding) of exactly what goes wrong when a
system *doesn't* have that dead-letter/retry path.

### G2. Picup-flavored: real-time dispatch/marketplace matching
Likely topics: two dispatches racing for the same driver (optimistic concurrency + retry vs.
pessimistic lock vs. single-writer-per-driver queue); delivery status state machine design; webhook
reliability with third parties.

**Bridge:** the Approval/maker-checker pattern (C1) and the general domain-state-guard discipline
(one-way status transitions, validated at the point of transition, not scattered `if` checks) is
the same *shape* of problem applied to a delivery instead of a capital transaction.

---

## H. Likely questions + answer skeletons

- **"Monolith vs microservices?"** → A1, cite the KYC migration's real distributed-systems cost as
  evidence, not just theory.
- **"Have you done event sourcing? Would you use it here?"** → A3. Precise distinction, honest "no,
  but here's exactly where in a real system I'd apply it and why."
- **"How would you migrate a legacy .NET Framework estate?"** → E2/E3. Inventory first, strategy
  choice depends on whether data itself needs dual-write sync, name 4+ concrete gotchas unprompted.
- **"Tell me about an architecture you evaluated and found real problems in."** → B3/B4. Your
  strongest answer — specific, prioritized (P0/P1/P2), ends with "here's what I'd have done
  differently and why."
- **"How do you handle eventual consistency / keeping two systems in sync?"** → B2. Transactional
  outbox + optimistic-lock batch claim + saga routing, described precisely.
- **"How would you design multi-tenancy?"** → D. Two real models, honest tradeoffs, D2 as sensible
  default.
- **"Azure or on-prem — does it matter to you?"** → F. Neutral, portability-by-design, name concrete
  swap points.
- **"What's a mistake you found (yours or someone else's) and how did you evaluate it?"** → B3 is
  strongest; the JWT-claim-remapping bug in the companion doc (a real bug in your own personal
  project, root-caused via a disciplined throwaway-repro process) is a good second example if they
  want a "yours specifically" story.

---

## I. Questions to ask them

- "Is the current platform a monolith, or already split into services — and if split, was that
  driven by team-scaling or by a genuine load-isolation need?"
- "For the Framework → modern .NET move — is it planned as strangler-fig-by-routing, or does data
  need to stay consistent across old and new during a longer transition (dual-write/sync), the way
  a lot of KYC-style migrations end up needing?"
- "How do you currently handle device data that arrives late or out of order — is there a
  reconciliation/backfill process, or does late data just get dropped?" (Cartrack)
- "How does dispatch handle two near-simultaneous assignment attempts for the same driver today?"
  (Picup)
- "Is there appetite to move things off Azure entirely, or is on-prem about specific
  data-residency-sensitive workloads rather than the whole platform?"
- "If you have any event-driven sync between systems today, how do you monitor for stuck/failed
  messages — is that something the team already has visibility into?"

---

## J. Weekend checklist
- [ ] Read `interview-notes-utano-notifications-and-debugging.md` once, out loud
- [ ] Practice the B5 30-second KYC-migration summary until it's fluent without notes
- [ ] Re-skim the E3 gotcha table until you can name 4+ without looking
- [ ] Decide your honest, confident answer to "have you done event sourcing" — A3 is the answer,
      don't waffle
- [ ] Memorize 2-3 of Section I's questions well enough to ask naturally
