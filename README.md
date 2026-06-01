# fstack

Full-stack skills framework for those who build. Pairs with [gstack](https://github.com/).

fstack profiles how you work and what you build with, then shapes recommendations across scaffolding, API design, and schema work to match.

You're not one developer. **You have modes.** Acme prod mode is careful. Late-night side-project mode is fast and weird. fstack notices and adapts:

1. Every observation gets logged with context (repo, hour, branch).
2. Observations periodically cluster by context.
3. Each cluster becomes a subprofile fstack can recognize and switch into.

You don't manage modes. They emerge. You can still pin one or create one manually if you want.

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

Every time a domain skill makes a choice, it logs a one-line annotation plus a dimension signal to `~/.fstack/observations/sessions.jsonl`. Each entry carries context (repo, hour-of-day, weekday, branch).

Once you have enough observations, `/fstack` runs `fstack-cluster cluster` in the preamble. Clusters group observations by `(repo, hour-band)`. Hour bands are `morning`, `afternoon`, `evening`, `late-night`. Each cluster crossing the sample threshold becomes a subprofile with `origin: "inferred"`.

Example after a few weeks of real use:

```
/fstack-profile list
* acme-afternoon              (inferred)  acme-afternoon — inferred from 47 observations
  acme-morning                (inferred)  acme-morning — inferred from 22 observations
  side-project-late-night     (inferred)  side-project-late-night — inferred from 31 observations
  default                     (manual)    default
```

The `acme-afternoon` subprofile, for example, might carry:

```
DEVELOPER
  risk_tolerance:    0.23 (cautious)
  test_rigor:        0.81 (full-coverage)
  architecture_care: 0.87 (principled)

Examples:
  - "POST /teams/invite in tRPC, separate schema + types"
  - "Migration with FK + ON DELETE CASCADE on teams"
  - "Scaffolded teams page with smoke + integration"
```

Versus `side-project-late-night`:

```
DEVELOPER
  risk_tolerance:    0.88 (bold)
  test_rigor:        0.18 (happy-path)
  architecture_care: 0.22 (pragmatic)

Examples:
  - "Quick prototype of voice memo idea"
  - "No tests, just see if it works"
  - "SQLite, one denormalized table"
```

The cluster engine notices the pattern. fstack switches based on context.

## Auto-switch

`/fstack`'s preamble:

1. If `last_cluster_at` is stale (> 24h) and there are new observations, recluster.
2. Read current context: repo (git toplevel basename) + hour.
3. Ask `fstack-cluster match` for the best subprofile.
4. If the match differs from active and `pinned` is null and `auto_switch` is true, switch.

The next session in the same context lands you in the right mode automatically.

Override:

- `/fstack-profile pin <key>` (or just `pin` for current) locks the active subprofile.
- `/fstack-profile unpin` clears the pin.
- `fstack-config set profile.auto_switch false` disables auto-switching entirely.

## Manual subprofiles

You can still create named ones explicitly. Useful for stuff fstack can't infer from `(repo, hour)`:

```
/fstack-profile create work --label "Work mode" --description "deep focus, no shipping"
/fstack-profile use work
/fstack-stack
```

Manual subprofiles are tagged `origin: "manual"`. The cluster engine never overwrites them.

## Profile mechanics

Each subprofile tracks:

- `developer`: 5 dimensions (0..1) — risk_tolerance, bias_for_action, scope_appetite, test_rigor, architecture_care.
- `preferences`: 5 string fields — code_style, comment_level, abstraction, error_handling, type_safety.
- `stack`: 10 array fields — language, runtime, frontend, ui, backend_framework, database, orm, auth, deploy, testing.
- `inferred`: rolling means + sample count, computed from observations.
- `examples`: up to 3 short annotations (for inferred subprofiles).

For manual subprofiles: `developer` is what you declared, `inferred.rolling` is what was observed.

For inferred subprofiles: `developer` is computed from observations, `inferred.rolling` matches.

## Config

`~/.fstack/config.json`. Versioned. Auto-migrates.

```bash
~/.fstack-bin/fstack-config show | jq
```

Schema (v0.3.0):

```json
{
  "version": "0.3.0",
  "install": { "mode": "dev|prod", "..." : "..." },
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
        "inferred":    { "sample_count": N, "rolling": {} },
        "examples":    [],
        "cluster_meta": { "repo": "...", "hour_band": "..." } | null,
        "declared_at": "...",
        "created_at":  "...",
        "last_used_at": "..."
      }
    }
  },
  "custom_skills": []
}
```

Observations: `~/.fstack/observations/sessions.jsonl`. One JSONL stream, all entries.

## Migration

Auto. First read of an old config upgrades:

- v0.1 → v0.2: wrap flat profile + top-level stack into `default` subprofile.
- v0.2 → v0.3: add `pinned`, `auto_switch`, `last_cluster_at`. Mark existing subprofiles `origin: "manual"`. Consolidate per-subprofile observation logs into `sessions.jsonl` with `profile_at_log` tag.

No data loss.

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

No shared state. Use both.

## License

MIT
