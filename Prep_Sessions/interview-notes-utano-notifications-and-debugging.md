# Utano: Appointment Notifications + Three Production Bugs Found While Testing

Session: 2026-07-30. Built appointment notifications for Utano (a multi-tenant .NET 10 modular
monolith for medical practice management), then found and fixed three real, pre-existing bugs
while testing it. Useful as interview material for architecture decisions *and* debugging
methodology — not just "I wrote a feature."

---

## Part 1 — What was built: domain-event notifications

**The ask:** notify staff when an appointment is booked, rescheduled, cancelled, or reassigned.

**The architecture decision that matters:** two ways to wire "Appointments" into "Notifications"
were on the table.

- **Direct call** — Appointments handler calls `INotificationRepository` straight, via an
  interface defined in the shared kernel (the exact pattern already used elsewhere in this
  codebase for `IPatientStatusChecker`). Cheap, one file, works for exactly one relationship.
- **Domain events** — the `Appointment` entity raises an event when it changes state; an EF Core
  `SaveChangesInterceptor` publishes queued events via MediatR after `SaveChanges`; Notifications
  subscribes with a handler. Appointments never knows who's listening.

**Why domain events won:** not because it's more "proper" in the abstract — because Billing
already carried `AppointmentId`/`VisitId` on its invoices, meaning a second real consumer
(auto-invoicing off appointment completion) was a plausible near-term ask, not a hypothetical.
Direct calls scale to *N* consumers by editing the publisher every time; events scale by adding a
handler with zero changes to the publisher. That's the actual argument for YAGNI-vs-events: it's
about whether more than one thing needs to react, not about which pattern is "cleaner."

**A design correction worth mentioning:** the first draft of the event-queue capability
(`AddDomainEvent`/`DomainEvents`) was going on the shared `AggregateRoot` base class — meaning
*every* entity in *every* module would silently get an event queue whether it used one or not.
Caught it before building: made it an opt-in `IHasDomainEvents` interface instead, implemented
only by entities that actually raise events. `AggregateRoot` itself never changed. Small decision,
but it's the difference between "blast-radius risk to the whole codebase" and "one extra
interface on the one class that needs it" — and it's the kind of thing worth catching yourself
rather than have a reviewer catch it.

**Payoff confirmed same session:** the interceptor was built once, for Appointments →
Notifications. A few hours later, `ClinicalNotes`' audit logging was converted to the same
pattern (`Visit` raises `VisitCompleted`/`VisitTriaged`/`VisitClinicalNotesUpdated`, an audit
handler reacts) — **zero changes to the interceptor itself**. That's the actual test of "is this
reusable infrastructure or did I just build something bespoke and call it general."

---

## Part 2 — Three bugs found while testing, and how each was actually diagnosed

### Bug 1: `ICurrentUserService.UserId` always resolved to `Guid.Empty`

**Symptom:** built the notification feature, booked an appointment for a test doctor, logged in
as that doctor — bell showed nothing.

**Wrong turns first (worth including — this is the honest version):**
1. Assumed stale React Query cache from a prior login in the same tab. Real bug (fixed it —
   `queryClient.clear()` was never called on login/logout), but not *the* bug — a fresh
   incognito-equivalent browser tab, logged in fresh, still showed nothing.
