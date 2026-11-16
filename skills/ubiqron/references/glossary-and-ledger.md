# Glossary and ledger

Two files per context, both inside the Domain module. The ledger is the source: every modeling action appends one line. The glossary is the current state, rebuilt from the ledger whenever the ledger changes (U13).

| File | Role |
|------|------|
| `<Context>.Domain/UBIQUITOUS-LANGUAGE.md` | What every term means today |
| `<Context>.Domain/DOMAIN-HISTORY.jsonl` | Every change, in order, never edited |

## Glossary format

```markdown
# Order: ubiquitous language

Current meaning of every term in the Order context. Rebuilt from DOMAIN-HISTORY.jsonl. Last ledger line: ord-0021.

| Term | Definition | Context | Example |
|------|------------|---------|---------|
| Order | The customer's request to buy from one merchant. Owns its lines and its status. | Order | "Order 4711 was placed at 12:03" |
| Order.Place | Moves the order to Placed. Refused when the order has no lines. Raises OrderPlaced. | Order | "The customer placed the order" |
| OrderPlaced | Event: an order was placed. Carries order id, merchant id, total. | Order (published) | "Fulfillment listens for OrderPlaced" |
| EmptyOrderException | Refusal: the order has no lines. Thrown by Order.Place. | Order | "Order rejected: no lines" |
| Merchant | The business that fulfils an order: identity and availability only. Not the Merchant of the Merchant context. | Order | "The merchant is unavailable" |
| OrderDomainService.ProcessOrder | Places an order when the merchant is available. Refused otherwise. | Order | "Process the order for merchant 17" |
| MerchantUnavailableException | Refusal: the merchant does not take orders right now. Thrown by OrderDomainService.ProcessOrder. | Order | "Order rejected: merchant 17 is unavailable" |
| Money | An amount with a currency. | SharedKernel | "12.50 EUR" |
| Shipment | Deprecated 2026-11-09, moved to Fulfillment. | Order | |
```

Rules:

- One row per aggregate, entity, value object, method, domain event, integration event, domain service method and exception. Methods are written `Aggregate.Method` or `Service.Method`.
- Definition: at most two sentences, plain English. A method: what it changes, what refuses it, what it raises. An exception: the refusal and who throws it.
- Context: this context's name. Add `(published)` when the term is in the published language. A Shared Kernel value object says `SharedKernel`.
- Example: one short sentence in the business's words. May be empty.
- A deprecated term keeps its row. The definition becomes `Deprecated <date>` plus `moved to <Context>` or `use <Term>`. Remove the row only when the code no longer has the item.
- No technical terms: no Repository, Handler, Controller, Dto.
- Rows are grouped by aggregate: the aggregate, then its methods, events and exceptions. Domain services, value objects and deprecated terms at the end.
- Line 3 names the last ledger id the file reflects. Review mode compares it with the ledger.

## Ledger schema

One JSON object per line. Append only. Never edit or delete a line; correct with a new line that has `supersedes`.

| Field | Required | Value |
|-------|----------|-------|
| `id` | yes | `<prefix>-<number>`: a three-letter lowercase prefix from the context name (`ord`, `mer`, `ful`) and a four-digit number, sequential in this file |
| `at` | yes | Date, `YYYY-MM-DD` |
| `context` | yes | This context's name |
| `kind` | yes | One of the 13 kinds below |
| `subject` | yes | What changed: `Order`, `Order.Place`, `OrderPlaced`, `MerchantUnavailableException`, a term, or the other context's name |
| `from` | no | The old value, or the source context in a move |
| `to` | no | The new value, the invariant sentence, or the target context in a move |
| `reason` | yes | One sentence, plain English |
| `ref` | no | Free text: a document, an issue, a branch, a URL the developer gave |
| `supersedes` | no | An `id` in the same file whose statement this line corrects |
| `related` | no | `<Context>:<id>`, the matching line in another context's ledger |

