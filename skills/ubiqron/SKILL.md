---
name: ubiqron
description: Strategic and tactical Domain-Driven Design for codebases built with Clean Architecture and Hexagonal ports and adapters, in any language; stack details such as project names and test framework come from stack profiles (.NET first). Use when the user mentions "bounded context", "aggregate", "aggregate root", "domain service", "domain event", "invariant", "ubiquitous language", "glossary", "context map", "domain history", "DOMAIN-HISTORY.jsonl", or asks to create a new bounded context, review a context against DDD rules, move an aggregate between contexts, or record a domain change. Also use when scaffolding a context's Domain module and its test module in a monorepo. Keeps the glossary, the temporal ledger and the context map inside the codebase.
---

# ubiqron: Domain-Driven Design with the model kept in the codebase

ubiqron helps one developer model, build and evolve bounded contexts in a monorepo, in any language.
Stack-specific details such as module names, test framework and dependency checks come from a stack profile.
It keeps three facts in the codebase: what each term means (glossary), how the model changed over time (ledger), and how contexts relate (context map).
Every modeling action updates these files. Code and model never drift apart.

## How to talk to the developer

- Write plain English. Short sentences. One idea per sentence. Active voice.
- The only technical words are DDD words: aggregate, invariant, bounded context, domain event. Use everyday words for everything else.
- No idioms. No filler words such as robust, leverage, seamlessly, comprehensive.
- Answer with lists and tables. A paragraph has at most three sentences.
- One phase per reply. Make a proposal, then stop and wait for the developer's choice.
- Budgets: at most 7 aggregate candidates, 5 invariants per aggregate, 3 context relationships, 10 review findings per reply.
- Never write or change files before the developer selects. Then write only what was selected.
- Ledger `reason` fields are one sentence. Glossary definitions are at most two sentences.

## Modes

| Mode | When | Read |
|------|------|------|
| New context | "Create a bounded context", "model the X context", "we need a new module for X" | [references/new-context-flow.md](references/new-context-flow.md) |
| Review | "Review", "check", "audit" an existing context against the rules | [references/review-mode.md](references/review-mode.md) |
| Maintain | Rename or deprecate a term, add or remove an invariant, add a method or event, move an aggregate, change a context relationship, answer "what did X mean on date D" | [references/glossary-and-ledger.md](references/glossary-and-ledger.md), [references/context-map.md](references/context-map.md) |
| Draw | "Draw", "diagram", "sequence for X", "states of X", "show the history of X", "context map" | [references/diagrams.md](references/diagrams.md) |

Pick the mode from the request. If the request fits none, ask one question to choose.

## Layout and stack profile

One bounded context is one folder under `src/`, with one module per layer:

| Module | Holds |
|--------|-------|
| Domain | aggregates, entities, value objects, domain services, domain events, domain exceptions, factories |
| Application | command handlers, query handlers, ports, processes |
| Infrastructure | adapters: persistence, messaging, external systems |
| Api | entry points: HTTP, messaging |
| Contract | published language: integration events and their small DTO and ID types (default home) |

At repo level: an optional `SharedKernel` module with value objects only, one Domain test module created together with each Domain module, and `ubiqron/CONTEXT-MAP.md`.

Exact folder names, file extensions, the test framework, scaffold commands and the way layer boundaries are checked come from the stack profile. Detect the stack from the repo before phase 1 and read the matching profile:

| Marker in the repo | Profile |
|--------------------|---------|
| `*.csproj`, `*.slnx` or `*.sln` | [references/stacks/dotnet.md](references/stacks/dotnet.md) |
| Anything else | [references/stacks/generic.md](references/stacks/generic.md); it reads the code first and asks the developer two questions once |

Each context also owns two files inside its Domain module:

| File | Purpose |
|------|---------|
| `<Context>.Domain/UBIQUITOUS-LANGUAGE.md` | Current meaning of every term, aggregate, method and event in this context |
| `<Context>.Domain/DOMAIN-HISTORY.jsonl` | Append-only history of every model change in this context |


