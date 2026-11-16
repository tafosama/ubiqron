# Diagram samples

Six Mermaid diagram kinds used by ubiqron, drawn on the food-ordering example. Labels are English, as they would be in a real repo.

| File | Kind | Lives in a real repo at |
|------|------|-------------------------|
| `CONTEXT-MAP.md` | `graph LR` | `ubiqron/CONTEXT-MAP.md`, section C |
| `PlaceOrder.sequence.md` | `sequenceDiagram` | `<Context>.Domain/Diagrams/` |
| `Order.state.md` | `stateDiagram-v2` | `<Context>.Domain/Diagrams/` |
| `OrderPaymentProcess.flow.md` | `flowchart TD` | `<Context>.Domain/Diagrams/` |
| `Order.overview.md` | `classDiagram` | `<Context>.Domain/Diagrams/` |
| `Order.timeline.md` | `timeline` | Not stored; generated on request from the ledger |

Rules shown here: one diagram per file, a `Ledger:` line at the end that names the last ledger id the diagram reflects, at most 15 nodes, labels taken from the glossary.

How to view: GitHub and GitLab render the fences. JetBrains Rider renders them in Markdown preview. VS Code needs the "Markdown Preview Mermaid Support" extension. For SVG: `npx -p @mermaid-js/mermaid-cli mmdc -i Order.state.md -o Order.state.svg`.

Quick look without an IDE plugin: open `preview.html` in a browser. It loads mermaid.js from a CDN and renders all six files on one page.
