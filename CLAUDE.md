# CLAUDE.md

For an AI working *on* fstack source, not for users.

## What it is

Lightweight skill framework for full-stack product engineers. Built on Claude Code skills.

- Profile-driven recommendations. 5 dimensions + tech stack + code preferences.
- **Behavior-first clustering.** Observations roll up into work contexts `(repo, branch_norm)`. Each context lives in a 2-D behavior space (rigor × appetite). Bin cells with enough observations become inferred subprofiles. Repo + branch + hour are descriptive, used at match time.
- Explicit subprofiles are opt-in via `create`.
- Tight surface. Seven skills.
- No bun, no node. Bash + jq.

Pairs with [gstack](https://github.com/). No shared state.

## Layout

```
/Users/eliot/Workspaces/fstack/
├── README.md           # users
├── CLAUDE.md           # this
├── VERSION             # 0.4.0
├── LICENSE
├── install
├── config.template.json
├── bin/
│   ├── fstack-config   # get/set/get-active/set-active + migrations
│   ├── fstack-profile  # subprofile UI
│   ├── fstack-observe  # signal log + context auto-capture
│   └── fstack-cluster  # behavior-first clustering
├── fstack/
├── fstack-profile/
├── fstack-stack/
├── fstack-skill/
├── fstack-scaffold/
├── fstack-api/
└── fstack-schema/
```

## State

`~/.fstack/`:

- `config.json`: versioned, subprofiled.
- `observations/sessions.jsonl`: append-only signal log.
- `observations/_legacy/`: v0.2 per-subprofile logs (untouched, kept for safety).
- `skills/<name>/SKILL.md`: user-created custom skills.

## Schema (v0.4.0)

```json
{
  "version": "0.4.0",
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
        "context_distribution": { "<repo>:<branch_norm> @ <band>": count, "..." : "..." },
        "cluster_meta": {
          "method": "behavior-2d-v2",
          "rigor_bin": "loose|balanced|rigorous",
          "appetite_bin": "small|steady|bold",
          "rigor_mean": 0..1,
          "appetite_mean": 0..1,
          "context_count": N,
          "observation_count": N,
          "last_clustered_at": "..."
        } | null,
        "declared_at": "...|null",
        "created_at":  "...",
        "last_used_at": "..."
      }
    }
  },
  "custom_skills": [...]
}
```

## Observation schema

```json
{
  "ts": "...",
  "profile_at_log": "<active subprofile at log time>",
  "skill": "fstack-scaffold|fstack-api|fstack-schema|...",
  "dimension": "risk_tolerance|bias_for_action|scope_appetite|test_rigor|architecture_care|null",
  "signal": 0..1|null,
  "weight": 1,
  "annotation": "short human-readable text",
  "context_note": "(legacy free-form, deprecated)",
  "context": {
    "repo": "<git toplevel basename>|null",
    "branch": "<git current branch>|null",
    "cwd": "<process cwd>",
    "hour": 0..23,
    "weekday": 1..7
  }
}
```

## Clustering algorithm (behavior-first)

1. Filter observations: drop annotate-only (no dimension), drop older than `--max-age-days` (default 60).
2. Aggregate per work context `(repo, branch_norm)` where branch normalizes:
   - `main / master / develop / trunk` → `main`
   - `spike-* / exp-* / experiment-*` → `spike`
   - `feature-* / feat-*` → `feature`
   - `fix-* / bugfix-* / hotfix-*` → `fix`
   - `release-* / rc-*` → `release`
   - `chore-* / refactor-*` → `chore`
   - else → `other`
3. For each context, compute weighted mean per dimension across all its observations.
4. Project to 2-D:
   - `rigor` = avg of available `(1 - risk_tolerance, test_rigor, architecture_care)`. Fall back to 0.5.
   - `appetite` = avg of available `(bias_for_action, scope_appetite)`. Fall back to 0.5.
5. Bin each axis:
   - rigor: `< 0.4` loose, `< 0.65` balanced, else rigorous
   - appetite: `< 0.4` small, `< 0.65` steady, else bold
6. Group contexts by 2-D cell. Cluster key = `<rigor_bin>-<appetite_bin>`.
7. Filter cells with `observation_count < --threshold` (default 3).
8. Emit each cluster as inferred subprofile. Compute:
   - `developer.*` = centroid (mean of context means)
   - `examples` = up to 5 unique annotations
   - `context_distribution` = `{ "<repo>:<branch_norm> @ <band>": observation_count, ... }`
9. Existing manual subprofiles untouched. Stale inferred subprofiles (no longer in clusters) dropped.

Cluster keys are stable across reclusters (same cell → same key). Updates are upserts.

## Matching algorithm

At `/fstack` start, given current `(repo, branch_norm, hour_band)`:

1. Build candidate key `<repo>:<branch_norm> @ <band>`.
2. Tiered score per inferred subprofile:
   - exact match in `context_distribution` (strongest)
   - any `<repo>:<branch_norm> @ *` (same repo + branch, any hour)
   - any `<repo>:* @ *` (same repo, any branch/hour)
   - tiebreaker: `cluster_meta.observation_count`
3. Pick top-scoring cluster. If all scores zero, return empty (no switch).

## Conventions

1. **No bun, no node.** Bash + jq.
2. **Skills are stateless.** Durable state through `fstack-config` / `fstack-observe` / `fstack-cluster`.
3. **Skills target active by default.** Use `get-active` / `set-active`.
4. **Annotations matter.** Domain skills log a short human-readable annotation per turn. Without annotations, cluster examples are empty and inferred subprofiles feel abstract.
5. **Behavior is primary; context is descriptive.** Repo + branch + hour go into `context_distribution`, never into the cluster key.
6. **Preambles stay tight.** No telemetry, no upgrade checks, no MCP probing.
7. **Confirm before writing declared values.** Free-form input + direct mutation is a trust boundary.

## When to write into which

| Action | Destination |
|--------|-------------|
| User declared a value via UI | `set-active developer.<dim>` / `set-active preferences.<key>` |
| Domain skill logging a choice | `fstack-observe log <dim> <sig> --annotation "..."` |
| Sub-skill emitting a contextual note (no signal) | `fstack-observe annotate "..."` |
| New manual subprofile | `fstack-profile create <key>` |
| Clustering output | automatic — never written from skills |

## Voice (for fstack itself)

Skills are instructions for Claude. Read like a senior eng leaving notes for another senior eng.

- Short. Direct. Numbered lists.
- No em dashes. No AI vocab.
- Imperatives: "Read X. Write Y."
- When auto-switching subprofile, say so in one line, don't apologize, don't ask.

## Migration policy

`_maybe_migrate` runs on every read after `exists`. Idempotent + additive.

Order:

1. `_migrate_v02`: wrap flat profile into subprofiles.
2. `_migrate_v03`: add pinned/auto_switch/origin/examples. Consolidate per-subprofile logs.
3. `_migrate_v04`: drop time-based `<repo>-<band>` inferred subprofiles (wrong cluster axis). Re-cluster runs on next `/fstack`.

Future steps follow same pattern: detect by missing field or version mismatch, rewrite atomically, log to stderr.

## Smoke test (behavior-first)

```bash
B=/Users/eliot/Workspaces/fstack/bin
mkdir -p ~/.fstack/observations
# Three behavior patterns; same repo (acme) appears in two
cat > ~/.fstack/observations/sessions.jsonl <<'EOF'
{"ts":"2026-05-20T14:00:00Z","profile_at_log":"default","skill":"fstack-api","dimension":"risk_tolerance","signal":0.2,"weight":1,"annotation":"strict validation","context":{"repo":"acme","branch":"main","hour":14}}
{"ts":"2026-05-21T10:00:00Z","profile_at_log":"default","skill":"fstack-schema","dimension":"architecture_care","signal":0.85,"weight":1,"annotation":"FK + cascade","context":{"repo":"acme","branch":"main","hour":10}}
{"ts":"2026-05-22T15:00:00Z","profile_at_log":"default","skill":"fstack-scaffold","dimension":"test_rigor","signal":0.8,"weight":1,"annotation":"full e2e","context":{"repo":"acme","branch":"main","hour":15}}
{"ts":"2026-05-24T16:00:00Z","profile_at_log":"default","skill":"fstack-scaffold","dimension":"risk_tolerance","signal":0.85,"weight":1,"annotation":"trying new arch","context":{"repo":"acme","branch":"spike-routing","hour":16}}
{"ts":"2026-05-24T16:30:00Z","profile_at_log":"default","skill":"fstack-scaffold","dimension":"bias_for_action","signal":0.9,"weight":1,"annotation":"ship + iterate","context":{"repo":"acme","branch":"spike-routing","hour":16}}
{"ts":"2026-05-25T15:00:00Z","profile_at_log":"default","skill":"fstack-api","dimension":"scope_appetite","signal":0.85,"weight":1,"annotation":"3 patterns","context":{"repo":"acme","branch":"spike-auth","hour":15}}
{"ts":"2026-05-22T23:00:00Z","profile_at_log":"default","skill":"fstack-api","dimension":"test_rigor","signal":0.15,"weight":1,"annotation":"no tests","context":{"repo":"voice","branch":"main","hour":23}}
{"ts":"2026-05-23T23:30:00Z","profile_at_log":"default","skill":"fstack-schema","dimension":"architecture_care","signal":0.2,"weight":1,"annotation":"sqlite denormalized","context":{"repo":"voice","branch":"main","hour":23}}
{"ts":"2026-05-23T23:45:00Z","profile_at_log":"default","skill":"fstack-api","dimension":"risk_tolerance","signal":0.8,"weight":1,"annotation":"throw together","context":{"repo":"voice","branch":"main","hour":23}}
EOF
$B/fstack-cluster cluster --threshold 3
$B/fstack-profile list
# Expect: rigorous-steady (production) AND loose-bold (move-fast) AND loose-* for voice
$B/fstack-cluster match --repo acme --branch main --hour 14
# rigorous-steady
$B/fstack-cluster match --repo acme --branch spike-foo --hour 16
# loose-bold
```

## Don't

- Don't add a CLI entry point.
- Don't add telemetry / phone-home.
- Don't depend on gstack.
- Don't grow beyond ~7 skills in v0.x.
- Don't let clustering overwrite manual subprofiles.
- Don't put context (repo, branch, hour) into the cluster key; that was the v0.3 mistake.
- Don't switch silently — always print the SWITCHED line.
- Don't add comments to skill bodies.
