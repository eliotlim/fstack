# CLAUDE.md

For an AI working *on* fstack source, not for users.

## What it is

Lightweight skill framework for full-stack product engineers. Built on Claude Code skills.

- Profile-driven recommendations. 5 numeric dimensions, tech stack, code preferences.
- Implicit observation + explicit declaration. Reconcile by surfacing the gap.
- Tight surface. Seven skills in v1. Don't sprawl.
- No bun, no node. Bash + jq.

Pairs with [gstack](https://github.com/). No shared state.

## Layout

```
/Users/eliot/Workspaces/fstack/
├── README.md           # users
├── CLAUDE.md           # this
├── VERSION
├── LICENSE
├── install             # dev / prod installer
├── config.template.json
├── bin/
│   ├── fstack-config   # get/set JSON fields in ~/.fstack/config.json
│   ├── fstack-profile  # pretty-print
│   └── fstack-observe  # log signals, roll up
├── fstack/             # /fstack: router + setup
├── fstack-profile/
├── fstack-stack/
├── fstack-skill/
├── fstack-scaffold/
├── fstack-api/
└── fstack-schema/
```

Each skill is one `SKILL.md`. Frontmatter: `name`, `version`, `description`, `allowed-tools`, `triggers`.

## State

`~/.fstack/`:

- `config.json`: main config (versioned).
- `observations.jsonl`: append-only signal log.
- `skills/<name>/SKILL.md`: user-created custom skills.

## Conventions

1. **No bun, no node.** Bash + jq. Installer too.
2. **Skills are stateless.** All durable state goes through `fstack-config` / `fstack-observe`. Don't write to `config.json` from a skill body; go through the binary so writes stay atomic.
3. **Preambles stay tight.** No telemetry, no upgrade checks, no MCP probing. Lower overhead than gstack on purpose.
4. **Confirm before writing declared profile.** Free-form input + direct mutation is a trust boundary.
5. **Stable bin path.** Skills resolve `_FSTACK_BIN` by following the SKILL.md symlink (dev) or falling back to common paths. Don't hardcode dev paths.

## Add a skill

1. Create `<skill-name>/SKILL.md` with frontmatter.
2. Add `<skill-name>` to `SKILLS` in `install`.
3. Run `./install --dev` again.
4. Update README's skill table.

## Add a profile dimension

1. Update `config.template.json` under `profile.developer`.
2. Update `fstack-profile` (show, dimensions, vibe).
3. Update `fstack-observe gap` jq.
4. Update `/fstack` setup to ask the new question.
5. Update `/fstack-profile` mappings.
6. Bump VERSION. Consider migration in `fstack-config` for existing users.

## Versioning

`VERSION` file + `config.json.version`. Schema changes → `fstack-config` migrates on next read. Not done in v1 (no real users yet).

## Don't

- Don't add a CLI entry point. fstack is invoked via `/fstack` in Claude Code.
- Don't add telemetry, analytics, or phone-home. Local-first.
- Don't depend on gstack. Compatible, not coupled.
- Don't pass ~7 skills in v1. Prove the loop, then expand.
- Don't add comments to skill bodies explaining what each step does. The skill IS instructions for Claude. Inline commentary is noise.

## Voice

Skills are instructions for Claude. Read like a senior eng leaving notes for another senior eng.

- Short. Direct. Numbered lists for steps.
- No em dashes. No AI vocab (delve, crucial, robust, comprehensive, calibrate-as-prose, honor, ceremoniously).
- Imperatives: "Read X. Write Y." not "Your job is to read X."
- Tables for mappings. Bullets for parallel. Numbers for sequence.

## Smoke test

```bash
./install --dev
~/.fstack-bin/fstack-config show 2>/dev/null || /Users/eliot/Workspaces/fstack/bin/fstack-config show
/Users/eliot/Workspaces/fstack/bin/fstack-config set profile.developer.risk_tolerance 0.8
/Users/eliot/Workspaces/fstack/bin/fstack-profile show
/Users/eliot/Workspaces/fstack/bin/fstack-observe log scope_appetite 0.7 --skill test --context smoke
/Users/eliot/Workspaces/fstack/bin/fstack-observe rollup
/Users/eliot/Workspaces/fstack/bin/fstack-observe gap
```

Real test: invoke `/fstack` in a fresh session, walk setup.
