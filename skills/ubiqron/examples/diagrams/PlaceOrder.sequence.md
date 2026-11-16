# PlaceOrder

Command flow inside the Order context. Shows the repository port (U4), the port-free domain service (U5), the typed exception (U7) and the aggregate raising its event.

```mermaid
sequenceDiagram
    participant Api
    participant H as PlaceOrderHandler
    participant MR as IMerchantRepository
    participant S as OrderDomainService
    participant O as Order
    participant OR as IOrderRepository
    Api->>H: PlaceOrderCommand
    H->>MR: GetById(merchantId)
    MR-->>H: merchant
    H->>O: Order.Create(...)
    H->>S: ProcessOrder(merchant, order)
    alt merchant unavailable
        S-->>H: MerchantUnavailableException
        H-->>Api: rejected
    else merchant available
        S->>O: Place()
        Note over O: raises OrderPlaced
        H->>OR: Add(order)
        H-->>Api: OrderId
    end
```

Ledger: ord-0003
