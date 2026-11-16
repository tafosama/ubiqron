# Tactical patterns

What goes in a Domain module and how to shape it. Examples use the food-ordering domain from SKILL.md. Code is pseudocode; the stack profile sets the real syntax and names.

## Aggregate

An aggregate is a group of objects changed together as one unit. One entity is the root. Outside code talks only to the root.

### Boundary

Start from the invariants, not from the database tables.

| If the rule spans | Then |
|-------------------|------|
| One entity | That entity is its own aggregate |
| An entity and the objects it owns | One aggregate; the owner is the root |
| Two things with their own lifecycle | Two aggregates; a domain service or a domain event links them |

Test: "Must both changes be true at the same moment?" Yes: same aggregate. A short delay is fine: two aggregates and an event.

### Rules

1. The root keeps every invariant of the aggregate. No outside code changes an inner object.
2. Keep it small: one root, a few value objects, sometimes a short list of child entities. `Order` with its lines is the usual size.
3. Reference another aggregate by id only. `Order` holds `MerchantId`, never `Merchant`.
4. One command changes one aggregate. Effects on other aggregates travel by domain event. The exception is U6: one invariant across two roots, kept by a domain service, saved in one unit of work, same context.
5. Every state change is a method with a business name. No setters.

```
class Order
  id: OrderId
  merchantId: MerchantId
  lines: list of OrderLine
  status: OrderStatus

  static Create(customerId, merchantId) -> Order
  AddLine(menuItemId, quantity, price)
  Place()        // status Placed, raises OrderPlaced; throws EmptyOrderException
  Cancel()       // throws OrderAlreadyDeliveredException
```

### Status and transitions

An aggregate with a status field has a small state machine. Each transition is one method. A method refuses a transition with a typed exception. Phase 3 offers a state diagram for it (`references/diagrams.md`).

## Entity

An entity is defined by identity. It stays the same thing while its fields change.

| Question | Entity | Value object |
|----------|--------|--------------|
| Same fields, different thing? | Yes; two orders can look alike | No; two equal `Money` values are one value |
| Has a lifecycle? | Yes | No; replaced, not changed |
| Mutable? | Yes, through methods | No |
| Compared by | Id | All fields |

Rules: the id never changes; equality is by id; behaviour sits on the entity, not in a service. An entity that is not a root lives inside one aggregate and is reached through the root: `OrderLine` inside `Order`.

## Value object

A value object is defined by its fields. It is immutable and valid from birth.

1. All fields are set at creation and never change. A change returns a new value.
2. It checks itself at creation. A `Money` with a negative amount throws. If a `Money` exists, it is valid.
3. Equal when all fields are equal.
4. Methods have no side effects: `money.Add(other)` returns a new `Money`.

| Value object | Replaces | Why |
|--------------|----------|-----|
| `Money(amount, currency)` | a number | No mixing of currencies; rounding in one place |
| `MerchantId`, `OrderId` | a string or a GUID | An order id cannot be passed where a merchant id is expected |
| `Address(street, city, postcode, country)` | four strings | Validated as one unit |
| `Quantity(value, unit)` | a number | No adding of kilos to litres |

When in doubt, make it a value object. Promoting it to an entity later is easy; the other way is not.

A value object used by more than one context goes to the SharedKernel module (U11). Everything else stays in its context.

## Domain event

A domain event is a fact that happened in the domain, named in past tense: `<Aggregate><Action>`.

| Good | Not an event name |
|------|-------------------|
| `OrderPlaced`, `OrderCancelled`, `MerchantBecameUnavailable` | `PlaceOrder` is a command; `OrderUpdated` says nothing; `OrderEvent` says nothing |

1. Raised inside the aggregate method that caused it. The aggregate keeps a list; the handler reads the list after saving.
2. Holds domain data only: ids and the values a listener needs. No envelope fields such as eventId, occurredAt or correlationId (U10). Those are added at the boundary (`references/application-layer.md`).
3. One class per fact. Specific names: `OrderLineAdded`, not `OrderChanged`.
4. A domain event stays inside the context. What crosses to another context is an integration event in the Contract module.

