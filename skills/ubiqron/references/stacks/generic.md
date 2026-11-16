# Stack profile: generic

Use this profile when no other profile matches the repo. It also holds the template for writing a new profile.

## Read the code first

Before asking anything, read three things: the build or package files at the repo root, one existing module, one existing test. Take from the code:

- Folder case: kebab, snake or Pascal.
- How modules import each other.
- The test framework and how tests are run.
- Interface and exception conventions, such as an `I` prefix or error types.

Ask only what the code does not show.

## Two questions, asked once

Record both answers as `DecisionRecorded` lines in the new context's ledger.

1. How do you run tests, and with which framework? Examples: `npm test` with vitest, `pytest`, `go test ./...`, `mvn test`.
2. How are module boundaries enforced here? Options: build tool modules such as workspaces, Gradle modules or Go packages; a lint rule on imports; nothing yet.

If the answer to question 2 is "nothing", say once that ubiqron will check boundaries by reading imports, then move on.

## Tree

```
src/<context>/domain
src/<context>/application
src/<context>/infrastructure
src/<context>/api
src/<context>/contract          published language, default home
src/shared-kernel               optional; value objects only
tests/<context>/domain
ubiqron/CONTEXT-MAP.md
```

Use the repo's case convention for folder names. If the repo already has a layout convention, keep it and map the five layers onto it. Write that mapping at the top of the context's `UBIQUITOUS-LANGUAGE.md`.

The domain folder owns `UBIQUITOUS-LANGUAGE.md` and `DOMAIN-HISTORY.jsonl`.

## Allowed dependencies

This table is rules U1, U9 and U11 in folder terms.

| Module | May depend on |
|--------|---------------|
| `domain` | `shared-kernel`, the language's standard library |
| `application` | `domain`; any `contract`, own or another context's |
| `infrastructure` | `application`; any `contract` |
| `api` | `application`, `infrastructure` |
| `contract` | `shared-kernel` |
| `tests/<context>/domain` | `domain` |

Any other dependency is a finding.

## Boundary check by imports

When the build tool does not enforce boundaries, review mode reads imports:

1. List every import statement under `src/<context>/domain`. Use the language's keyword: `import`, `from`, `require`, `using`, `use`.
2. Drop imports that resolve inside `domain`, inside `shared-kernel`, or to the standard library.
3. Each remaining import is a U1 finding. If it is an interface that an outer layer implements for I/O, report it as U3 instead.
4. List every import under `src/<context>` that points into another context's folder. Each one that is not that context's `contract` is a U9 finding.

Third-party packages inside `domain` are not a finding by default. Mention them once if they are I/O libraries such as an ORM or HTTP client.

## Naming

Follow the language.

| Thing | Rule |
|-------|------|
| Interfaces | `I` prefix only if the repo uses it |
| Domain exception | One typed exception per business refusal. In a language without exceptions, one error type or variant per refusal, matched with the language's error matching. |
| Domain event | Past tense: `OrderPlaced` |
| Integration event | Past tense, lives in `contract` |
| Repository port | Named after the aggregate, lives in `application/<aggregate>/ports` |
| Tests | A plain English sentence in the framework's naming style |

## Scaffold

Default scaffold, phase 4: the domain folder and its test folder.

1. Create the folders from the tree.
2. Create the module file the language needs: `package.json` or `index.ts`, `__init__.py`, a `package` clause, a `pom.xml` or `build.gradle` module. If unsure of the command, ask once.
3. Create `UBIQUITOUS-LANGUAGE.md` and `DOMAIN-HISTORY.jsonl` in the domain folder.
4. Create one failing test per selected test case, in the repo's framework.
5. Register the test folder with the test runner if the runner needs it.
6. Run the tests once and report the result in one line.

`application`, `infrastructure`, `api` and `contract` only when the developer chooses them.

## Test skeleton

One test file per aggregate or domain service. One test per selected test case. The body fails until written. Example in pseudocode:

```
test "order is rejected when merchant is unavailable":
    fail("Not implemented. Write when OrderDomainService.processOrder exists.")
```

## Ledger validation test

Write it in the repo's test framework. It reads `DOMAIN-HISTORY.jsonl` by a path relative to the test file, parses each line as JSON and checks: required fields present; `kind` is one of the 13 kinds; `id` is unique and in order; `supersedes` points to an id in the same file; `at` is an ISO date. It does not check `related`; review mode does that across contexts.

## Developer-choice options

Ask once, record as `DecisionRecorded`, never report as a finding.

| Choice | Options |
|--------|---------|
| Reconstitution | The ORM writes fields through a hidden constructor; an explicit `reconstitute(...)` function; a plain mapping function from a persistence record |
| Repository methods | Collection style with change tracking; persistence style with explicit `update` or `save`; query methods on the same port |
| Dispatcher | Plain functions or classes wired by hand; any library |
| Envelope and outbox | Own outbox table plus a background publisher; any messaging library |
| Current time | Whatever the code does. Never ask, never a finding. |

## Writing a new profile

Copy this file to `references/stacks/<stack>.md` and keep these headings: Detect, Tree, Allowed dependencies, Boundary check, Naming, Scaffold commands, Test skeleton, Ledger validation test, Developer-choice options. Add a row to the stack table in `SKILL.md`. Write only what was tried on a real repo.
