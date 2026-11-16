# Application layer

The Application module runs use cases. It loads aggregates through ports, calls the Domain, saves, and publishes. It holds no business rules; those belong in aggregates and domain services. It knows nothing about HTTP or a database; those are Infrastructure and Api.

## Folders

Defaults. The developer may change them; record the change once as `DecisionRecorded`.

```
Application/
  <Aggregate>/
    Commands/<UseCase>/     one command and its handler
    Queries/<Query>/        one query and its handler
    Ports/                  repository port for this root (U4)
  Ports/                    context-level ports: publisher, unit of work, external systems
  Events/                   handlers for domain events; translation to integration events
  Processes/                sagas and process managers (U12)
```

## Command and command handler

A command is the intent to change one aggregate: `PlaceOrder`, `CancelOrder`. One handler per command.

The handler does five things, in order:

1. Load the aggregates it needs through ports.
2. Call one aggregate method or one domain service.
3. Save through the repository port, inside one unit of work.
4. Hand the domain events the aggregate raised to the event handlers or the outbox.
5. Return the id or nothing. Never the aggregate.

```
class PlaceOrderHandler
  orders: OrderRepository          // port, Application/Order/Ports
  merchants: MerchantRepository    // port, Application/Merchant/Ports
  domainService: OrderDomainService
  unitOfWork: UnitOfWork           // port, or the repository's save call

  Handle(command: PlaceOrder) -> OrderId
    merchant = merchants.GetById(command.MerchantId)
    order = Order.Create(command.CustomerId, command.MerchantId)
    for each line in command.Lines: order.AddLine(line.ItemId, line.Quantity, line.Price)
    domainService.ProcessOrder(merchant, order)     // may throw MerchantUnavailableException
    orders.Add(order)
    unitOfWork.Commit()
    return order.Id
```

A business `if` in a handler, such as `if merchant.IsAvailable()`, is a rule in the wrong place. Move it to the aggregate or the domain service.

Domain exceptions pass through the handler. Api maps them to responses.

## Query and query handler

A query reads. It changes nothing and raises no events. One handler per query. It returns a result type made for the caller, not the aggregate.

The handler may use the repository port (U8) or a separate read port. Both are fine. The developer picks; do not propose a read port unless the developer asks.

```
class GetOrderByIdHandler
  orders: OrderRepository
  Handle(query: GetOrderById) -> OrderView
```

## Ports

A port is an interface that Application owns and Infrastructure implements.

| Port | Where | Notes |
|------|-------|-------|
| Repository | `Application/<Aggregate>/Ports/` | One per aggregate root, named after it (U4). The method set is the developer's choice |
| Unit of work | `Application/Ports/` | Optional; some stacks commit through the repository |
| Event publisher or outbox | `Application/Ports/` | Used by event handlers and processes |
| External system | `Application/Ports/` | `PaymentGateway`, `GeocodingService`; the adapter in Infrastructure translates (ACL) |

A port speaks the domain language: it takes and returns aggregates, value objects and ids, not database rows. No port lives in Domain (U3). No domain service takes one (U5).

## Unit of work

One command, one transaction. The handler opens it, the Domain runs, the handler commits. When a domain service changes two roots (U6), both are saved in the same transaction. How the transaction is opened is the developer's choice: an explicit port, the repository's save call, or the stack's own mechanism.

## Domain events inside the context

An event handler in `Application/Events/` reacts to a domain event from the same context. It runs after the aggregate is saved. It may load other aggregates through ports and call their methods, each in its own transaction. This is how one aggregate's change reaches another without a shared transaction.

## Publishing to another context

Reference flow, Merchant to Order. Each piece is described; no shape is imposed.

1. `Merchant.Register()` raises the domain event `MerchantRegistered` in Domain. Domain data only.
2. An event handler in Merchant's `Application/Events/` maps it to the integration event `MerchantRegistered` in Merchant's Contract module. Only the fields consumers need: merchant id, name, availability.
3. The handler wraps it in an envelope and writes it to an outbox row in the same transaction as the aggregate.
4. A background publisher reads the outbox and sends the message. Delivery is at least once.
5. Order's consumer in its Infrastructure receives the message, checks the eventId against the ones already seen, and hands the event to a translator.
6. The translator in Order creates Order's own `Merchant`, through a command or directly through the repository port.

Envelope fields, as commonly used:

| Field | Meaning |
|-------|---------|
| eventId | Unique per occurrence; consumers use it to skip duplicates |
| eventType | Name and version of the integration event |
| occurredAt | When the aggregate raised it |
| aggregateId, aggregateType | Which aggregate produced it |
| payload | The integration event |
| correlationId, causationId | The request that started the chain, and the event that caused this one |

The envelope shape, the outbox table and the publisher are the developer's choice. A messaging library brings its own; ubiqron does not mandate one. Two points are fixed: the domain event class has no envelope fields (U10), and the integration event type lives in the published-language module, Contract by default (U9).

## Processes

A process, also called a saga or process manager, drives a flow that spans several aggregates or contexts over time. It lives in `Application/Processes/` (U12).

| A process | A domain service |
|-----------|------------------|
| Uses ports: repositories, publisher, external systems | Takes no ports |
| Has its own state: which step it is on | Stateless |
| Reacts to events, sends commands | Is called by a handler |
| Spans transactions; compensates on failure | Runs inside one transaction |

```
class OrderPaymentProcess
  state: orderId, attempts, status

  On(OrderPlaced)       -> paymentGateway.Charge(...)                       // port
  On(PaymentSucceeded)  -> send MarkOrderPaid
  On(PaymentFailed)     -> if attempts < 3 then retry else send CancelOrder  // compensation
```

A process is written only when the developer asks for it. Its flow diagram is described in `references/diagrams.md`.

## Developer choices in this layer

Ask once, record as `DecisionRecorded`, never ask again, never report as a finding.

| Choice | When to ask | Options |
|--------|-------------|---------|
| Dispatch | Application module is scaffolded | Own handler interfaces wired by hand or by DI; any library |
| Unit of work | First repository port | Explicit port; repository save call; stack mechanism |
| Repository methods | First repository port | Collection style (`Add`, `Remove`); persistence style (`Update`, `Save`); queries on the same port |
| Envelope and outbox | First integration event | Own table and publisher; messaging library |
| Home of the published language | First integration event | Contract module (default); another module |
| Read port for queries | Never ask | The repository port is fine (U8) |

Stack-specific options are in the stack profile.