```
class OrderPlaced
  orderId: OrderId
  merchantId: MerchantId
  total: Money
```

## Domain service

A domain service holds a rule that needs more than one aggregate, or a rule that belongs to no aggregate.

| Use one when | Do not use one when |
|--------------|---------------------|
| A rule reads two roots: "an order is rejected when the merchant is unavailable" | The rule touches one aggregate only; put it on the aggregate |
| A calculation needs values from several aggregates | The work needs a port; that is a handler or a process (U5, U12) |

1. No ports (U5). Inputs are aggregates and value objects the handler has loaded.
2. Default: change one aggregate, read the others (U6).
3. Exception: one invariant across two roots in the same context. The service changes both, the handler saves both in one unit of work, the ledger names the invariant.
4. Returns a result or throws a typed domain exception.
5. Stateless. An interface for it is allowed and optional (U3).

```
class OrderDomainService
  ProcessOrder(merchant: Merchant, order: Order)
    if not merchant.IsAvailable() then throw MerchantUnavailableException(merchant.Id)
    order.Place()

class PurchasingDomainService
  Invoice(purchaseOrder: PurchaseOrder, lines: list of InvoiceLine) -> PurchaseInvoice
    invoice = PurchaseInvoice.Create(purchaseOrder.Id, lines)
    for each line in lines: purchaseOrder.InvoiceLine(line.ItemId, line.Quantity)
    return invoice
    // invariant: the invoiced quantity is on the purchase order at once (ledger pur-0012)
```

The second example is the U6 exception. The handler saves `purchaseOrder` and `invoice` in one unit of work.

## Domain exception

A domain exception is a business refusal with a name.

1. One class per refusal: `MerchantUnavailableException`, `EmptyOrderException`, `OrderAlreadyDeliveredException` (U7).
2. Usually derived from one base, `DomainException`, so outer layers can catch the family.
3. Named `<Reason>Exception`. The message is one plain sentence with the ids involved.
4. No general exception with a code, and no message switch. In a language without exceptions, one error type per refusal.
5. A value object that rejects its input throws a typed exception too: `NegativeAmountException`, `UnknownCurrencyException`.

Each invariant in phase 3 names the exception that guards it. The glossary lists each exception with the rule it enforces.

## Factory

A factory builds an aggregate or a value object when creation has rules.

| Shape | When |
|-------|------|
| Static `Create(...)` on the root | Default. Creation checks its own invariants and raises the first event |
| A separate factory class in Domain | Creation needs several objects or has variants, such as `OrderFactory.FromReorder(previousOrder)` |
| Reconstitution | Loading from storage. Not a factory call; the style is the developer's choice |

A factory never calls a port. If creation needs data from elsewhere, the handler loads it and passes it in.

## Naming

- Classes and methods use glossary terms. A method reads like a sentence the business would say: `order.Cancel()`, not `order.SetStatus(4)`.
- No technical suffixes in Domain: no Dto, Manager, Helper, Processor, Record, Row.
- One term per concept. If two words are used for one thing, pick one and record the other as `TermDeprecated`.
- A name that is hard to find means a concept is missing or two concepts are mixed. Say so in the proposal.

## Common mistakes

| Mistake | Fix |
|---------|-----|
| Aggregate with getters and setters, rules in a service | Move the rule into the aggregate method |
| `Order` holds a `Merchant` object | Hold `MerchantId`; pass `Merchant` into a domain service when a rule needs it |
| One aggregate holds a whole object graph | Split along invariants; reference by id |
| A domain service takes a repository | U5. The handler loads, the service decides |
| One exception type with a code | U7. One typed exception per refusal |
| Domain event with eventId and occurredAt fields | U10. Move those to the envelope at the boundary |
| Entity named `OrderRecord` or `OrderRow` | Name it `Order` |
