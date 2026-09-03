# Tools to Learn - EA Modeling and Diagramming

See `README.md` for the "EA before frameworks" context and `TOGAF/README.md` for the ADM breakdown this tooling supports. Grouped by what each is actually for, since "EA tooling" and "diagramming tooling" are related but not the same skill.

---

## EA-specific modeling/repository tools

| Tool | Why it matters |
|---|---|
| **Sparx Systems Enterprise Architect** | The single most commonly named tool in actual EA job specs. Supports UML, BPMN, and ArchiMate, with built-in TOGAF ADM and Zachman templates. Worth knowing at a basic level purely because it's named so often as a job requirement. |
| **Archi** | Free, open-source, ArchiMate-only. The natural first tool to practice ArchiMate notation hands-on without a license cost - good fit for personal study. |
| **BiZZdesign Enterprise Studio** | A leading commercial EA repository tool, ArchiMate-native, common in larger/regulated enterprises. |
| **LeanIX** / **Ardoq** | Modern SaaS alternatives to the legacy heavyweight tools above - lighter-weight application portfolio management and EA, increasingly what organizations are migrating toward. Worth knowing these exist even without going deep on either yet. |
| **MEGA International (HOPEX)** | Enterprise-grade EA/GRC tool, common in large regulated financial-services orgs - relevant given Apex/SonataOne's compliance-heavy domain. |

## Notation to learn (matters more than any single tool)

- **ArchiMate** - the de facto standard EA notation, maintained by The Open Group (same body as TOGAF), purpose-built to model Business/Application/Technology layers together. Highest-value item on this whole list - most of the tools above are built around it, and it pairs directly with the TOGAF study already underway.
- **BPMN** - for Business Architecture process flows (Phase B-level work).
- **UML** - still relevant for logical/System Model-level detail (Phase C).
- **C4 model** (Simon Brown) - a lighter-weight alternative to full ArchiMate for Application/Technology diagrams specifically (Context/Container/Component/Code levels). Diagram-as-code, so it fits naturally alongside the mermaid habit already built up in the applied initiative rather than replacing it.

## General diagramming tools

| Tool | Notes |
|---|---|
| **draw.io / diagrams.net** | Free, has ArchiMate/UML/BPMN/C4 shape libraries, most transferable day-to-day skill, integrates with Confluence. Probably the single most practically useful tool to get fluent in. |
| **Visio** | Extremely common in enterprise environments already on Microsoft 365; has ArchiMate/TOGAF stencils available. |
| **Lucidchart** | Commercial, cloud-based, similar space to draw.io - common enough to be worth recognizing even if draw.io is the daily driver. |
| **Structurizr / PlantUML** | Diagram-as-code tools implementing C4 - the direct "grown-up" version of what mermaid already does in the applied initiative. |

## Honest caveat, given the applied initiative is personal practice, not a chartered EA function

Mermaid-in-markdown (what `Apex-SonataOne-Initiative/` already uses throughout) is a genuinely good choice for *that* work - version-controlled, diffable, zero tool cost - but it is not what a real EA job will expect day to day. If the goal is employability rather than just this initiative's own documentation, ArchiMate notation plus fluency in one real tool (Archi to start, Sparx EA if targeting the tool most often named in job specs) is the highest-leverage pair to actually learn.