## Rules

These rules are fixed. Review mode reports a finding for each violation and cites the rule id.

| Id | Rule |
|----|------|
| U1 | Dependencies point inward. Domain references no other module except SharedKernel. Application references Domain. Infrastructure and Api reference Application. Contract references nothing except SharedKernel. |
| U2 | Domain holds aggregates, entities, value objects, domain events, domain services, domain exceptions and factories. |
| U3 | No port interfaces in Domain. Domain-internal interfaces are fine: an interface for a domain service, marker interfaces for domain events, event-holding aggregates, versioning or auditing. Any interface that an outer layer implements for I/O is a port and belongs in Application. An Elasticsearch or Postgres port in Domain is a violation. |
| U4 | A repository is a port in Application: `Application/<Aggregate>/Ports/`, named after the aggregate, one per aggregate root, implemented in Infrastructure. Its method set is the developer's choice. |
| U5 | A domain service takes no ports. Its inputs are aggregates and value objects that the handler already loaded. It returns a result or throws a domain exception. |
| U6 | A domain service coordinates aggregates inside one context. Default: it changes one aggregate and reads the others. Exception: when one invariant spans two roots and must hold at once, the service may change both. The handler then saves both in one unit of work. Same context only. The invariant is named in the ledger. |
| U7 | Domain exceptions are typed. One exception class per business refusal, such as `MerchantUnavailableException`, usually derived from a common `DomainException`. In a language without exceptions, one error type per refusal. No generic exception with an error code. |
| U8 | Application has command handlers and query handlers. A query handler may inject the repository port. No dispatcher library is required. |
| U9 | A context references another context only through that context's published-language module, by default `Contract`. Never its Domain, Application, Infrastructure or Api module. |
| U10 | The published language holds integration events and the small DTO and ID types they carry. No aggregates, no entities, no behaviour. A domain event class holds domain data only; envelope fields such as eventId or occurredAt are added at the boundary. |
| U11 | Shared Kernel is a separate small module with value objects only. Contract modules may reference it. |
| U12 | Sagas and process managers live in Application, default folder `Application/Processes/`. They may use ports. They are not domain services. |
| U13 | Every modeling action appends a ledger line. The glossary and the context map are projections of the ledger. |
| U14 | Code does not use an item that the ledger marks deprecated. |
| U15 | A stored diagram ends with the last ledger id it reflects. A diagram older than a later ledger line about its subject is reported as stale. Diagrams are projections; drawing one writes no ledger line. |

Tactical detail: [references/tactical-patterns.md](references/tactical-patterns.md). Application detail: [references/application-layer.md](references/application-layer.md). Strategic detail: [references/strategic-design.md](references/strategic-design.md).

## Developer choices

These are the developer's call. Ask once, record the answer as a `DecisionRecorded` ledger line, do not ask again, never report them as findings.

| Choice | Options | When to ask |
|--------|---------|-------------|
| Reconstitution from the database | The ORM writes fields through a hidden constructor, or an explicit `Reconstitute(...)` method | Only when the Infrastructure module is scaffolded |
| Repository methods | Collection style (`Add`, `Remove`, change tracking) or persistence style (`Update`, `Save`); query methods on the same port; unit of work or the ORM's save call | When the first repository port is written |
| Command and query dispatch | Own code or any library | When the Application module is scaffolded |
| Event envelope and outbox | Own code or a messaging library; options per stack profile | When the first integration event is published |
| Home of the published language | The `Contract` module (default) or another module | When the first integration event is created |
| Current time in the domain | Passed in from the handler, a default time zone in Domain, or a shared helper in Domain | Never ask; accept what the code does |
| Specification pattern | The developer's own approach, if they use it | Never ask |
| Subdomain type | Core, supporting or generic, chosen with three short questions | Phase 1 of a new context. The type never sets how deep the model goes. |

