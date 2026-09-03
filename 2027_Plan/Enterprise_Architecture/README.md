# Enterprise Architecture

Notes and resources for Enterprise Architecture (EA), focused on the two major frameworks:

- TOGAF
- Zachman Framework

---

## Enterprise Architecture — Before the Frameworks

Before learning any specific framework, it's worth being clear on what EA actually *is* and why
it exists — the frameworks are just tools for doing this, not the thing itself.

### What EA is

Enterprise Architecture is the practice of translating an organization's strategy and business
goals into a structured blueprint that guides how the business, its data, its applications, and
its technology all fit together — and, critically, how they change over time without descending
into chaos.

It sits above individual project/system architecture. A solutions architect designs *one*
system; an enterprise architect makes sure *all* the systems, across the whole organization, are
coherent, aligned to strategy, and not quietly duplicating or contradicting each other.

### The problem EA solves

Without EA, organizations tend to accumulate:

- **Silos** — each department/team builds what it needs in isolation, with no shared view of the
  whole.
- **Duplication** — the same capability (e.g. "customer record") built or bought multiple times
  in incompatible ways.
- **Technical debt at the organizational level** — not just messy code in one system, but
  years of disconnected, hard-to-change decisions across *many* systems.
- **Strategy-execution gap** — leadership sets a direction, but nothing structurally connects
  that direction to what IT/engineering actually builds.

EA exists to give leadership and delivery teams a **shared, structured view** that closes that
gap — so technology investment is traceable back to business goals, and changes in one part of
the organization can be reasoned about in terms of their impact elsewhere.

### The four commonly recognized architecture domains

Almost every EA framework (TOGAF included) organizes the work around some version of these four
layers:

1. **Business Architecture** — strategy, governance, organization structure, key business
   processes and capabilities. The "why" and "what" of the business itself.
2. **Data/Information Architecture** — the structure of an organization's logical and physical
   data assets, and how data is managed, shared, and governed across the enterprise.
3. **Application Architecture** — the individual applications/systems, their interactions, and
   their relationship to the core business processes.
4. **Technology Architecture** — the hardware, software, network, and infrastructure needed to
   support the above three layers (sometimes called Infrastructure Architecture).

A useful mental model: **Business Architecture defines the destination, the other three layers
describe the vehicle.**

### What a framework actually gives you

A framework (TOGAF, Zachman, or otherwise) doesn't do the thinking for you — it gives you:

- A **common vocabulary**, so "architecture" means the same thing across a whole organization.
- A **structured way to organize artifacts** (diagrams, models, decisions) so nothing important
  falls through the cracks.
- A **repeatable process or classification scheme**, so EA work isn't reinvented ad hoc every
  time.

This distinction matters going in: **TOGAF is a *methodology*** (a process for *doing* EA
end-to-end), while **Zachman is a *taxonomy*** (a classification scheme for *organizing*
architecture artifacts) — they answer different questions and are often used together rather
than as competing choices.

---

## TOGAF (The Open Group Architecture Framework)

Broken down in full in its own folder: **[`TOGAF/README.md`](./TOGAF/)** — core components, the
ADM cycle phase by phase (inputs/outputs for each), certification path, and a study plan.

Short version: TOGAF is the most widely adopted EA **methodology** — an end-to-end, iterative
process (the ADM) for developing, governing, and maintaining an enterprise architecture. It's the
more directly practical of the two frameworks in this plan to start with, since it's what most
enterprise job specs actually name and it has a formal certification path.

---

## The Zachman Framework

### What it is

Created by John Zachman (originally at IBM, 1987) — the **oldest** EA framework, and
fundamentally different in kind from TOGAF: it is **not a process** for producing an
architecture. It's a **classification schema (taxonomy)** for organizing the artifacts that
describe an enterprise, ensuring nothing important is ever left undocumented or lost. Often
described by analogy to an architect's set of building plans — floor plan, electrical plan,
plumbing plan, etc. — each a different, necessary view of the *same* building.

### The structure — a 6×6 matrix

Zachman is a grid of **6 questions (columns) × 6 perspectives (rows)** = 36 cells, each
representing a distinct, necessary view of the enterprise. No single cell is "the architecture"
— the value is in having *all 36* views available and consistent with each other.

**The 6 interrogatives (columns) — the "what" of each view:**

| Question | Focus |
|---|---|
| **What** | Data — the things/entities the enterprise cares about |
| **How** | Function/Process — how work actually gets done |
| **Where** | Network/Location — geographic and logical distribution |
| **Who** | People/Organization — roles and responsibilities |
| **When** | Time — events, cycles, scheduling |
| **Why** | Motivation — goals, strategy, drivers behind everything else |

