# TOGAF (The Open Group Architecture Framework)

Full breakdown of TOGAF, pulled out of the main Enterprise Architecture notes into its own
folder since it's the framework this plan focuses on learning first. See
`../README.md` for the "EA before frameworks" context and the Zachman comparison.

---

## What it is

TOGAF is the most widely adopted EA methodology, maintained by The Open Group. It provides an
end-to-end, iterative **process** for developing, governing, and maintaining an enterprise
architecture — not just a way to classify artifacts (that's Zachman's job), but a way to actually
*produce* them and keep them current as the organization changes.

Currently on **TOGAF 10** (The Open Group retired the TOGAF 9 track) — same core method as TOGAF
9, reorganized and modernized in presentation.

---

## Core Components

- **ADM (Architecture Development Method)** — the heart of TOGAF, a cyclical, phase-based
  process for developing an architecture from scratch and evolving it over time (broken down
  phase by phase below).
- **Enterprise Continuum** — a way of classifying architecture and solution artifacts on a
  spectrum from generic/foundational to organization-specific, so reusable patterns aren't
  reinvented per project. Splits into two continuums:
  - **Architecture Continuum** — abstract → specific *architecture* artifacts (generic
    foundation architectures → industry-specific → organization-specific).
  - **Solutions Continuum** — the corresponding abstract → specific *implementations* of those
    architectures (generic reusable products/services → organization-specific deployed
    solutions).
- **Architecture Content Framework** — a structured way to describe architectural work products:
  - **Deliverables** — formal, contracted outputs (e.g. a signed-off Architecture Definition
    Document).
  - **Artifacts** — the diagrams, models, matrices, and catalogs that make up a deliverable.
  - **Building Blocks** — reusable components. **Architecture Building Blocks (ABBs)** are
    abstract/logical (e.g. "a patient record capability"); **Solution Building Blocks (SBBs)**
    are their concrete/physical implementations (e.g. "the Utano Patients module").
- **Architecture Capability Framework** — guidance on the organizational structures, skills,
  roles, and governance needed to actually run an EA practice on an ongoing basis, not just
  produce one set of documents once.
- **TOGAF Reference Models** — foundational reference architectures (e.g. the Technical
  Reference Model, the Integrated Information Infrastructure Model) that can be tailored rather
  than started from a blank page.
- **Architecture Repository** — the actual store (conceptually — could be a wiki, a tool, a
  folder structure) holding everything the ADM produces: the Architecture Landscape (what
  exists), reference material, standards, and governance logs. This is where Requirements
  Management (below) actually lives.

---

## The ADM Cycle — phase by phase

The ADM is a loop, not a one-off waterfall — each phase has real inputs and outputs, and Phase H
deliberately feeds back into Phase A.

### Preliminary Phase
- **Focus**: establish the EA capability itself, before touching any specific architecture.
- **Inputs**: organizational context, existing business strategy/principles.
- **Outputs**: tailored ADM (adapted to the organization), Architecture Principles, governance
  framework, scope of the organizations to be affected.

### Phase A — Architecture Vision
- **Focus**: define scope, identify stakeholders, create a high-level vision, secure sponsorship.
- **Inputs**: a request for architecture work, business goals/drivers, the tailored ADM from
  Preliminary.
- **Outputs**: an Approved Statement of Architecture Work, the Architecture Vision document,
  a stakeholder map.

### Phase B — Business Architecture
- **Focus**: model the current (baseline) and target business architecture.
- **Inputs**: Architecture Vision.
- **Outputs**: baseline + target Business Architecture, a business capability map, gap analysis
  between the two.

### Phase C — Information Systems Architecture
- **Focus**: split into **Data Architecture** and **Application Architecture** — both get a
  baseline, a target, and a gap analysis.
- **Inputs**: Business Architecture from Phase B.
- **Outputs**: baseline + target Data Architecture, baseline + target Application Architecture,
  gap analyses for both.

### Phase D — Technology Architecture
- **Focus**: the infrastructure/technology needed to support the Business, Data, and Application
  architectures above.
- **Inputs**: outputs of Phases B and C.
- **Outputs**: baseline + target Technology Architecture, gap analysis.

### Phase E — Opportunities & Solutions
- **Focus**: consolidate the gap analyses from B/C/D, identify delivery vehicles (projects,
  programs), do initial implementation planning.
- **Inputs**: all gap analyses so far.
- **Outputs**: a consolidated list of work packages, an initial Implementation and Migration
  Plan, Transition Architectures if the move to target state needs to happen in stages.

### Phase F — Migration Planning
- **Focus**: prioritize the work packages from Phase E, factoring in cost/risk/benefit, and turn
  them into a real roadmap.
- **Inputs**: work packages + Transition Architectures from Phase E.
- **Outputs**: a finalized, prioritized, costed Implementation and Migration Plan.

### Phase G — Implementation Governance
- **Focus**: oversee the actual implementation projects to ensure what gets built stays
  conformant with the architecture.
- **Inputs**: the Migration Plan.
- **Outputs**: Architecture Contracts (formal agreements that a project will comply), compliance
  assessments against those contracts.

### Phase H — Architecture Change Management
- **Focus**: monitor for changes (business, technology, regulatory) that require the
  architecture itself to be revisited.
- **Inputs**: everything currently in the Architecture Repository + external change signals.
- **Outputs**: change requests, a decision on whether a new ADM cycle needs to start —
  **feeds directly back into Phase A**, closing the loop.

### Requirements Management (the center of the cycle)
Not a numbered phase — sits in the middle, because requirements are gathered, validated, and
updated continuously across *every* other phase, not captured once at the start. All requirements
live in the Architecture Repository and are referenced by every phase as they run.

**Key idea to internalize**: the ADM is explicitly a **cycle**, not a project plan you run once.
Phase H feeding back into Phase A is the whole point — EA is continuous governance of change,
not a one-time deliverable that gets shelved after go-live.

---

## Certification Path

Structured as a single combined exam split into two levels, taken back-to-back or separately:

| Level | Tests | Format |
|---|---|---|
| **Foundation** | Knowledge/recall of TOGAF's concepts, terminology, and structure — the ADM phases, Enterprise Continuum, Content Framework, etc. | Closed-book, multiple choice |
| **Practitioner** | Application — analyzing scenarios and applying TOGAF concepts to them, not just naming them | Open-book, scenario-based multiple choice |

Both are administered by The Open Group (via Pearson VUE). Training isn't mandatory — self-study
against the official TOGAF 10 standard plus a reputable prep course/question bank is a common,
accepted path. **Foundation alone is a reasonable first target**: it's what most job specs
actually check for, and it's the natural entry point before deciding whether to push on to
Practitioner.

---

## Why Learn TOGAF First

Given the goal of becoming a Solutions Architect, TOGAF is the more directly practical of the two
frameworks in this plan to start with — it's the one most enterprise job specs actually name, it
has a formal and widely recognized certification path, and its ADM process maps naturally onto
real project delivery work already being done day to day.

---

## Study Plan / Resources

Fill in as study actually starts:

- [ ] Read the official TOGAF 10 standard (free on The Open Group's site) end to end once, no
      notes — just get the shape of it.
- [ ] Re-read ADM phase by phase, taking notes against a real reference case (Utano is a natural
      one, since it's already deeply familiar) — for each phase, actually write down what its
      inputs/outputs would look like for Utano specifically.
- [ ] Pick a Foundation prep course/question bank and work through it.
- [ ] Sit TOGAF Foundation.
- [ ] Decide on Practitioner based on how Foundation went and what real opportunities need it.
