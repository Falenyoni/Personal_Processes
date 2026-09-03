# System Mental Model

Diagram-first companion to `../IDR-Rebuild/` - built for someone new to the system to get a working mental model fast, without reading 13 dense investigative documents first. Every diagram here is reproduced faithfully from that folder; this folder adds sequence and near-zero extra text. For evidence, file/line citations, and the full reasoning behind any diagram, follow the link at the top of each document back to its `../IDR-Rebuild/` source.

## Reading order

| Doc | Answers |
|---|---|
| [`01-Diagrams-And-Flows.md`](./01-Diagrams-And-Flows.md) | The 10-minute version - landscape, core domain model, the happy-path flows, and where the platform is headed. Start here. |
| [`02-Monolith-Today.md`](./02-Monolith-Today.md) | What's in IDR (the monolith) and how it happens - the three journeys, who can be a KYC subject, IKYC/MKYC as IDR runs them, the permission model. |
| [`03-Rebuild-Today.md`](./03-Rebuild-Today.md) | What's in the Rebuild (TemplateAPI + ManagedServices) and how it happens - today's actual scope, Buy Button, Profile/Visa/answer flows, question merging, and two known live defects. |
| [`04-Sync-And-Known-Defects.md`](./04-Sync-And-Known-Defects.md) | How IDR and the Rebuild actually talk to each other today, the three-repo pipeline most mental models miss (`SyncExchange`), and the confirmed sync bugs. |
| [`05-Target-Architecture.md`](./05-Target-Architecture.md) | The missing pieces, part 1 - what a clean-slate design looks like, and where the actual build already agrees with it. |
| [`06-Migration-Roadmap-And-Fixes.md`](./06-Migration-Roadmap-And-Fixes.md) | The missing pieces, part 2 - schema mapping, migration phases, team ownership, and the concrete fixes designed for every defect in doc 04. |

## Working convention

Diagrams first, minimal text - one or two lines of caption per diagram, no re-derivation of evidence already established in `../IDR-Rebuild/`. If a claim needs defending, it belongs in that folder, not here.
