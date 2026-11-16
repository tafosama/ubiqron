# ubiqron

A Claude Code skill for Domain-Driven Design that keeps the model inside the codebase.

ubiqron helps one developer create, review and evolve bounded contexts in a monorepo. It writes three things next to the code and keeps them in step:

| File | What it holds |
|------|---------------|
| `<Context>.Domain/UBIQUITOUS-LANGUAGE.md` | The current meaning of every term, aggregate, method and event, and the list of use cases |
| `<Context>.Domain/DOMAIN-HISTORY.jsonl` | An append-only history of every model change, use cases included |
| `ubiqron/CONTEXT-MAP.md` at the root of your repo | Which contexts exist, who feeds whom, and where the contract lives |

## What it does

- **New context.** Four short phases: scope, aggregates, behaviour and tests, scaffold. Each phase is a proposal the developer picks from. Nothing is written before the selection.
- **Review.** Checks an existing context against fifteen fixed rules (U1 to U15) and returns a findings list. No score.
- **Maintain.** Rename, deprecate, add a rule, a method or a use case, move an aggregate, change a relationship. Answer "what did X mean on date D" and "which use cases does Order have" from the files, without scanning the code.
- **Draw.** Mermaid diagrams as projections of the code and the ledger: context map, sequence, state, flow, overview, timeline.

Architecture: Clean Architecture with Hexagonal ports and adapters. Domain holds the model, Application owns the ports, Infrastructure and Api implement them. Contexts reference each other only through a published-language module.

Stack: the core is language-agnostic. A stack profile adds names, commands and the test framework. .NET has a full profile. Everything else uses the generic profile, which reads the code first and asks two questions once.

## Install

With the [skills CLI](https://github.com/vercel-labs/skills):

```bash
# pick the agent and the scope interactively
npx skills add tafosama/ubiqron

# Claude Code, this project only: installs to .claude/skills/ubiqron
npx skills add tafosama/ubiqron --skill ubiqron -a claude-code

# Claude Code, every project: installs to ~/.claude/skills/ubiqron
npx skills add tafosama/ubiqron --skill ubiqron -a claude-code -g
```

Or by hand:

```bash
git clone https://github.com/tafosama/ubiqron.git
mkdir -p ~/.claude/skills
cp -r ubiqron/skills/ubiqron ~/.claude/skills/ubiqron
```

For one project only, copy to `<your repo>/.claude/skills/ubiqron` instead.

Then ask your agent: "create a bounded context for ..." or "review the Order context".

If another Domain-Driven Design skill is installed, remove or disable it first. Two skills that react to the same words, such as "bounded context" or "aggregate", get in each other's way.

## Layout

```
skills/ubiqron/
  SKILL.md                     entry point: rules, modes, developer choices, ledger in one screen
  references/
    new-context-flow.md        the four phases
    review-mode.md             checks and finding format
    strategic-design.md        bounded contexts, subdomain type, nine patterns, published language
    context-map.md             template of the context map and its ledger link
    tactical-patterns.md       aggregates, entities, value objects, events, services, exceptions, factories
    application-layer.md       handlers, ports, processes, publishing between contexts
    glossary-and-ledger.md     glossary and use-case table, ledger schema, 15 kinds, dual-write, validation test
    diagrams.md                six Mermaid kinds, budgets, stale check
    stacks/dotnet.md           .NET profile
    stacks/generic.md          fallback profile and the template for new profiles
  examples/diagrams/           one sample per diagram kind, with a preview.html
```

The skill folder is self-contained. Nothing outside `skills/ubiqron/` is needed at run time.

## Principles

- Plain English everywhere: short sentences, DDD terms only.
- Fixed rules are about placement and ownership. Implementation choices such as repository methods, dispatch library, outbox shape or reconstitution style stay with the developer.
- Every modeling action appends a ledger line. The glossary, the context map and the diagrams are projections.

## Credits

The lean `SKILL.md` plus `references/` structure, the three subdomain tests and the living-glossary table follow the `domain-driven-design` skill in [wondelai/skills](https://github.com/wondelai/skills), MIT licensed.

## License

MIT. See [LICENSE](LICENSE).
