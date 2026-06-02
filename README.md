# fstack

Full-stack skills framework for those who build. Pairs with [gstack](https://github.com/).

fstack profiles how you work and what you build with, then shapes recommendations across scaffolding, API design, and schema work to match.

You're not one developer. **You have modes.** Some work is mature production. Some is early-stage that needs to move fast. Some is a late-night learning prototype.

fstack notices and adapts:

1. Every action gets logged with context (repo, branch, hour, weekday).
2. Observations roll up into **work contexts** keyed by `(repo, branch-type)`.
3. Each context lives in a 2-D behavior space: **rigor** × **appetite**. Contexts in the same cell cluster together.
4. Each cluster becomes a subprofile fstack can recognize and switch into.

The same repo can hold multiple modes. `acme:main` clusters with production-style work; `acme:spike` clusters with fast experiments. Behavior is the axis. Context is descriptive.

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
| `/fstack` | Router + setup. Auto-clusters and auto-switches per context. |
| `/fstack-profile` | Show, edit, switch, pin, list, cluster. |
| `/fstack-stack` | Declare tech stack on the active subprofile. |
| `/fstack-skill` | Make your own skills. |
| `/fstack-scaffold` | Scaffold a feature. Logs annotated observations. |
| `/fstack-api` | Stub an endpoint. Logs annotated observations. |
| `/fstack-schema` | Model + migration. Logs annotated observations. |

## How modes emerge

Domain skills log every meaningful choice. Each observation lands in `~/.fstack/observations/sessions.jsonl` with auto-captured context.

When `/fstack` runs, it batches observations into **work contexts**:

- Session key = `(repo, branch_norm)` where branches normalize to `main / spike / feature / fix / release / chore / other`.
- Per-context: weighted mean of each observed dimension, hour distribution, list of recent annotations.

Each context is then projected to two axes:

- **rigor** = avg of `(1 - risk_tolerance)`, `test_rigor`, `architecture_care`. How careful.
- **appetite** = avg of `bias_for_action`, `scope_appetite`. How ambitious.

Both axes get binned `loose / balanced / rigorous` × `small / steady / bold`. Nine cells. Each cell with `>= threshold` observations becomes a subprofile:

| Cell | Alias |
|------|-------|
| loose-small | quick-hack |
| loose-steady | iterate |
| loose-bold | move-fast |
| balanced-small | polish |
| balanced-steady | default-shipping |
| balanced-bold | growth-push |
| rigorous-small | maintenance |
| rigorous-steady | production |
| rigorous-bold | scale-push |

Example output:

```
/fstack-profile list
* default              (manual)    default
  rigorous-steady      (inferred)  production
  loose-bold           (inferred)  move-fast
  loose-small          (inferred)  quick-hack
```

```
/fstack-profile show rigorous-steady
SUBPROFILE: rigorous-steady (inferred) — production
  inferred: rigorous rigor, steady appetite. Seen in: acme:main @ afternoon (4), acme:main @ morning (4)

DEVELOPER
  risk_tolerance:    0.18 (cautious)
  test_rigor:        0.83 (full-coverage)
  architecture_care: 0.88 (principled)

SEEN IN
  acme:main @ afternoon  (4)
  acme:main @ morning  (4)

EXAMPLES
  - FK + ON DELETE CASCADE
  - full integration tests
  - locked schema before merge
```

Versus the same repo on a spike branch:

```
/fstack-profile show loose-bold
SUBPROFILE: loose-bold (inferred) — move-fast
  inferred: loose rigor, bold appetite. Seen in: acme:spike @ afternoon (6)

DEVELOPER
  risk_tolerance:    0.83 (bold)
  bias_for_action:   0.88 (shipper)
  scope_appetite:    0.85 (ambitious)
  test_rigor:        0.20 (happy-path)

EXAMPLES
  - experimental component lib
  - ship + iterate, see what breaks
  - no tests on spike branch
```

## Auto-switch

`/fstack` preamble:

1. If `last_cluster_at` is stale (> 24h) and there are new observations, recluster.
2. Read current context: repo, branch_norm, hour.
3. Tiered match against inferred clusters' `context_distribution`:
   - **Exact**: `<repo>:<branch_norm> @ <band>` — best signal.
   - **Same repo + branch**: matches across hour bands.
   - **Same repo**: matches across any branch/band.
4. Pick the cluster with the highest score. If nothing matches, stay put.
5. If match differs from active and `pinned == null` and `auto_switch == true`, switch.

Switch is announced in one line. No surprises.

## Overrides

- `/fstack-profile pin <key>` (or just `pin` for current) locks the active subprofile.
- `/fstack-profile unpin` clears.
- `fstack-config set profile.auto_switch false` disables entirely.

Manual subprofiles still work for situations clustering can't infer:

```
/fstack-profile create work --label "Work mode" --description "deep focus"
/fstack-profile use work
```

Manual subprofiles carry `origin: "manual"` and are never overwritten by clustering.

## Profile mechanics

Each subprofile tracks:

- `developer`: 5 dimensions (0..1) — risk_tolerance, bias_for_action, scope_appetite, test_rigor, architecture_care.
- `preferences`: 5 string fields — code_style, comment_level, abstraction, error_handling, type_safety.
- `stack`: 10 array fields — language, runtime, frontend, ui, backend_framework, database, orm, auth, deploy, testing.
- `inferred`: rolling means + sample count.
- `examples`: up to 5 representative annotations (inferred only).
- `context_distribution`: `{ "<repo>:<branch_norm> @ <band>": count }` (inferred only).

For manual subprofiles: `developer` is what you declared.
For inferred: `developer` is computed from observations.

## Config

`~/.fstack/config.json`. Versioned. Auto-migrates.

```bash
~/.fstack-bin/fstack-config show | jq
```

Schema (v0.4.0):

```json
{
  "version": "0.4.0",
  "install": { "mode": "dev|prod", "...": "..." },
  "profile": {
    "active": "<key>",
    "pinned": "<key>|null",
    "auto_switch": true,
    "last_cluster_at": "...",
    "subprofiles": {
      "<key>": {
        "label": "...",
        "description": "...",
        "origin": "manual|inferred",
        "developer":   { "...": "..." },
        "preferences": { "...": "..." },
        "stack":       { "...": "..." },
        "inferred":    { "...": "..." },
        "examples":    [],
        "context_distribution": {},
        "cluster_meta": { "method": "behavior-2d-v2", "rigor_bin": "...", "appetite_bin": "...", "...": "..." } | null,
        "declared_at": "...",
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
- v0.2 → v0.3: add `pinned`, `auto_switch`. Mark existing subprofiles `origin: "manual"`. Consolidate observation logs into `sessions.jsonl`.
- v0.3 → v0.4: drop time-based `<repo>-<band>` inferred subprofiles (the cluster axis was wrong); re-cluster on next `/fstack` produces behavior-first ones. Manual subprofiles preserved.

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
