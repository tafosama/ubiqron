# OrderPaymentProcess

Saga in `Order.Application/Processes/` (U12). It uses the `IPaymentGateway` port, which is the port of the anti-corruption layer towards the Payment Provider. Shows the compensation branch and the retry.

```mermaid
flowchart TD
    A([OrderPlaced]) --> B["Send AuthorizePayment through IPaymentGateway"]
    B --> C{Result}
    C -->|PaymentAuthorized| D["Order.MarkPaid()"]
    D --> E["Publish OrderPaid from Order.Contract"]
    E --> F([Done])
    C -->|PaymentDeclined| G["Order.Reject(PaymentDeclined)"]
    G --> H([Done])
    C -->|Timeout| I{"Attempts below 3?"}
    I -->|yes| B
    I -->|no| G
```

Ledger: ord-0021
