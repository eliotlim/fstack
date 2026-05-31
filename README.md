# fstack

Full-stack skills framework for those who build. Pairs with [gstack](https://github.com/).

fstack profiles how you work and what you build with, then shapes recommendations across scaffolding, API design, and schema work to match.

One thing it's opinionated about: **you make it yours**. Declare what you like. fstack notes what you actually do. The two get reconciled when they drift.

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
| `/fstack` | Router + setup. 5 questions on first run. Surfaces what you have. |
| `/fstack-profile` | View, edit, reconcile 5 dimensions + 5 preferences. |
| `/fstack-stack` | Declare tech stack across 10 categories. |
| `/fstack-skill` | Make your own skills. Stored at `~/.fstack/skills/`. |
| `/fstack-scaffold` | Scaffold a feature, route, page, or component. Logs observations. |
| `/fstack-api` | Stub an endpoint. Detects style (tRPC, Hono, Next, FastAPI, etc.). |
| `/fstack-schema` | Model + migration. Honors chosen ORM + DB. |

## Profile

Two parallel views:

1. **Declared.** What you said in setup (or edited via `/fstack-profile`). Five 0.0..1.0 dimensions.
2. **Observed.** What you actually do. Domain skills log a signal each time they choose. Rolled into a rolling mean.

`/fstack-profile gap` shows the diff. Drift > 0.2 on any dimension? You either update declared (your taste evolved) or fix the drift (your behavior isn't matching your description).

Same mechanism as gstack's `/plan-tune`, applied to full-stack recommendations.

## Config

Everything lives in `~/.fstack/config.json`. Versioned.

```bash
~/.fstack-bin/fstack-config show | jq
# or: <repo>/bin/fstack-config show
```

Schema:

```json
{
  "version": "0.1.0",
  "install": { "mode": "dev|prod", "install_path": "...", "source_path": "...", "installed_skills": ["..."] },
  "profile": {
    "developer":   { "risk_tolerance": 0.5, "bias_for_action": 0.5, "...": "..." },
    "preferences": { "code_style": "...", "comment_level": "...", "...": "..." },
    "inferred":    { "sample_count": 0, "rolling": {} }
  },
  "stack": { "language": [], "runtime": [], "...": [] },
  "custom_skills": [{ "name": "...", "description": "...", "path": "..." }]
}
```

Observations are appended to `~/.fstack/observations.jsonl`. `fstack-observe rollup` writes them into `profile.inferred.rolling`.

## Custom skills

Codify a personal pattern as a slash command:

```
/fstack-skill
> Make a skill that runs my pre-deploy checklist: build, smoke test, screenshot, post in Slack.
```

Skill stub goes to `~/.fstack/skills/<name>/SKILL.md`, symlinked into `~/.claude/skills/<name>/SKILL.md`. Restart Claude Code to pick it up.

## With gstack

Designed to compose. Rough split:

- **fstack**: what you build (scaffold, api, schema, profile).
- **gstack**: how you operate (browse, qa, ship, review, deploy, retro).

No integration bridge. No shared state. Use both, each respects what's in your repo.

## License

MIT