## The ledger in one screen

`<Context>.Domain/DOMAIN-HISTORY.jsonl`. Append only. One JSON object per line. Never edit or delete a line; correct with a new line that has `supersedes`.

Fields: `id, at, context, kind, subject, from?, to?, reason, ref?, supersedes?, related?`

Kinds: `TermIntroduced, TermRenamed, TermDeprecated, AggregateIntroduced, AggregateDeprecated, InvariantAdded, InvariantRemoved, ResponsibilityMoved, MethodIntroduced, MethodDeprecated, EventIntroduced, ContextRelationshipChanged, DecisionRecorded`

```jsonl
{"id":"ord-0003","at":"2026-11-06","context":"Order","kind":"InvariantAdded","subject":"OrderDomainService.ProcessOrder","to":"An order is rejected when the merchant is unavailable","reason":"Busy merchants can stop taking new orders"}
```

Two-context operations write one line in each ledger, linked by `related` (`<Context>:<id>`). Moving `Shipment` from Order to Fulfillment is `AggregateDeprecated` in Order with `to: "Fulfillment"` and `AggregateIntroduced` in Fulfillment with `from: "Order"`. Compute both ids first, then write both lines.

Full schema and the glossary format: [references/glossary-and-ledger.md](references/glossary-and-ledger.md).

## Canonical example

Use this example in every explanation unless the developer's own domain is at hand.

A food-ordering platform with many merchants.

- **Merchant context.** `Merchant` is the business record: address, VAT number, bank details, location.
- **Order context.** `Merchant` is the business that fulfils an order: identity and availability, nothing more. A busy merchant sets itself `Unavailable`.
- The two `Merchant` classes are separate. They never share code.
- Merchant context publishes `MerchantRegistered` as an integration event in its Contract module. Order context consumes it; a translator in Order creates Order's own `Merchant`.
- `OrderDomainService.ProcessOrder(merchant, order)` throws `MerchantUnavailableException` when the merchant is unavailable. The rule lives once in the domain service, even if several handlers create orders.
- Strong-consistency case: `PurchaseInvoice.Create(...)` followed by `PurchaseOrder.InvoiceLine(...)` in one domain service, both saved in one unit of work, because the invoiced quantity must be on the purchase order at once.

## Not part of ubiqron

- Team structure. A bounded context is owned by one team or one developer. ubiqron does not plan teams or map boundaries to teams.
- Scores. Review mode returns findings with rule ids, not points.

## References

| File | Content |
|------|---------|
| [references/new-context-flow.md](references/new-context-flow.md) | Four phases: scope, aggregates, behaviour and tests, scaffold; one optional state diagram in phase 3 |
| [references/review-mode.md](references/review-mode.md) | What to check for U1 to U15, cross-context checks, finding format |
| [references/strategic-design.md](references/strategic-design.md) | Bounded contexts, subdomain questions, the nine relationship patterns, published language |
| [references/context-map.md](references/context-map.md) | `ubiqron/CONTEXT-MAP.md` template, pattern defaults, ledger link |
| [references/tactical-patterns.md](references/tactical-patterns.md) | Aggregates, entities, value objects, domain services, domain events, domain exceptions, factories |
| [references/application-layer.md](references/application-layer.md) | Command and query handlers, ports, processes, developer choices |
| [references/stacks/dotnet.md](references/stacks/dotnet.md) | .NET profile: project tree, csproj references, xUnit, scaffold commands, EF and library options |
| [references/stacks/generic.md](references/stacks/generic.md) | Fallback profile for any other stack, plus the template for new profiles |
| [references/glossary-and-ledger.md](references/glossary-and-ledger.md) | `UBIQUITOUS-LANGUAGE.md` format, ledger schema, kinds, dual-write, validation test |
| [references/diagrams.md](references/diagrams.md) | Six Mermaid diagram kinds, file layout, budgets, stale check, preview page |
