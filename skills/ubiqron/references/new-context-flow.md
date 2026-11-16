# New context flow

Four phases. One phase per reply. Each reply ends with a proposal and a question; then stop and wait for the developer's choice. Nothing is written to disk before phase 4.

## Before phase 1

1. Detect the stack with the table in SKILL.md and read the matching profile in `references/stacks/`.
2. Read `ubiqron/CONTEXT-MAP.md` if it exists, to know the existing contexts and their glossaries.
3. Read every URL and document the developer gave. Keep notes for yourself; do not echo the content back.

How to read research inputs: collect the nouns the business uses (term candidates), the verbs that change state (method and event candidates) and the sentences with "must", "cannot", "only when", "always" (invariant candidates). Note the source name for the ledger `ref` field.

## Phase 1: Scope

Ask at most five questions in one message, each with a default the developer can accept by saying "ok".

| # | Question | Default |
|---|----------|---------|
| 1 | Context name, one or two words | Taken from the developer's description |
| 2 | One sentence: what this context is responsible for | The developer's own sentence, tightened |
| 3 | Subdomain type, from the three tests below | From the table below |
| 4 | Which existing contexts does it talk to, and in which direction? For each upstream: your own context or an external system? | Taken from the research inputs and the context map |
| 5 | Any URL or document still to read? | None |

Subdomain type is asked with three short questions. The answers suggest a type; the developer confirms or changes it. The type never sets how deep the model goes (SKILL.md, developer choices).

| Test | Question | Points to |
|------|----------|-----------|
| Differentiation | Would a customer choose this product because of how well this part works? | Core |
| Outsourcing | Could this be bought off the shelf without losing anything? | Generic |
| Talent | Would you put your best effort and time here? | Core |

Defaults before asking: login, notifications, file storage, logging are generic; billing, invoicing, reporting are usually supporting; the thing the product sells is core.

Reply format: a five-row table "Scope", then one question: "Confirm, or change a row."

## Phase 2: Aggregates

Propose at most seven aggregate root candidates.

```
Aggregate candidates for Order context (pick, drop or add):
1. Order     - the customer's request to buy; owns lines and status
2. Merchant  - the business that fulfils an order; owns availability
3. Delivery  - the trip from merchant to customer; owns courier and ETA

Not proposed: Payment (belongs to Billing context), Menu (belongs to Catalog).
Reply with the numbers to keep, or name what is missing.
```

Rules for the list:
- A root has its own lifecycle and at least one rule that only it can keep.
- A concept that only lives inside another root is an entity or a value object; say so in the "Not proposed" line.
- A concept that belongs to another context goes to "Not proposed" with that context's name.
- If a candidate's name already exists in another context's glossary, add one line: "Order's Merchant is a different model from Merchant's Merchant. Here it means: ..." and ask the developer to confirm the local meaning.

Wait for the selection.

## Phase 3: Behaviour and tests

Handle at most three aggregates per reply. For each selected aggregate, propose:

| Block | Budget | Shape |
|-------|--------|-------|
| Methods | 7 | Verb name, what it changes, event it raises |
| Invariants | 5 | One sentence, the method that keeps it, the exception name |
| Domain service | 0 or 1 | Only when a rule reads two roots: name and signature, such as `OrderDomainService.ProcessOrder(merchant, order)` |
| Test cases | 1 per invariant + 1 per method | Plain English sentence: "Order is rejected when the merchant is unavailable" |

```
Order - methods
1. Place()        -> Placed, raises OrderPlaced
2. MarkPaid()     -> Paid
3. Accept()       -> Accepted
4. Cancel()       -> Cancelled
5. Deliver()      -> Delivered, raises OrderDelivered

Order - invariants
a. An order is rejected when the merchant is unavailable  (ProcessOrder, MerchantUnavailableException)
b. Cancel is refused after Deliver                         (Cancel, OrderAlreadyDeliveredException)
c. An order needs at least one line                        (Place, EmptyOrderException)

Tests: one per line above, plus one happy path per method (8 tests).
Keep, drop or add by letter and number. Draw the state diagram for Order? (yes/no)
```

The state diagram question is asked only for an aggregate with a status field. No other diagram is offered in this flow.

Wait for the selection.

## Phase 4: Scaffold

Ask once: "Create Domain and its tests now. Also Application, Infrastructure or Contract?" Default: Domain and tests only.

Write, in this order, using the stack profile for names and commands:

1. The Domain module and the Domain test module.
2. One class per selected aggregate root with the selected methods as stubs that fail when called, one comment per invariant above the method that keeps it, one class per typed exception, one class per domain event, the value objects named in phases 2 and 3, the domain service with its signature.
3. One failing test per selected test case, named with the plain English sentence.
4. `UBIQUITOUS-LANGUAGE.md` with one row per aggregate, method, event, exception and value object. Format in `references/glossary-and-ledger.md`.
5. `DOMAIN-HISTORY.jsonl` with the first lines: `DecisionRecorded` for the subdomain type, the stack and every developer choice made; `AggregateIntroduced` per root; `MethodIntroduced` per method; `InvariantAdded` per invariant; `EventIntroduced` per event; `ContextRelationshipChanged` per neighbour, written in both ledgers with `related`.
6. `ubiqron/CONTEXT-MAP.md`: add the context row and the relationship rows. Create the file if it does not exist. Format in `references/context-map.md`.
7. `Diagrams/<Aggregate>.state.md` if the developer said yes in phase 3.
8. Run the build or the tests once, as the stack profile says.

Reply format: a table of files written with one count per file (classes, tests, rows, lines), one line with the build result, at most three "next" suggestions.

## Budgets

- One phase per reply. Five questions in phase 1. Seven aggregates. Seven methods and five invariants per aggregate. Three aggregates per phase-3 reply. Three relationships.
- Never paste document content. Never write before the developer selects. Never scaffold Application, Infrastructure, Api or Contract without the phase-4 answer.
