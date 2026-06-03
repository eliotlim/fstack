# CLAUDE.md

For an AI working *on* fstack source, not for users.

## What it is

Lightweight skill framework for full-stack product engineers. Built on Claude Code skills.

- 7-dimensional profile (risk_tolerance, bias_for_action, scope_appetite, test_rigor, architecture_care, detail_preference, autonomy). Each spectrum has two legitimate ends. No "high = better."
- **Archetype-keyed clustering.** Observations roll up into work contexts `(repo, branch_norm)`. Each context lives in a 2-D space (discipline x ambition). Bin cells with enough observations become subprofiles keyed by archetype name (`production`, `sprint`, `exploration`, etc.).
- Explicit subprofiles are opt-in via `create`. Manual key collisions with archetype names take precedence.
- **Auto-suggest, not auto-switch.** When match differs from active, /fstack prints a SUGGEST line and asks the user before switching.
- **Show calibration.** Domain skills print a CALIBRATION block before proposing so the active dim values are visible.
- Tight surface. Seven skills.
- No bun, no node. Bash + jq.

Pairs with [gstack](https://github.com/). No shared state.

## Layout

```
/Users/eliot/Workspaces/fstack/
├── README.md           # users
├── CLAUDE.md           # this
├── VERSION             # 0.6.0
├── LICENSE
├── install
├── config.template.json
├── bin/
│   ├── fstack-config   # get/set/get-active/set-active + safe jq paths + migrations
│   ├── fstack-profile  # subprofile UI + calibrate helper
│   ├── fstack-observe  # signal log + forget + auto context capture
│   └── fstack-cluster  # archetype-keyed clustering + hour-band smoothed match
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

- `config.json`: versioned, subprofiled, archetype-keyed. Stamped with `last_migrated_version`.
- `observations/sessions.jsonl`: append-only signal log.
- `observations/_legacy/`: v0.2 per-subprofile logs (untouched).
- `skills/<name>/SKILL.md`: user-created custom skills.

## Dimensions

Seven dimensions at `developer.*`. Each is 0..1. Both ends are legitimate.

| Dimension | Low end | High end |
|-----------|---------|----------|
| `risk_tolerance` | stability | speed |
| `bias_for_action` | deliberate | action |
| `scope_appetite` | focused | ambitious |
| `test_rigor` | lean | rigorous |
| `architecture_care` | pragmatic | principled |
| `detail_preference` | detail-oriented | big-picture |
| `autonomy` | seek-permission | ask-forgiveness |

The last two come from gstack's plan-tune psychographic. They fit full-stack work because archetypes vary on communication density and delegation comfort.

## Cluster axes

Only 5 dimensions feed the cluster axes. detail_preference + autonomy are profile-only for v0.6.

- **discipline** = avg of `(1 − risk_tolerance, test_rigor, architecture_care)`. Low (< 0.4) = fluid, mid = balanced, high (≥ 0.65) = deliberate.
- **ambition** = avg of `(bias_for_action, scope_appetite)`. Low = focused, mid = balanced, high = ambitious.

9 cells. Each maps to an archetype name:

| | focused | balanced | ambitious |
|---|---|---|---|
| **fluid** | exploration | iteration | sprint |
| **balanced** | polish | shipping | expansion |
| **deliberate** | maintenance | production | hardening |

## Schema (v0.6.0)

```json
{
  "version": "0.6.0",
  "last_migrated_version": "0.6.0",
  "install": { ... },
  "profile": {
    "active": "<key>",
    "pinned": "<key>|null",
    "auto_suggest": true,
    "last_cluster_at": "...",
    "subprofiles": {
      "<key>": {
        "label": "...",
        "description": "...|null",
        "origin": "manual|inferred",
        "developer": {
          "risk_tolerance": 0..1|null,
          "bias_for_action": 0..1|null,
          "scope_appetite": 0..1|null,
          "test_rigor": 0..1|null,
          "architecture_care": 0..1|null,
          "detail_preference": 0..1|null,
          "autonomy": 0..1|null
        },
        "preferences": { "code_style": "...|null", "...": "..." },
        "stack":       { "language": [], "...": [], "declared_at": "...|null" },
        "inferred":    { "sample_count": N, "rolling": {} },
        "examples":    [],
        "context_distribution": { "<repo>:<branch_norm> @ <band>": count },
        "cluster_meta": {
          "method": "behavior-2d-v<VERSION>",
          "archetype": "production",
          "bin_key": "deliberate-balanced",
          "discipline_bin": "fluid|balanced|deliberate",
          "ambition_bin": "focused|balanced|ambitious",
          "discipline_mean": 0..1,
          "ambition_mean": 0..1,
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

## Clustering algorithm

1. Filter observations: drop annotate-only, drop older than `--max-age-days` (60).
2. Aggregate per `(repo, branch_norm)`. branch_norm: `main/master/develop/trunk → main`, `spike/exp/experiment[-/]? → spike`, `feature/feat[-/]? → feature`, `fix/bugfix/hotfix[-/]? → fix`, `release/rc[-/]? → release`, `chore/refactor[-/]? → chore`, else `other`. Bare `spike` and `spike-foo` both normalize to `spike`.
3. Per context, compute weighted mean per dimension.
4. Project to `(discipline, ambition)`. Fall back to 0.5 if no relevant dim.
5. Bin each axis to 3 levels. `bin_key = "<discipline_bin>-<ambition_bin>"`. Look up archetype name.
6. Group contexts by archetype. Cells with `observation_count < --threshold` (default 3) dropped.
7. Emit each cluster as an inferred subprofile keyed by archetype name. Compute centroid, examples, context_distribution.
8. Manual subprofiles preserved. If a manual entry has the same key as an archetype, clustering skips it and prints `skipped (manual collision): <name>`.

## Matching algorithm

Given current `(repo, branch_norm, hour_band)`:

1. Build candidate keys: `<repo>:<branch_norm> @ <band>` plus the two adjacent bands.
2. Score each inferred subprofile: `exact_count + 0.5 * neighbor_band_counts`.
3. Tier fallback: exact (smoothed) → same repo+branch any band → same repo any branch/band. Tiebreaker: observation count.
4. Return top archetype, or empty if all zero.

## Auto-suggest flow

When match differs from active, /fstack prints `SUGGEST: <archetype> (matches <repo>:<branch>, current <active>)`. The skill body then asks via AskUserQuestion (switch / keep / pin current). fstack only mutates `profile.active` on explicit yes.

## Show calibration

`fstack-profile calibrate <skill>` prints a `CALIBRATION` block listing the dims that skill cares about with their current values + band labels. Domain skills (api, schema, scaffold) call this in their preamble and re-print on the proposal so the active subprofile's effect is visible at every step.

## Conventions

1. **No bun, no node.** Bash + jq.
2. **Skills are stateless.** Durable state through `fstack-config` / `fstack-observe` / `fstack-cluster` / `fstack-profile`.
3. **Skills target active by default.** Use `get-active` / `set-active`.
4. **Annotations matter.** Domain skills log a short human-readable annotation per turn. Cluster examples come from these.
5. **Behavior is the cluster axis; context is descriptive.** Repo / branch / hour go into `context_distribution`, never into the cluster key.
6. **Both ends of every dimension are legitimate.** When asking the user where they fall, do not imply one end is "right."
7. **Preambles stay tight.** No telemetry, no upgrade checks.
8. **Confirm before writing declared values.** Free-form input + direct mutation is a trust boundary.
9. **Suggest, don't switch.** Auto-mode never mutates active without the user's yes.
10. **Always show calibration.** The user must see which dim values shape the proposal.

## Voice (for fstack itself)

Skills are instructions for Claude. Read like a senior eng leaving notes for another senior eng.

- Short. Direct. Numbered lists.
- No em dashes. No AI vocab.
- Imperatives. "Read X. Write Y."
- When you suggest a switch, say so in one line. Don't apologize.

## Migration policy

`_maybe_migrate` runs on every read after `exists`. Short-circuits if `last_migrated_version` matches the binary's target version. Otherwise additive + idempotent.

Order:

1. `_migrate_v02`: wrap flat profile into subprofiles.
2. `_migrate_v03`: add pinned/auto_switch/origin. Consolidate observation logs.
3. `_migrate_v04`: drop time-keyed `<repo>-<band>` inferred subprofiles.
4. `_migrate_v05`: add detail_preference + autonomy. Drop bin-keyed inferred subprofiles.
5. `_migrate_v06`: rename `auto_switch` → `auto_suggest`. Reset `last_cluster_at` (forces re-cluster against the new method-version tag).

Stamps `last_migrated_version` at the end so subsequent reads skip the whole chain.

## Smoke test

```bash
B=/Users/eliot/Workspaces/fstack/bin
export FSTACK_HOME=/tmp/fstack-smoke/.fstack
rm -rf /tmp/fstack-smoke && mkdir -p "$FSTACK_HOME/observations"
$B/fstack-config init /Users/eliot/Workspaces/fstack >/dev/null
cat > "$FSTACK_HOME/observations/sessions.jsonl" <<'EOF'
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
# Expect: production (acme:main), sprint (acme:spike), iteration (voice:main)

$B/fstack-cluster match --repo acme --branch main --hour 14
# production
$B/fstack-cluster match --repo acme --branch spike-foo --hour 16
# sprint
$B/fstack-cluster match --repo voice --branch main --hour 23
# iteration

# Show calibration
$B/fstack-profile calibrate api production

# Forget by filter
$B/fstack-observe forget --skill fstack-api
```

## Don't

- Don't add a CLI entry point.
- Don't add telemetry / phone-home.
- Don't depend on gstack.
- Don't grow beyond ~7 skills in v0.x.
- Don't let clustering overwrite manual subprofiles.
- Don't put context into the cluster key (that was v0.3's mistake).
- Don't imply one end of a dimension is "right" (that was the pre-v0.5 framing flaw).
- Don't switch silently (that was the pre-v0.6 mistake; only suggest).
- Don't hide the calibration (the user must see which archetype shaped the output).
- Don't add comments to skill bodies.