2. Assumed the running backend process was stale (hadn't picked up code changes). Checked file
   timestamps on the compiled DLL against the process start time — they matched. Not it either.

**How it actually got isolated:** rather than keep guessing, went data-first.
- Queried the production database directly (found the real connection string via
  `dotnet user-secrets list`, since it wasn't in `appsettings.json`) — the notification rows
  existed, with the exact right `RecipientUserId` and `PracticeId`.
- Wrote a **throwaway standalone console app** that referenced the actual Notifications project
  and ran the exact same EF query against the same database with a hand-built stub
  `ICurrentUserService` — it returned the correct 4 rows. So: the query is right, the data is
  right, the *live endpoint* is still wrong. That combination — isolated code path works, live
  endpoint doesn't — points at the web request pipeline specifically, not the feature code.
- Added a temporary `/api/notifications/_debug` endpoint that just echoed
  `currentUserService.UserId/PracticeId/Role/Email` back as JSON, hit it with a real browser JWT.
  `PracticeId` and `Role` were correct. `UserId` and `Email` were `Guid.Empty`/`""`.

**Root cause:** ASP.NET Core's JWT bearer handler remaps well-known short claim names (`sub`,
`email`) to their long `ClaimTypes` equivalents during token *validation*, by default
(`MapInboundClaims = true`). The token issuer used the short names; the claims reader
(`CurrentUserService`) also read the short names — but by the time the token was validated, they'd
already been silently renamed underneath. Custom claims (`"PracticeId"`) and claims already
issued in long form (`ClaimTypes.Role`) were untouched by the remap, which is exactly why *those*
worked and masked the bug everywhere except code that specifically needed `UserId`/`Email`.

**Fix:** one line — `options.MapInboundClaims = false;`.

**Blast radius, once you know what to look for:** grepped every use of
`currentUserService.UserId` across the codebase. Found it had also completely broken
self-service password change (`GetByIdAsync(Guid.Empty)` → "User not found" for *every* user,
*every* time) and had been quietly writing `Guid.Empty` into every audit log entry ever created
(the human-readable `UserName` was fine — it read a claim that was already long-form — so nobody
had noticed the `UserId` column was garbage).

**Verified the fix, not just declared victory:** round-tripped a real password change through the
actual API (change → re-login with new password → change back → confirm original still works),
rather than trusting that "the debug endpoint looks right now" was sufficient.

**Why this is a good interview story:** it's not "I found a bug," it's "I had a plausible-sounding
theory twice, tested it against real evidence both times, and moved on when the evidence didn't
support it" — and the actual isolation technique (reproduce in a minimal throwaway harness outside
the app, compare against the live path) is a transferable debugging skill, not a one-off trick.

---

### Bug 2: no audit trail for the most audit-worthy action in the system

Once the audit log's `UserId` bug was fixed, went looking for what else touched
`IAuditService`. Found only two call sites in the entire app — `Visit.Complete()` and
`Visit.Triage()` — despite `UpdateVisitHandler` also editing diagnosis, treatment, prescription,
symptoms, and clinical notes on every save, completely unaudited. In a medical records system,
editing a diagnosis is arguably *more* audit-relevant than marking a visit complete.

Also surfaced, while investigating: `IAuditService` only exists in the `ClinicalNotes` module —
Billing, Inventory, Patients, and Identity have no audit trail of any kind. Flagged as a separate,
larger product decision rather than folding it in reactively — not every gap found mid-session
should get fixed mid-session; some need a deliberate scoping conversation.

**Fix:** same domain-event pattern, third and fourth adopters —
`VisitClinicalNotesUpdatedEvent` raised from `UpdateClinicalNotes()`, audited the same way.

---

### Bug 3: no exception-handling middleware in the entire API

Found while re-checking an earlier throwaway observation instead of letting it drop: a wrong
password on login had returned a raw `500` with a stack trace, not a clean `400`. Traced it
structurally — grepped the whole `Utano.API` project for `UseExceptionHandler`,
`IExceptionHandler`, anything exception-related in the request pipeline. Nothing. Zero.

**Why this mattered more than it looked:** the frontend's API client code already expected a
structured error response —
```ts
const err = await res.json().catch(() => null)
throw new Error(err?.detail ?? 'Failed to book appointment')
```
— the standard ASP.NET Core `ProblemDetails` shape. But since nothing ever produced that shape,
`res.json()` always threw, got swallowed by `.catch(() => null)`, and the user saw the generic
fallback string — never the specific, deliberately-written message from the backend (e.g. *"The
doctor already has an appointment in that time slot"*). Every `UtanoDomainException` thrown
anywhere in the app, for any validation failure, had been silently replaced by generic text or a
raw stack trace, this entire time.

**Fix:** one `IExceptionHandler` implementation — `UtanoDomainException` → `400` with a
`ProblemDetails` body (`detail` = the real message); anything unexpected → sanitized `500`,
logged server-side, no internals leaked to the client. Registered as the first thing in the
middleware pipeline.

**Why this is worth telling:** it's the highest-leverage fix of the whole session — a single
20-line file that fixes error UX for every feature in the entire application, and it was found by
not dismissing a small, easy-to-ignore anomaly ("huh, that should've been a 400") as unimportant.

---

## Talking points if asked to summarize in 30 seconds

*"Built a domain-events pipeline for appointment notifications in a modular monolith — chose
events over a direct call specifically because a second module already had the data shape to
become a second consumer, not because events are inherently better. While testing it, traced a
'why is this empty' symptom through three wrong theories to the actual root cause — a JWT claim
remapping default — using a throwaway isolated repro instead of guessing at the live system.
That one fix also happened to unbreak password changes and audit logging that had been silently
broken the whole time. Then found the app had no global exception handling at all, which meant
every specific error message the backend ever wrote had been invisible to users — fixed that too.
Same session, same debugging instinct: don't accept 'weird, moving on' for something you can't
explain."*
