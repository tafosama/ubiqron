# Stack profile: .NET

Use this profile when the repo has `*.csproj`, `*.slnx` or `*.sln` files.

## Principle

The solution tree and the folder tree are the same thing. One bounded context is one folder under `src/` with one project per layer. Project boundaries are rules. Folders inside a project are defaults the developer may change.

## Tree

```
<repo>/
  <Solution>.slnx                                 or .sln if the repo uses it
  ubiqron/CONTEXT-MAP.md
  src/
    SharedKernel/SharedKernel.csproj              optional; value objects only
    <Context>/
      <Context>.Domain/<Context>.Domain.csproj
      <Context>.Application/<Context>.Application.csproj
      <Context>.Infrastructure/<Context>.Infrastructure.csproj
      <Context>.Api/<Context>.Api.csproj
      <Context>.Contract/<Context>.Contract.csproj  published language, default home
  tests/
    <Context>.Domain.Tests/<Context>.Domain.Tests.csproj   xUnit
```

## Inside each project

Defaults. The developer may move folders inside a project; the project boundary stays.

```
<Context>.Domain/
  UBIQUITOUS-LANGUAGE.md
  DOMAIN-HISTORY.jsonl
  Primitives/               AggregateRoot, Entity, ValueObject, DomainException,
                            IDomainEvent, IHasDomainEvents, IHasVersion, IAuditable
  <Aggregate>/              the root, its entities, value objects, domain events, domain exceptions
  Services/                 domain services and their interfaces

<Context>.Application/
  <Aggregate>/
    Commands/<UseCase>/     <UseCase>Command.cs, <UseCase>Handler.cs
    Queries/<UseCase>/      <UseCase>Query.cs, <UseCase>Handler.cs, result type
    Ports/                  I<Aggregate>Repository.cs and other ports this aggregate needs
  Ports/                    ports shared by the context, such as IUnitOfWork, IIntegrationEventPublisher
  Events/                   domain event handlers, including translation to integration events
  Processes/                sagas and process managers

<Context>.Infrastructure/
  Persistence/              DbContext, entity configurations, repository implementations, outbox
  Messaging/                publisher and consumer adapters
  <ExternalSystem>/         adapter plus translator for one external system: the anti-corruption layer
  DependencyInjection.cs    registers adapters against ports

<Context>.Contract/
  Events/                   integration events, past tense
  <small DTO and ID types the events carry>

<Context>.Api/              entry points and the composition root
```

## Allowed project references

This table is rules U1, U9 and U11 in csproj terms.

| Project | May reference |
|---------|---------------|
| `<Context>.Domain` | `SharedKernel` |
| `<Context>.Application` | `<Context>.Domain`; any `.Contract` project, own or another context's |
| `<Context>.Infrastructure` | `<Context>.Application`; any `.Contract` project |
| `<Context>.Api` | `<Context>.Application`, `<Context>.Infrastructure` |
| `<Context>.Contract` | `SharedKernel` |
| `<Context>.Domain.Tests` | `<Context>.Domain` |

Any other reference is a finding.

## Boundary check

List every project reference as `from -> to` and compare with the table above:

```bash
grep -rHo '<ProjectReference Include="[^"]*"' --include='*.csproj' src tests \
  | sed -E 's#^(.*)/[^/]+\.csproj:<ProjectReference Include="(.*)"#\1 -> \2#'
```

- A `.Domain` project that references anything except `SharedKernel`: U1.
- A reference to another context's `.Domain`, `.Application`, `.Infrastructure` or `.Api`: U9.
- A `.Contract` project that references anything except `SharedKernel`: U1.

Other checks in review mode:

| Rule | How to look |
|------|-------------|
| U3 | `grep -rn "interface I" src/<Context>/<Context>.Domain --include='*.cs'`. A name ending in `Port`, `Repository`, `Publisher`, `Gateway`, `Client` or `Store` is a port. For other names, check whether the implementation lives in Infrastructure; if so, it is a port. |
| U5 | Constructor and method parameters of classes under `Domain/Services`. A parameter whose type is an interface declared outside Domain is a port. |
| U7 | `throw new Exception(`, `throw new InvalidOperationException(` or `throw new DomainException(new` inside Domain. |
| U8 | A query handler that calls a `DbContext` directly is fine only if the developer chose that; it is never a finding. A command handler with no unit of work is not a finding either. |
| U14 | Grep for each deprecated subject from the ledger across `src/<Context>`. |

## Naming