## Kinds

| Kind | subject | from | to | When |
|------|---------|------|----|------|
| `TermIntroduced` | the term | | definition | A value object, entity or business term enters the glossary |
| `TermRenamed` | old name | old name | new name | The business or the developer renames a concept |
| `TermDeprecated` | the term | | replacement term or target context | The term leaves the model |
| `AggregateIntroduced` | aggregate | source context in a move | one-sentence purpose | New aggregate root, or one arriving from another context |
| `AggregateDeprecated` | aggregate | | target context in a move | Root removed or moved away |
| `InvariantAdded` | `Aggregate.Method` or `Service.Method` | | the invariant sentence | A rule is added; name the exception in `reason` |
| `InvariantRemoved` | same | the invariant sentence | | A rule is dropped |
| `ResponsibilityMoved` | `Aggregate.Method` or the rule | old owner | new owner | A method or rule moves to another aggregate or context |
| `MethodIntroduced` | `Aggregate.Method` | | what it does and the event it raises | New method on a root or a domain service |
| `MethodDeprecated` | `Aggregate.Method` | | replacement method, or empty | Method removed |
| `EventIntroduced` | event name | | fields carried; add `published` for an integration event | New domain or integration event |
| `ContextRelationshipChanged` | the other context | old pattern, or empty | role, pattern, mechanism | A relationship is added, changed or removed; format in `references/context-map.md` |
| `DecisionRecorded` | the choice | | the answer | Subdomain type, stack, any developer choice |

There is no kind for diagrams. Drawing writes nothing (U15).

## Examples

Order ledger:

```jsonl
{"id":"ord-0001","at":"2026-11-06","context":"Order","kind":"DecisionRecorded","subject":"subdomain type","to":"Core","reason":"Customers choose the product for how well ordering works"}
{"id":"ord-0002","at":"2026-11-06","context":"Order","kind":"AggregateIntroduced","subject":"Order","to":"The customer's request to buy from one merchant","reason":"Phase 2 selection","ref":"docs/ordering-brief.md"}
{"id":"ord-0003","at":"2026-11-06","context":"Order","kind":"InvariantAdded","subject":"OrderDomainService.ProcessOrder","to":"An order is rejected when the merchant is unavailable","reason":"Busy merchants can stop taking new orders; MerchantUnavailableException"}
{"id":"ord-0004","at":"2026-11-06","context":"Order","kind":"EventIntroduced","subject":"OrderPlaced","to":"orderId, merchantId, total; published","reason":"Fulfillment starts a shipment from it"}
{"id":"ord-0005","at":"2026-11-06","context":"Order","kind":"ContextRelationshipChanged","subject":"Merchant","to":"downstream, Customer-Supplier, MerchantRegistered","reason":"Order needs to know which merchants exist","related":"Merchant:mer-0004"}
{"id":"ord-0006","at":"2026-11-07","context":"Order","kind":"TermRenamed","subject":"Restaurant","from":"Restaurant","to":"Merchant","reason":"The business also serves shops, not only restaurants"}
{"id":"ord-0007","at":"2026-11-08","context":"Order","kind":"InvariantAdded","subject":"OrderDomainService.ProcessOrder","to":"An order is rejected when the merchant is unavailable or closed","reason":"Closed merchants were missing from the rule","supersedes":"ord-0003"}
{"id":"ord-0008","at":"2026-11-09","context":"Order","kind":"AggregateDeprecated","subject":"Shipment","to":"Fulfillment","reason":"Shipping has its own rules and its own pace of change","related":"Fulfillment:ful-0001"}
```

Fulfillment ledger, the other half of the move:

```jsonl
{"id":"ful-0001","at":"2026-11-09","context":"Fulfillment","kind":"AggregateIntroduced","subject":"Shipment","from":"Order","to":"The trip from merchant to customer","reason":"Shipping has its own rules and its own pace of change","related":"Order:ord-0008"}
```

