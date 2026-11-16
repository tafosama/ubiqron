# Review mode

Review one bounded context against rules U1 to U15. The result is a list of findings, not a score. The developer picks which findings to fix. Nothing is changed before that.

## Steps

1. Detect the stack and read the matching profile in `references/stacks/`. It says how to list module dependencies and how to find interfaces, exceptions and handlers in this stack.
2. Read `ubiqron/CONTEXT-MAP.md`, the context's `DOMAIN-HISTORY.jsonl`, its `UBIQUITOUS-LANGUAGE.md` and its `Diagrams/` folder.
3. Run the checks below. Stop collecting after 10 findings; count the rest.
4. Reply with the findings table. Wait for the developer to pick numbers.
5. Fix only the picked findings. A fix that changes the model writes a ledger line. A fix that only moves code writes none.

When the developer asks for the whole repo, review one context per reply, in context-map order.

## Checks

| Rule | Look at | Finding when |
|------|---------|--------------|
| U1 | Module dependencies, listed with the profile's boundary check | Domain depends on anything but SharedKernel; Contract depends on anything but SharedKernel; an inner module references an outer one |
| U2 | Files in Domain | A file in Domain does HTTP, database, messaging, file or framework work |
| U3 | Interfaces in Domain | An interface in Domain has an implementation in Application, Infrastructure or Api |
| U4 | Repository interfaces | A repository interface is outside `Application/<Aggregate>/Ports/`; two repositories for one root; a repository for an entity that is not a root |
| U5 | Domain service constructors and method parameters | A parameter or field whose type is a port, repository, publisher, client or gateway |
| U6 | Domain services that change state | A service changes two roots and the ledger has no `InvariantAdded` line naming the rule; a service touches an aggregate of another context |
| U7 | Throw statements in Domain | A general-purpose exception; an exception carrying an error code or a string code instead of a typed class |
| U8 | Application module | A command or query handled outside Application; a handler that calls Infrastructure code directly instead of a port |
| U9 | References between contexts | A module references another context's Domain, Application, Infrastructure or Api |
| U10 | Contract module; domain event classes | Contract holds an aggregate, an entity or a method with behaviour; a domain event class has envelope fields such as eventId, occurredAt or correlationId |
| U11 | SharedKernel module | It holds an entity, an aggregate, a service, a port or a handler |
| U12 | Sagas and process managers | One lives in Domain, or is written as a domain service |
| U13 | Code names against the glossary and the ledger | An aggregate, method, event or exception in code has no glossary row; a glossary row has no ledger line; a command, query, event handler or process in Application has no row in the use-case table; a use-case row has no code; the glossary's last ledger id is older than the ledger's last line |
| U14 | Deprecated subjects from the ledger | A deprecated aggregate, method, term or event is still referenced in code; the code of a deprecated use case still exists |
| U15 | `Diagrams/*.md` | A diagram's `Ledger:` id is older than a ledger line about its subject (procedure in `references/diagrams.md`) |

Use cases missing from the table are one finding per context, not one per handler: the count and the first three names. The fix writes one `UseCaseIntroduced` line per missing use case and rebuilds the table. This is how an existing codebase gets its first use-case table. The stack profile says how to list handlers.

Two checks cross the context boundary:

| Check | Finding when |
|-------|--------------|
| `related` links | A `related` value points to an id that does not exist in the other context's ledger |
| Context map against code | A relationship row has no matching module reference in code; a module reference to another context has no relationship row |

## Not checked

These are developer choices (SKILL.md). Never report them.

- Reconstitution style. Check only that no code path except persistence loading can build an aggregate while skipping its invariants.
- Repository method names and count.
- Specification pattern, present or absent.
- Envelope shape and outbox implementation.
- How the domain gets the current time.
- Dispatcher library, present or absent.
- Folder names that differ from the defaults, when a `DecisionRecorded` line records the choice.

## Finding format

One row per finding. Order: boundaries first (U1, U9, U11), then Domain purity (U2, U3, U5, U7, U10), then structure (U4, U6, U8, U12), then model files (U13, U14, U15, cross checks).

```
Review of Order context: 5 findings.

| # | Rule | Where | What | Fix |
|---|------|-------|------|-----|
| 1 | U9 | Order.Application module reference | References Merchant.Domain | Reference Merchant.Contract and translate MerchantRegistered |
| 2 | U3 | Order.Domain/Services/OrderSearchPort | Interface implemented in Order.Infrastructure | Move it to Order.Application/Order/Ports/ |
| 3 | U7 | Order.Domain/Order/Order, line 41 | Throws a general exception with code "ORD_EMPTY" | Add EmptyOrderException |
| 4 | U13 | Order.Application | 6 use cases have no row: CancelOrder, GetOrderById, RefundOrder and 3 more | Write 6 UseCaseIntroduced lines and rebuild the use-case table |
| 5 | U15 | Order.Domain/Diagrams/Order.state.md | Ledger: ord-0017; newer lines ord-0019, ord-0021 | Redraw the state diagram |

Reply with the numbers to fix.
```

- Where: a file path, plus a line number when the stack profile gives one.
- What: one sentence.
- Fix: one sentence, no code.
- More than 10 findings: last line "6 more findings. Say next to see them."
- No findings: "Review of Order context: no findings." plus one line with what was checked: modules, interfaces, throw statements, use cases, ledger lines, diagrams.

## After the selection

1. Make the picked fixes, one commit-sized change per finding.
2. Write a ledger line for each fix that changes the model: a new glossary row is `TermIntroduced`, `AggregateIntroduced`, `MethodIntroduced` or `EventIntroduced`; a new use-case row is `UseCaseIntroduced`; a removed deprecated use writes nothing; a moved port writes nothing.
3. Rebuild the glossary rows that changed. Redraw a picked stale diagram and update its `Ledger:` line.
4. Run the build or the tests once, as the stack profile says.
5. Reply with a table: finding number, file changed, ledger id or "no ledger line", and one line with the build result.
