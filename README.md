# fstack

Full-stack skills framework for those who build. Pairs with [gstack](https://github.com/).

fstack profiles how you work and what you build with, then shapes recommendations across scaffolding, API design, and schema work to match.

You're not one developer. **You have modes.** Acme prod-mode is careful, late-night side-project mode is fast and weird. fstack supports both: each subprofile holds its own dimensions, stack, preferences, and observations. Switch when context shifts.

## Install

```bash
./install --dev       # symlink SKILL.md into ~/.claude/skills/ (recommended while iterating)
./install --prod      # copy them (source repo can then be deleted)
./install --status
./install --uninstall
```

Needs `jq` (`brew install jq`).

After install, open a new Claude Code session. Type `/fstack`.

## Skills

| Skill | Does |
|-------|------|
| `/fstack` | Router + setup. 5 questions on first run. Lists subprofiles. |
| `/fstack-profile` | Show, edit, switch, create, clone, rename, remove subprofiles. |
| `/fstack-stack` | Declare tech stack on the active subprofile. |
| `/fstack-skill` | Make your own skills. Stored at `~/.fstack/skills/`. Shared across subprofiles. |
| `/fstack-scaffold` | Scaffold a feature, route, page, or component. Reads active subprofile. |
| `/fstack-api` | Stub an endpoint. Detects style (tRPC, Hono, Next, FastAPI). |
| `/fstack-schema` | Model + migration. Honors active subprofile's ORM + DB. |

## Subprofiles

Each subprofile is independent. Switch when your context shifts:

```
/fstack-profile list
* default     default
  acme-prod   Acme (production) — mature product, careful with prod
  late-night  Late-night prototype — experiments, learning
```

Create one:

```
/fstack-profile create acme-prod --label "Acme (production)" --description "mature product, careful with prod" --clone-from default
```

Switch:

```
/fstack-profile use acme-prod
```

Edits to dimensions, stack, and preferences land on the active subprofile. Observations log to `~/.fstack/observations/<active>.jsonl`. Custom skills are shared.

Common patterns:

- **Per-project.** `acme`, `personal-site`, `client-foo`.
- **Per-mode.** `prod`, `prototype`, `learning`.
- **Per-stage.** `early-stage`, `growth`, `mature`.

Default starts as `default`. If you only ever work in one mode, you only ever need that one.

## Profile mechanics

Each subprofile tracks two parallel views:

1. **Declared.** What you said in setup (or via `/fstack-profile`). Five 0.0..1.0 dimensions per subprofile.
2. **Observed.** What you actually do in that mode. Domain skills log a signal each time they choose. Rolled into a rolling mean.

`/fstack-profile gap` shows the diff for the active subprofile. Drift > 0.2 on any dimension means you update declared (your taste evolved) or address the drift (your behavior isn't matching how you'd describe yourself in this mode).

Different modes can carry different gaps. That's normal: you're more cautious in prod than in learning.

## Config

Everything lives in `~/.fstack/config.json`. Versioned. Migrates on first read.

```bash
~/.fstack-bin/fstack-config show | jq
# or: <repo>/bin/fstack-config show
```

Schema (v0.2.0):

```json
{
  "version": "0.2.0",
  "install": { "mode": "dev|prod", "install_path": "...", "source_path": "...", "installed_skills": ["..."] },
  "profile": {
    "active": "default",
    "subprofiles": {
      "default": {
        "label": "default",
        "description": null,
        "developer":   { "risk_tolerance": 0.5, "...": "..." },
        "preferences": { "code_style": "...", "...": "..." },
        "stack":       { "language": [], "...": [] },
        "inferred":    { "sample_count": 0, "rolling": {} },
        "declared_at": "...",
        "created_at":  "...",
        "last_used_at": "..."
      }
    }
  },
  "custom_skills": [{ "name": "...", "description": "...", "path": "..." }]
}
```

Observations: `~/.fstack/observations/<subprofile-key>.jsonl`. `fstack-observe rollup` writes per-subprofile means into `inferred.rolling`.

Custom skills are top-level: they're tools you make, not mode-dependent.

## v0.1 → v0.2 migration

Auto. First read of an old config wraps `developer`, `preferences`, and the top-level `stack` into a `default` subprofile. Existing `observations.jsonl` moves to `observations/default.jsonl`. No data loss.

## Custom skills

```
/fstack-skill
> Make a skill that runs my pre-deploy checklist: build, smoke test, screenshot, post in Slack.
```

Skill stub goes to `~/.fstack/skills/<name>/SKILL.md`, symlinked into `~/.claude/skills/<name>/SKILL.md`. Restart Claude Code to pick it up. Shared across subprofiles.

## With gstack

Designed to compose.

- **fstack**: what you build (scaffold, api, schema, profile).
- **gstack**: how you operate (browse, qa, ship, review, deploy, retro).

No integration bridge. No shared state. Use both, each respects what's in your repo.

## License

MIT