## Two-context operations

Moving an aggregate, a method, a rule or a term, and changing a relationship, touch two ledgers.

1. Read the last id in both files. Compute both new ids.
2. Write the source line with the Deprecated or Moved kind and `to` = target context. Write the target line with the Introduced or Added kind and `from` = source context. Each `related` points at the other.
3. Update both glossaries: the source row becomes deprecated with `moved to <Context>`; the target gains the row.
4. Update `ubiqron/CONTEXT-MAP.md` when a relationship changed.
5. Reply with both ids.

Never leave one side written. If a write fails, write the missing line before anything else.

## Maintain mode: from request to lines

| Developer says | Ledger | Glossary | Code |
|----------------|--------|----------|------|
| "Rename Restaurant to Merchant" | `TermRenamed` | Row renamed; the old name stays one row as deprecated | Rename class, methods, events, tests |
| "Deprecate X" | `TermDeprecated`, `AggregateDeprecated` or `MethodDeprecated` | Row marked deprecated | Mark the item; U14 reports remaining use |
| "Add a rule: ..." | `InvariantAdded` | Method row updated with the refusal | Guard and exception added, one test |
| "Drop the rule ..." | `InvariantRemoved` | Method row updated | Guard removed, test removed |
| "Add method X to Y" | `MethodIntroduced`; `EventIntroduced` if it raises one | New rows | Method stub, test |
| "Move X to context Z" | One pair of lines, see above | Both glossaries | Class moved; references through Contract only |
| "Z now feeds Y by events" | `ContextRelationshipChanged` in both | None | Context map row |
| "Why was X deprecated?" | Read only | | Reply with the line's reason, date and `related` partner |
| "What did X mean on <date>?" | Read only | | Replay, below |

Propose the lines first, in the reply, as a short table: kind, subject, to, reason. Write after the developer agrees. One modeling action per reply unless the developer lists several.

## Replaying history

To answer "what did X mean on date D":

1. Take every line whose `subject` is X or starts with `X.`, with `at` on or before D.
2. Drop every line that a later line in the set `supersedes`.
3. Follow `related` into the other ledger when X was moved.
4. Reply with a short table: date, kind, what it said. Offer the timeline diagram (`references/diagrams.md`) when there are more than five lines.

## Validation test

Each Domain test module has one test that reads its own ledger file. The stack profile says how to link the file. It checks only its own file; `related` targets are checked by review mode.

```
test "domain history is valid"
  lines = all lines of DOMAIN-HISTORY.jsonl, empty lines skipped
  seenIds = empty set
  previousNumber = 0
  for each line, with its line number N
    record = parse JSON; fail "line N is not valid JSON" when parsing fails
    for each field in [id, at, context, kind, subject, reason]
      fail "line N: missing <field>" when absent or empty
    fail "line N: unknown kind <kind>" when kind is not one of the 13 kinds
    fail "line N: id <id> is not <prefix>-<4 digits>" when id does not match
    fail "line N: id <id> is out of order" when its number <= previousNumber
    fail "line N: duplicate id <id>" when id is in seenIds
    fail "line N: at <at> is not YYYY-MM-DD" when at does not parse as a date
    fail "line N: context <context> does not match this module" when context differs
    when supersedes is present: fail "line N: supersedes <id> not found" unless it is in seenIds
    when related is present: fail "line N: related must be <Context>:<id>" unless it has that shape
    add id to seenIds; previousNumber = its number
```

The failure message names the line and the field. A malformed line is a file error, not a model change: fix it in place. A wrong statement in a valid line is corrected with a new line that has `supersedes`.

## Review checks on these files

Review mode (`references/review-mode.md`) checks: every code item has a glossary row and a ledger line (U13); no deprecated subject is used in code (U14); every `related` resolves in the other ledger; the glossary's last ledger id equals the last line of the file.
