# CLAUDE.md

For an AI working *on* fstack source, not for users.

## What it is

Lightweight skill framework for full-stack product engineers. Built on Claude Code skills.

- Profile-driven recommendations. 5 numeric dimensions, tech stack, code preferences.
- **Multiple subprofiles.** Users have modes (prod, prototype, learning). Each carries its own dimensions, stack, preferences, observations.
- Implicit observation + explicit declaration. Reconcile per subprofile.
- Tight surface. Seven skills. Don't sprawl.
- No bun, no node. Bash + jq.

Pairs with [gstack](https://github.com/). No shared state.

## Layout

```
/Users/eliot/Workspaces/fstack/
├── README.md           # users
├── CLAUDE.md           # this
├── VERSION             # 0.2.0
├── LICENSE
├── install             # dev / prod installer
├── config.template.json
├── bin/
│   ├── fstack-config   # get/set + get-active/set-active + migrate
│   ├── fstack-profile  # multi-subprofile management
│   └── fstack-observe  # per-subprofile signal log
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

- `config.json`: main config (versioned, subprofile-shaped).
- `observations/<subprofile-key>.jsonl`: append-only signal log per subprofile.
- `skills/<name>/SKILL.md`: user-created custom skills (top-level, shared across subprofiles).

## Schema (v0.2.0)

```json
{
  "version": "0.2.0",
  "install": { ... },
  "profile": {
    "active": "<key>",
    "subprofiles": {
      "<key>": {
        "label": "...",
        "description": "...|null",
        "developer":   { "risk_tolerance": 0..1|null, "...": "..." },
        "preferences": { "code_style": "...|null", "...": "..." },
        "stack":       { "language": [], "...": [], "declared_at": "...|null" },
        "inferred":    { "sample_count": N, "rolling": {}, "last_observation_at": "..." },
        "declared_at": "...|null",
        "created_at":  "...",
        "last_used_at": "..."
      }
    }
  },
  "custom_skills": [{ "name": "...", "description": "...", "path": "..." }]
}
```

## Conventions

1. **No bun, no node.** Bash + jq.
2. **Skills are stateless.** All durable state goes through `fstack-config` / `fstack-observe`. Don't write `config.json` from a skill body; go through the binary so writes stay atomic.
3. **Skills target the active subprofile by default.** Use `get-active` / `set-active` / `set-active-json`. Hardcode a subprofile key only when explicitly inspecting another.
4. **Preambles stay tight.** No telemetry, no upgrade checks, no MCP probing.
5. **Confirm before writing declared values.** Free-form input + direct mutation is a trust boundary.
6. **Stable bin path.** Skills resolve `_FSTACK_BIN` by following the SKILL.md symlink (dev) or falling back to common paths.

## Active subprofile, the rule

Reads + writes default to `profile.subprofiles[profile.active]`. Skills:

- `fstack-config get-active developer.risk_tolerance` → reads from active
- `fstack-config set-active developer.risk_tolerance 0.8` → writes to active
- `fstack-config set-active-json stack.language '["TypeScript"]'` → writes JSON to active
- `fstack-profile vibe` → vibe of active
- `fstack-profile vibe acme-prod` → vibe of a specific subprofile
- `fstack-observe log <dim> <sig>` → logs to `observations/<active>.jsonl`
- `fstack-observe rollup --profile <key>` → rollup a specific subprofile

When the user's message implies a non-active subprofile ("for prod I'm careful"), don't silently write to active. Either offer to switch, or ask which subprofile to write to.

## Add a skill

1. Create `<skill-name>/SKILL.md` with frontmatter.
2. Add `<skill-name>` to `SKILLS` in `install`.
3. Run `./install --dev` again.
4. Update README's skill table.

## Add a profile dimension

1. Update `config.template.json` under `profile.subprofiles.default.developer` and the migration jq in `fstack-config _maybe_migrate`.
2. Update `fstack-profile show`, `dimensions`, `vibe`.
3. Update `fstack-observe gap`.
4. Update `/fstack` setup question.
5. Update `/fstack-profile` mappings.
6. Bump VERSION.

## Migration policy

`fstack-config _maybe_migrate` runs on every read after `exists`. Idempotent. v0.1 → v0.2 wraps old flat shape into `default` subprofile and relocates the old `observations.jsonl`. Future migrations follow the same pattern: detect on read, rewrite atomically, log to stderr.

## Don't

- Don't add a CLI entry point. fstack is invoked via `/fstack`.
- Don't add telemetry, analytics, or phone-home. Local-first.
- Don't depend on gstack. Compatible, not coupled.
- Don't pass ~7 skills in v1. Prove the loop, then expand.
- Don't add cross-subprofile bleed (observations from one subprofile shouldn't change another's `inferred`).
- Don't add comments to skill bodies explaining what each step does. The skill IS instructions for Claude.

## Voice

Skills are instructions for Claude. Read like a senior eng leaving notes for another senior eng.

- Short. Direct. Numbered lists for steps.
- No em dashes. No AI vocab.
- Imperatives: "Read X. Write Y." not "Your job is to read X."
- Tables for mappings. Bullets for parallel. Numbers for sequence.

## Smoke test

```bash
./install --dev
B=/Users/eliot/Workspaces/fstack/bin
$B/fstack-config migrate
$B/fstack-profile list
$B/fstack-profile create test --label "Test" --clone-from default
$B/fstack-profile use test
$B/fstack-config set-active developer.risk_tolerance 0.2
$B/fstack-profile vibe                   # should differ from default
$B/fstack-observe log scope_appetite 0.7 --skill test --context smoke
$B/fstack-observe rollup
$B/fstack-observe gap
$B/fstack-profile use default
$B/fstack-profile remove test
```

Real test: invoke `/fstack` in a fresh session, walk setup, create a second subprofile.