**The 6 perspectives (rows) — "whose" view it is, from most abstract to most concrete:**

| Row | Perspective |
|---|---|
| **Scope (Contextual)** | Planner's view — the enterprise from the outside, boundary/scope only |
| **Business Model (Conceptual)** | Owner's view — business concepts and relationships, no tech yet |
| **System Model (Logical)** | Designer's view — logical models, still technology-agnostic |
| **Technology Model (Physical)** | Builder's view — how it's actually implemented with real technology constraints |
| **Detailed Representations** | Subcontractor's view — out-of-context detail, ready for construction (e.g. actual code, configs) |
| **Functioning Enterprise** | The real, operating instance of the enterprise itself — the actual running thing, not a model of it |

### Why this matters in practice

- Zachman forces completeness: it's a checklist that surfaces *what's missing* (e.g. "we have
  no documented answer for **Why** at the **Logical** level") rather than telling you *how* to
  go build it.
- It's **descriptive, not prescriptive** — no phases, no sequence, no "start here." You can
  populate cells in any order, and different roles naturally focus on different rows.
- It pairs naturally with TOGAF: **TOGAF tells you the process for developing architecture
  content; Zachman gives you a way to check that content is complete and consistently
  classified.** They aren't rival choices — many real EA practices use TOGAF's ADM to *do* the
  work and a Zachman-style matrix to *audit* coverage of what's been produced.

### Zachman vs. TOGAF — the one-line distinction to remember

| | TOGAF | Zachman |
|---|---|---|
| **Type** | Methodology (a process) | Taxonomy (a classification scheme) |
| **Answers** | "How do we develop and govern the architecture?" | "Have we captured every necessary view of the enterprise?" |
| **Structure** | Iterative phase cycle (ADM) | Static 6×6 matrix |
| **Certification** | Formal, widely recognized (Foundation/Practitioner) | Less formalized, less commonly a hard job requirement |

---

## Certification Paths

### TOGAF

Full detail (exam levels, format, study plan) now lives in **[`TOGAF/README.md`](./TOGAF/)**.
Short version: two levels, Foundation (recall) then Practitioner (application), both via The
Open Group/Pearson VUE — Foundation alone is a reasonable first target.

### Zachman

Certification exists (**Zachman International** runs it) but is far less standardized and far
less commonly demanded in job specs than TOGAF's. Structured as a small set of course-based exams
(currently badged as Associate/Practitioner/Professional-type tiers) run through Zachman
International directly rather than a body like Pearson VUE.

**Practical call for this plan**: treat Zachman as a *concept to understand deeply and apply*
(the 6×6 matrix, the discipline of checking completeness) rather than a certification to chase —
the payoff is in how it sharpens TOGAF work, not in the certificate itself. Revisit formal
certification only if a specific opportunity later makes it worth it.

### Suggested order

1. Learn TOGAF's ADM properly (`TOGAF/README.md`'s breakdown + the official TOGAF 10 standard
   docs, freely readable on The Open Group's site).
2. Sit **TOGAF Foundation**.
3. Apply the Zachman matrix informally against a real system (Utano is a good candidate - it's
   already deeply familiar) to internalize the classification habit before deciding whether
   Practitioner-level TOGAF or Zachman certification is worth pursuing next.
4. Once comfortable with the concepts, start learning the tooling and notation actually used to
   produce EA artifacts day to day - see **[`Tooling.md`](./Tooling.md)** for the specific tools
   and notation (ArchiMate, Archi, Sparx EA, draw.io, C4) worth learning, and why each earns its
   place.

---

## Applying This - the Apex/SonataOne Initiative

The "apply it against a real system" step below has started, at real depth: **[`Apex-SonataOne-Initiative/`](./Apex-SonataOne-Initiative/)** - the ADM cycle and a Zachman matrix walked against the actual Investor Services/KYC platform (`IDR`, `TemplateAPI`, `ManagedServices`, `Sync`/`SyncExchange`), grounded in the extensive investigation now tracked at [`../../01_Planninng/IDR-Rebuild/`](../../01_Planninng/IDR-Rebuild/). Scoped to Apex/SonataOne only.

## Notes

Add reasoning, sequencing decisions, and resource links here as study progresses — e.g. which
TOGAF certification track to pursue, notes from any Zachman matrix exercises done against a real
system, how this connects to the Solutions Architect track.
