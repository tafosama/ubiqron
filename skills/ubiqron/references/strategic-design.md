# Strategic design

Strategic design answers three questions before any class is written: where the boundaries are, how important each part is, and how the parts talk to each other. ubiqron keeps the answers in `ubiqron/CONTEXT-MAP.md` and in each context's ledger.

## Bounded context

A bounded context is a boundary for meaning. Inside it, every term has one meaning and the model is consistent. Outside it, the same word may mean something else.

| Term | Merchant context | Order context |
|------|------------------|---------------|
| Merchant | The business record: address, VAT number, bank details, location | The business that fulfils an order: identity and availability |
| Order | Not present | The customer's request to buy; owns lines and status |

One class per meaning. The two `Merchant` classes never share code. Translation happens at the boundary, never inside the model.

### Signals for a boundary

| Signal | Example |
|--------|---------|
| The same word means two things | `Merchant` above |
| A concept exists on one side only | `Courier` exists in Fulfillment, not in Order |
| People translate when they talk | "When billing says account, they mean the invoice address" |
| Different consistency needs | Payments need a strict order of steps; recommendations do not |
| Different rate of change | Tax rules change yearly; the menu changes daily |

Organisation signals such as team ownership are not used. A context is owned by one developer or one team by definition.

### Size

| Too large | Too small |
|-----------|-----------|
| Concepts inside do not relate to each other; two glossaries would be clearer | Translation layers between concepts that always change together |
| A change in one aggregate forces changes in unrelated aggregates | Every use case crosses the boundary |

When in doubt, start with one context and split later. A split is recorded in the ledger as `ResponsibilityMoved`, or as `AggregateDeprecated` plus `AggregateIntroduced` (`references/glossary-and-ledger.md`).

## Subdomain type

Each context gets one type. It is recorded as `DecisionRecorded` and shown in the context map. The type never decides how deep the model goes; a generic context can still have invariants and tests.

| Type | Meaning | Typical examples |
|------|---------|------------------|
| Core | The reason a customer picks this product | Order taking, pricing, risk assessment |
| Supporting | Needed, built in house because nothing fits, not a differentiator | Billing, reporting, merchant records |
| Generic | Every business needs it; buying is normal | Login, notifications, file storage |

The three tests and their questions are in `references/new-context-flow.md`, phase 1. For a generic type, say once that an off-the-shelf product is an option, then continue with whatever the developer chooses.

## The nine relationship patterns

Every relationship has an upstream, the side that defines the model, and a downstream, the side that adapts. The last column says what the pattern looks like in a monorepo.

| Code | Pattern | Meaning | In code |
|------|---------|---------|---------|
| CS | Customer-Supplier | Upstream serves downstream and takes its needs into account | Downstream references upstream's Contract module; a translator in downstream builds local objects |
| ACL | Anti-Corruption Layer | Downstream translates a model it cannot change into its own | Adapter and translator in downstream's Infrastructure; the foreign model never enters Domain |
| OHS | Open Host Service | Upstream exposes one stable protocol for many consumers | A versioned API or event set in upstream's Api and Contract modules |
| PL | Published Language | A shared, documented format for what crosses the boundary | The Contract module: integration events and their small types |
| SK | Shared Kernel | Two contexts share a small set of value objects | The SharedKernel module, value objects only (U11) |
| CF | Conformist | Downstream uses the upstream model as is, no translation | Downstream references upstream's Contract and uses its types directly in Application |
| P | Partnership | Two contexts change together and plan together | Each references the other's Contract; releases are coordinated |
| SW | Separate Ways | No integration; each side builds what it needs | No reference between the two |
| BBM | Big Ball of Mud | An existing system with no clear model | One row in the map, type External or Supporting; an ACL around it |

### What ubiqron proposes

| Situation | Proposal |
|-----------|----------|
| Upstream is one of the developer's own contexts | CS, with an integration event in upstream's Contract and a translator in downstream |
| Upstream is an external system | ACL in downstream's Infrastructure |
| Upstream serves several downstreams | OHS plus PL in the Contract module |
| The developer says translation is not worth it | CF |
| The developer says the two sides will not integrate | SW |
| The developer asks for it | P or BBM; never proposed by ubiqron |

At most three relationships per reply. Each accepted relationship becomes a row in the context map and a `ContextRelationshipChanged` line in both ledgers.

## Published language

The published language is the part of a context that other contexts may see.

| Belongs in it | Does not |
|---------------|----------|
| Integration events, past tense: `MerchantRegistered` | Aggregates and entities |
| The small DTO and ID types those events carry | Domain events; they stay in Domain |
| A version suffix when an event changes shape | Any method with behaviour |

Default home: the context's Contract module. The developer may choose another place; the choice is recorded once as `DecisionRecorded` (U9, U10).

Changing a published type: add fields, never remove or rename them. For a breaking change, add a new type such as `MerchantRegisteredV2` and keep the old one until no consumer uses it. Record `EventIntroduced` now and `TermDeprecated` later.

## Translation at the boundary

The Merchant to Order flow, used in every explanation:

1. Merchant context: `Merchant.Register()` raises the domain event `MerchantRegistered` inside the aggregate.
2. An event handler in Merchant's Application maps it to the integration event `MerchantRegistered` in Merchant's Contract and publishes it.
3. Order context consumes the event. A translator in Order builds Order's own `Merchant` from the fields it needs: id, name, availability.
4. Order's `Merchant` never carries a VAT number or bank details. Those belong to Merchant context.

The handler, envelope and outbox side of this flow is in `references/application-layer.md`.

## When a boundary changes

| Change | Ledger lines | Context map |
|--------|--------------|-------------|
| Move an aggregate to another context | `AggregateDeprecated` in the source with `to`; `AggregateIntroduced` in the target with `from`; linked by `related` | New or changed relationship rows when the move creates a dependency |
| Move one method or rule | `ResponsibilityMoved` in both ledgers, linked by `related` | Usually none |
| Split a context | New context row; one pair of move lines per moved aggregate | Rows for the new pair |
| Merge two contexts | `AggregateIntroduced` lines in the target; the source row becomes Deprecated | Rows of the source move to the target |
| Change a relationship pattern | `ContextRelationshipChanged` in both ledgers, `from` holds the old pattern | Row updated; Source column shows the downstream id |

Propose first, write after the developer agrees. Details in `references/context-map.md` and `references/glossary-and-ledger.md`.
