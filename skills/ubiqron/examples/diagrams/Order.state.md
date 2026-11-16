# Order lifecycle

Transition labels are the aggregate's methods. Notes are the invariants that refuse a transition. A state with no transition for a method means that method is refused there.

```mermaid
stateDiagram-v2
    [*] --> Placed : Place()
    Placed --> Rejected : Reject(reason)
    Placed --> Paid : MarkPaid()
    Paid --> Accepted : Accept()
    Paid --> Cancelled : Cancel()
    Accepted --> Delivered : Deliver()
    Accepted --> Cancelled : Cancel()
    Delivered --> [*]
    Rejected --> [*]
    Cancelled --> [*]
    note right of Placed
        ProcessOrder rejects when the merchant is unavailable
    end note
    note right of Delivered
        Cancel is refused after Deliver
    end note
```

Ledger: ord-0017
