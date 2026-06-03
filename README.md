# fstack

Full-stack skills framework for those who build. Pairs with [gstack](https://github.com/).

fstack profiles how you work and what you build with, then shapes recommendations across scaffolding, API design, and schema work to match.

You're not one developer. **You have archetypes.** Production work is deliberate and balanced. A spike is fluid and ambitious. A late-night side project is fluid and focused. None of these is "better" than the others. fstack notices which archetype your current work fits and switches accordingly.

The archetype is the identity. `(repo, branch)` is just where it shows up — the same archetype can span many repos and many branches.

## Install

```bash
./install --dev       # symlink SKILL.md into ~/.claude/skills/
./install --prod      # copy them
./install --status
./install --uninstall
```

Needs `jq` (`brew install jq`).

After install, open a new Claude Code session and type `/fstack`.

## Skills

| Skill | Does |
|-------|------|
| `/fstack` | Router + setup. Auto-clusters and auto-switches archetype per context. |
| `/fstack-profile` | Show, edit, switch, pin archetypes. |
| `/fstack-stack` | Declare tech stack on the active subprofile. |
| `/fstack-skill` | Make your own skills. |
| `/fstack-scaffold` | Scaffold a feature. Logs annotated observations. |
| `/fstack-api` | Stub an endpoint. Logs annotated observations. |
| `/fstack-schema` | Model + migration. Logs annotated observations. |

## Seven dimensions

Each dimension is a spectrum between two legitimate working styles. Pick where you actually live, not where you think you "should" be.

| Dimension | Low end | High end |
|-----------|---------|----------|
| `risk_tolerance` | cautious, measured | bold, experimental |
| `bias_for_action` | deliberate, plan-first | action-oriented, shipping |
| `scope_appetite` | focused, narrow | ambitious, comprehensive |
| `test_rigor` | lean, streamlined | rigorous, thorough |
| `architecture_care` | pragmatic, direct | principled, structured |
| `detail_preference` | terse, signal-heavy | thorough, explanatory |
| `autonomy` | collaborative, consultative | autonomous, self-directed |

The last two are imported from gstack's psychographic. They fit full-stack work — different archetypes call for different communication density and different levels of delegation.

## Archetypes

Observations roll up into work contexts `(repo, branch_norm)`. Each context projects onto two axes:

- **discipline** = avg of `(1 − risk_tolerance, test_rigor, architecture_care)`. Fluid ↔ deliberate.
- **ambition** = avg of `(bias_for_action, scope_appetite)`. Focused ↔ ambitious.

Both axes bin `low / mid / high`, giving 9 cells with archetype names:

| | focused | balanced | ambitious |
|---|---|---|---|
| **fluid** | exploration | iteration | sprint |
| **balanced** | polish | shipping | expansion |
| **deliberate** | maintenance | production | hardening |

Each cell that crosses the observation threshold becomes a subprofile keyed by archetype name. You reference it directly:

```
/fstack-profile use production
/fstack-profile show sprint
/fstack-profile pin exploration
```

Same archetype can span many `(repo, branch)` combinations:

```
/fstack-profile show production
ARCHETYPE: production (inferred, active)
  deliberate x balanced

DIMENSIONS
  risk_tolerance:    0.18 (cautious)
  test_rigor:        0.83 (rigorous)
  architecture_care: 0.88 (principled)
  ...

SEEN IN
  acme:main @ afternoon  (8)
  acme:main @ morning    (4)
  client-foo:main @ afternoon  (3)

EXAMPLES
  - FK + ON DELETE CASCADE
  - locked schema before merge
  - full integration tests
```

## Auto-switch

`/fstack` preamble:

1. Re-cluster if stale (> 24h) and there are new observations.
2. Read current context: `(repo, branch_norm, hour)`.
3. Tiered lookup against each archetype's `context_distribution`:
   - exact match `<repo>:<branch_norm> @ <band>`
   - same repo + branch
   - same repo
4. Pick the highest-scoring archetype. Empty if nothing matches.
5. Switch if match differs from active, `auto_switch == true`, and `pinned == null`.

Print one line: `SWITCHED: default -> production (matched on acme:main @ afternoon)`.

## Overrides

- `/fstack-profile pin <archetype>` (or just `pin` for current) locks the active subprofile.
- `/fstack-profile unpin` clears.
- `fstack-config set profile.auto_switch false` disables entirely.
- `/fstack-profile create <name>` creates a manual subprofile. Manual names that collide with archetype names take precedence (clustering skips them).

## Config

`~/.fstack/config.json`. Versioned. Auto-migrates.

```bash
~/.fstack-bin/fstack-config show | jq
```

Schema (v0.5.0):

```json
{
  "version": "0.5.0",
  "install": { "mode": "dev|prod", "...": "..." },
  "profile": {
    "active": "<archetype-or-manual-key>",
    "pinned": "<key>|null",
    "auto_switch": true,
    "last_cluster_at": "...",
    "subprofiles": {
      "<key>": {
        "label": "...",
        "description": "...",
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
        "preferences": { "...": "..." },
        "stack":       { "...": "..." },
        "inferred":    { "...": "..." },
        "examples":    [],
        "context_distribution": { "<repo>:<branch_norm> @ <band>": count },
        "cluster_meta": {
          "method": "behavior-2d-v3",
          "archetype": "production",
          "bin_key": "deliberate-balanced",
          "discipline_bin": "deliberate",
          "ambition_bin": "balanced",
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
  "custom_skills": []
}
```

Observations: `~/.fstack/observations/sessions.jsonl`. One JSONL stream.

## Migration

Auto. First read of an old config upgrades:

- v0.1 → v0.2: wrap flat profile + top-level stack into `default` subprofile.
- v0.2 → v0.3: add `pinned`, `auto_switch`. Mark existing subprofiles `origin: "manual"`. Consolidate observation logs.
- v0.3 → v0.4: drop time-keyed inferred subprofiles; behavior-first re-cluster on next `/fstack`.
- v0.4 → v0.5: add `detail_preference` + `autonomy` dimensions. Drop inferred subprofiles (key format moved from bin-name to archetype-name). Re-cluster.

Manual subprofiles preserved at every step.

## Custom skills

```
/fstack-skill
> Make a skill that runs my pre-deploy checklist.
```

Skills go to `~/.fstack/skills/<name>/SKILL.md`, symlinked into `~/.claude/skills/<name>/SKILL.md`. Shared across subprofiles.

## With gstack

Designed to compose.

- **fstack**: what you build (scaffold, api, schema, profile).
- **gstack**: how you operate (browse, qa, ship, review, deploy, retro).

No shared state.

## License

MIT