| Thing | Convention | Example |
|-------|------------|---------|
| Interface | `I` prefix | `IOrderRepository`, `IOrderDomainService` |
| Marker interfaces in Domain | `I` prefix | `IDomainEvent`, `IHasDomainEvents`, `IHasVersion`, `IAuditable` |
| Domain exception | `<Reason>Exception`, derives from `DomainException` | `MerchantUnavailableException` |
| Domain event | past tense, `<Aggregate><Action>` | `OrderPlaced` |
| Integration event | past tense, in `.Contract/Events` | `MerchantRegistered` |
| Command and handler | `<Verb><Aggregate>Command`, `<Verb><Aggregate>Handler` | `PlaceOrderCommand`, `PlaceOrderHandler` |
| Query and handler | `<Get...>Query`, `<Get...>Handler` | `GetOrderByIdQuery`, `GetOrderByIdHandler` |
| Repository port | `I<Aggregate>Repository` in `Application/<Aggregate>/Ports/` | `IOrderRepository` |
| Test method | plain English with underscores | `Order_is_rejected_when_merchant_is_unavailable` |

## Scaffold commands

Use the target framework the repo already uses. Read `Directory.Build.props` or an existing `.csproj`. Never hardcode a framework version.

Default scaffold, phase 4: Domain and its test project.

```bash
dotnet new classlib -n <Context>.Domain -o src/<Context>/<Context>.Domain
rm src/<Context>/<Context>.Domain/Class1.cs
dotnet new xunit -n <Context>.Domain.Tests -o tests/<Context>.Domain.Tests
rm tests/<Context>.Domain.Tests/UnitTest1.cs
dotnet add tests/<Context>.Domain.Tests reference src/<Context>/<Context>.Domain
dotnet sln <Solution>.slnx add src/<Context>/<Context>.Domain tests/<Context>.Domain.Tests
```

If `SharedKernel` exists and the context uses its types:

```bash
dotnet add src/<Context>/<Context>.Domain reference src/SharedKernel
```

Only when the developer chooses them:

```bash
dotnet new classlib -n <Context>.Application -o src/<Context>/<Context>.Application
dotnet add src/<Context>/<Context>.Application reference src/<Context>/<Context>.Domain
dotnet new classlib -n <Context>.Contract -o src/<Context>/<Context>.Contract
dotnet new classlib -n <Context>.Infrastructure -o src/<Context>/<Context>.Infrastructure
dotnet add src/<Context>/<Context>.Infrastructure reference src/<Context>/<Context>.Application
dotnet sln <Solution>.slnx add src/<Context>/<Context>.Application src/<Context>/<Context>.Contract src/<Context>/<Context>.Infrastructure
```

The Api project depends on the developer's host choice. Ask which template (`webapi`, `worker`) before creating it.

After scaffolding, run `dotnet build` once and report the result in one line.

## Test skeleton

One test class per aggregate or domain service. One `[Fact]` per selected test case. The body fails until the developer or the skill writes it.

```csharp
public class OrderDomainServiceTests
{
    [Fact]
    public void Order_is_rejected_when_merchant_is_unavailable()
    {
        Assert.Fail("Not implemented. Write when OrderDomainService.ProcessOrder exists.");
    }
}
```

## Ledger validation test

Link the ledger file into the test project so the test reads the real file:

```xml
<ItemGroup>
  <Content Include="../../src/<Context>/<Context>.Domain/DOMAIN-HISTORY.jsonl"
           Link="DOMAIN-HISTORY.jsonl"
           CopyToOutputDirectory="PreserveNewest" />
</ItemGroup>
```

`DomainHistoryTests` checks, per line: valid JSON; required fields present; `kind` is one of the 13 kinds; `id` is unique and in order; `supersedes` points to an id in the same file; `at` is an ISO date. It does not check `related`; review mode does that across contexts. The test body is in `references/glossary-and-ledger.md`.

## Developer-choice options in .NET

Ask once, record as `DecisionRecorded`, never report as a finding.

| Choice | Options seen in .NET |
|--------|----------------------|
| Reconstitution | EF Core with a private parameterless constructor and backing fields; or a static `Reconstitute(...)` method that the repository calls after mapping from a persistence model |
| Repository methods | EF Core change tracking: `Add`, `Remove`, no `Update`. Dapper or raw SQL: explicit `Update` or `Save`. Commit through an `IUnitOfWork` port or a save method on the repository port. Query methods on the same port are fine. |
| Dispatcher | Own `ICommandHandler<TCommand>` and `IQueryHandler<TQuery, TResult>` registered in DI; MediatR, now under a commercial licence; Wolverine; any other |
| Envelope and outbox | Own outbox table plus a background publisher; MassTransit; Wolverine; CAP |
| Current time | Whatever the code does: a `DateTimeOffset` passed from the handler, `TimeProvider`, or `DateTime.UtcNow` inside Domain. Never ask, never a finding. |
