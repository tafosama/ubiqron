# Diagrams

Diagrams are pictures of what the code, the ledger and the context map already say. They are never the source of truth. Drawing or updating one writes no ledger line. Samples for every kind are in `examples/diagrams/`.

## Kinds

| Kind | Mermaid | Shows | Drawn from | When |
|------|---------|-------|------------|------|
| Context map | `graph LR` | Contexts, direction, pattern code, mechanism | `ubiqron/CONTEXT-MAP.md` relationship table | Every time the map changes; it is section C of that file |
| Use case sequence | `sequenceDiagram` | Api, handler, ports, domain service, aggregate, event, outbox, consumer | Handler and domain service code | On request; offer it for a flow that crosses contexts |
| Aggregate state | `stateDiagram-v2` | States, the methods that move between them, the invariants that refuse a move | The aggregate's status field and methods | Offered once in phase 3; on request after a method or invariant changes |
| Process flow | `flowchart TD` | Saga steps, decisions, retries, compensation | `Application/Processes/` code | When a saga is written or changed |
| Context overview | `classDiagram` | Aggregates, entities, value objects, domain service, events, id references | The Domain module | On request |
| History timeline | `timeline` | Ledger lines in date order | `DOMAIN-HISTORY.jsonl` | On "show the history of X"; shown, not stored |

## Files

```
<Context>.Domain/Diagrams/
  <Aggregate>.state.md
  <UseCase>.sequence.md
  <Process>.flow.md
  <Context>.overview.md
```

- One diagram per file. Markdown with one `mermaid` fence.
- Line 1: `# <Title>`. Then one sentence on what the diagram shows. Then the fence.
- Last line: `Ledger: <id>`, the newest ledger line the diagram reflects (U15). The context map carries ids in its table instead. The timeline is not stored.
- The context map diagram lives in `ubiqron/CONTEXT-MAP.md`, not in a Diagrams folder.

## Budgets

- At most 15 nodes per diagram, at most 8 participants in a sequence. Split a bigger flow into two files.
- Labels are glossary terms and method names. Plain English, no sentences inside nodes except invariant notes.
- No colours. Only two styles: dashed border for an external system, dotted arrows for Shared Kernel.

## How to draw each kind

**Context map.** One node per row of the contexts table; type in brackets. One arrow per row of the relationships table, upstream to downstream. Edge label: pattern code, colon, mechanism. Codes: CS Customer-Supplier, ACL Anti-Corruption Layer, OHS Open Host Service, PL Published Language, SK Shared Kernel, CF Conformist, P Partnership, SW Separate Ways, BBM Big Ball of Mud.

**Sequence.** Participants in call order: Api, the handler, each port it uses, the domain service, the aggregate. Ports keep their interface names. An `alt` block per typed exception the domain service or aggregate can throw. A `Note over <aggregate>` for each domain event it raises. For a cross-context flow, continue after the handler: outbox, publisher, the consuming context's translator, the aggregate it creates.

**State.** One state per value of the status field. One transition per method that changes it, labelled with the method name. A `note` beside a state for each invariant that refuses a transition there. A method with no transition from a state is refused in that state; do not draw refused moves.

**Flow.** Start and end as stadium nodes named after the event or outcome. A rectangle per command or call, a diamond per decision. Draw the compensation branch and the retry loop; both are part of the process. Name the port the saga uses.

**Overview.** One class per aggregate root, entity, value object, domain service and domain event in the context. Stereotype first: `<<AggregateRoot>>`, `<<Entity>>`, `<<ValueObject>>`, `<<DomainService>>`, `<<DomainEvent>>`. Composition arrow from root to its entities and value objects. A dashed arrow with the label `by <Id> only` between aggregates: they never hold each other.

**Timeline.** Group ledger lines by `at`. One entry per line: kind, subject, short `to` text, the id in brackets. Filter by subject when the developer names one. Show it in the reply; write a file only if the developer asks.

## Stale check (review mode, U15)

For each file in `Diagrams/`: read its `Ledger:` id and its subject (the aggregate, use case or process in the file name). Find ledger lines with a later id whose `subject` starts with that name. If any exist, report one finding: file, the newer ledger ids, and "diagram may be stale". Do not redraw unless the developer picks the finding.

## Viewing

- GitHub, GitLab and JetBrains IDEs render the fences.
- VS Code needs the "Markdown Preview Mermaid Support" extension. Cursor does not render Mermaid.
- When the developer's editor does not render, write `Diagrams/preview.html`: one page that embeds every diagram in a `<pre class="mermaid">` block and loads `mermaid.min.js` from a CDN, then open it with the system browser. Regenerate it when a diagram file changes. A sample is in `examples/diagrams/preview.html`.
- For an SVG file: `npx -p @mermaid-js/mermaid-cli mmdc -i <file>.md -o <file>.svg`.

## Phase 3 hook

After the developer selects methods and invariants for an aggregate, ask one yes-or-no question: "Draw the state diagram for <Aggregate>?" On yes, write `<Aggregate>.state.md` in phase 4 together with the Domain scaffold. No other diagram is offered during the new-context flow.
