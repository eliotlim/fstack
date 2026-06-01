# CLAUDE.md

For an AI working *on* fstack source, not for users.

## What it is

Lightweight skill framework for full-stack product engineers. Built on Claude Code skills.

- Profile-driven recommendations. 5 numeric dimensions + tech stack + code preferences.
- **Implicit multi-mode.** Observations cluster by (repo, hour-band). Each cluster is a subprofile fstack recognizes. Auto-switches on `/fstack` invocation when context matches a cluster.
- Explicit mode is opt-in: user creates named manual subprofiles or pins one.
- Tight surface. Seven skills.
- No bun, no node. Bash + jq.

Pairs with [gstack](https://github.com/). No shared state.

## Layout

```
/Users/eliot/Workspaces/fstack/
├── README.md           # users
├── CLAUDE.md           # this
├── VERSION             # 0.3.0
├── LICENSE
├── install             # dev / prod installer
├── config.template.json
├── bin/
│   ├── fstack-config   # get/set + get-active/set-active + migrate
│   ├── fstack-profile  # subprofile management
│   ├── fstack-observe  # signal log + context capture
│   └── fstack-cluster  # cluster observations into inferred subprofiles
├── fstack/             # /fstack: router, auto-cluster, auto-switch, setup
├── fstack-profile/     # show / edit / pin / unpin / list / use / create / cluster
├── fstack-stack/
├── fstack-skill/
├── fstack-scaffold/
├── fstack-api/
└── fstack-schema/
```

## State

`~/.fstack/`:

- `config.json`: versioned, subprofiled.
- `observations/sessions.jsonl`: append-only signal log (single file, all observations across all subprofiles).
- `observations/_legacy/`: per-subprofile logs from v0.2 (untouched, kept for safety).
- `skills/<name>/SKILL.md`: user-created custom skills.

## Schema (v0.3.0)

```json
{
  "version": "0.3.0",
  "install": { ... },
  "profile": {
    "active": "<key>",
    "pinned": "<key>|null",
    "auto_switch": true,
    "last_cluster_at": "...",
    "subprofiles": {
      "<key>": {
        "label": "...",
        "description": "...|null",
        "origin": "manual|inferred",
        "developer":   { "risk_tolerance": 0..1|null, "...": "..." },
        "preferences": { "code_style": "...|null", "...": "..." },
        "stack":       { "language": [], "...": [], "declared_at": "...|null" },
        "inferred":    { "sample_count": N, "rolling": {} },
        "examples":    [],
        "cluster_meta": { "repo": "...", "hour_band": "morning|afternoon|evening|late-night" } | null,
        "declared_at": "...|null",
        "created_at":  "...",
        "last_used_at": "..."
      }
    }
  },
  "custom_skills": [{ "name": "...", "description": "...", "path": "..." }]
}
```

## Observation schema

```json
{
  "ts": "...",
  "profile_at_log": "<active subprofile when logged>",
  "skill": "fstack-scaffold|fstack-api|fstack-schema|...",
  "dimension": "risk_tolerance|bias_for_action|scope_appetite|test_rigor|architecture_care|null",
  "signal": 0.0..1.0|null,
  "weight": 1,
  "annotation": "short human-readable text",
  "context_note": "(legacy free-form context, deprecated, prefer annotation)",
  "context": {
    "repo": "<git toplevel basename>|null",
    "branch": "<git current branch>|null",
    "cwd": "<process cwd>",
    "hour": 0..23,
    "weekday": 1..7
  }
}
```

`annotate` entries set `dimension: null, signal: null, weight: 0` — they exist for cluster examples only.

## Clustering

`fstack-cluster cluster` groups `sessions.jsonl` entries by `(context.repo, context.hour | band)` where band is `late-night | morning | afternoon | evening`. Clusters crossing `--threshold N` (default 3) become inferred subprofiles. Each gets:

- `developer.*`: weighted means of signals for known dimensions, others stay null.
- `inferred.rolling`: same as developer.
- `examples`: up to 3 unique annotations.
- `cluster_meta`: { repo, hour_band, last_clustered_at }.

Existing manual subprofiles are skipped — never overwritten by clustering. Existing inferred ones get updated in place (same key = `<repo>-<band>`).

`fstack-cluster match --repo X --hour H` returns the best subprofile key. Prefers exact (repo, band) match, then (repo, any band), then empty.

Auto-cluster trigger lives in `/fstack` preamble:

- Only if `observations >= 5`.
- Only if `last_cluster_at` is null OR older than 24h.

Auto-switch trigger:

- Only if `auto_switch == true`.
- Only if `pinned == null`.
- Only if `match` returns a non-empty key different from `active`.

## Conventions

1. **No bun, no node.** Bash + jq.
2. **Skills are stateless.** Durable state through `fstack-config` / `fstack-observe` / `fstack-cluster`. Don't write `config.json` directly.
3. **Skills target active by default.** Use `get-active` / `set-active`. Specify a key only when explicitly inspecting another.
4. **Annotations matter.** Domain skills log a short human-readable annotation per turn. Without annotations, inferred clusters have no examples — they're abstract numbers.
5. **Preambles stay tight.** No telemetry, no upgrade checks, no MCP probing.
6. **Confirm before writing declared values.** Free-form input + direct mutation is a trust boundary.
7. **Stable bin path.** Skills resolve `_FSTACK_BIN` by following the SKILL.md symlink (dev) or falling back to common paths.

## When to write into which

| Action | Destination |
|--------|-------------|
| User declared a value via UI | `set-active developer.<dim>` / `set-active preferences.<key>` |
| Domain skill logging a choice | `fstack-observe log <dim> <sig> --annotation "..."` |
| Sub-skill emitting a contextual note (no signal) | `fstack-observe annotate "..."` |
| New manual subprofile | `fstack-profile create <key>` |
| Clustering output | (automatic, do not touch from skills) |

## Voice (for fstack itself)

Skills are instructions for Claude. Read like a senior eng leaving notes for another senior eng.

- Short. Direct. Numbered lists for steps.
- No em dashes. No AI vocab.
- Imperatives: "Read X. Write Y." not "Your job is to read X."
- When auto-switching subprofile, say so in one line, don't apologize, don't ask.

## Migration policy

`_maybe_migrate` runs on every read after `exists`. Each version step is idempotent and additive — no destructive changes to fields that exist.

Order:

1. `_migrate_v02`: wrap flat profile into subprofiles. Detect by `profile.subprofiles == null`.
2. `_migrate_v03`: add pinned/auto_switch/origin/examples/cluster_meta. Detect by `profile.auto_switch == null`. Consolidate per-subprofile observation logs.

Future steps follow the same pattern.

## Smoke test

```bash
B=/Users/eliot/Workspaces/fstack/bin

# Seed synthetic observations across two contexts
mkdir -p ~/.fstack/observations
cat > ~/.fstack/observations/sessions.jsonl <<'EOF'
{"ts":"2026-05-25T14:00:00Z","profile_at_log":"default","skill":"fstack-api","dimension":"architecture_care","signal":0.85,"weight":1,"annotation":"API with strict schema","context":{"repo":"acme","hour":14}}
{"ts":"2026-05-25T15:00:00Z","profile_at_log":"default","skill":"fstack-schema","dimension":"test_rigor","signal":0.8,"weight":1,"annotation":"Migration with FK","context":{"repo":"acme","hour":15}}
{"ts":"2026-05-25T16:00:00Z","profile_at_log":"default","skill":"fstack-scaffold","dimension":"scope_appetite","signal":0.3,"weight":1,"annotation":"Smaller scaffold","context":{"repo":"acme","hour":16}}
{"ts":"2026-05-28T23:00:00Z","profile_at_log":"default","skill":"fstack-api","dimension":"risk_tolerance","signal":0.9,"weight":1,"annotation":"Quick endpoint, no validation","context":{"repo":"side","hour":23}}
{"ts":"2026-05-29T01:00:00Z","profile_at_log":"default","skill":"fstack-scaffold","dimension":"test_rigor","signal":0.15,"weight":1,"annotation":"No tests, just see if it works","context":{"repo":"side","hour":1}}
{"ts":"2026-05-29T01:30:00Z","profile_at_log":"default","skill":"fstack-schema","dimension":"architecture_care","signal":0.2,"weight":1,"annotation":"SQLite, denormalized","context":{"repo":"side","hour":1}}
EOF

$B/fstack-cluster cluster
$B/fstack-profile list             # two inferred should appear
$B/fstack-cluster match --repo acme --hour 14         # acme-afternoon
$B/fstack-cluster match --repo side --hour 1          # side-late-night
$B/fstack-profile pin acme-afternoon
$B/fstack-profile list             # [pinned] shown
$B/fstack-profile unpin
```

Real test: invoke `/fstack` in a fresh session in any git repo — preamble should announce active and switch if context matches a cluster.

## Don't

- Don't add a CLI entry point.
- Don't add telemetry / phone-home.
- Don't depend on gstack.
- Don't grow beyond ~7 skills in v0.x.
- Don't let clustering overwrite manual subprofiles.
- Don't switch silently — always print the SWITCHED line.
- Don't add comments to skill bodies explaining what each step does.
